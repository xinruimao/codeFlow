
## Datastore

https://github.com/ipfs/go-datastore/blob/master/datastore.go

```
/*
Datastore represents storage for any key-value pair.

Datastores are general enough to be backed by all kinds of different storage:
in-memory caches, databases, a remote datastore, flat files on disk, etc.

The general idea is to wrap a more complicated storage facility in a simple,
uniform interface, keeping the freedom of using the right tools for the job.
In particular, a Datastore can aggregate other datastores in interesting ways,
like sharded (to distribute load) or tiered access (caches before databases).

While Datastores should be written general enough to accept all sorts of
values, some implementations will undoubtedly have to be specific (e.g. SQL
databases where fields should be decomposed into columns), particularly to
support queries efficiently. Moreover, certain datastores may enforce certain
types of values (e.g. requiring an io.Reader, a specific struct, etc) or
serialization formats (JSON, Protobufs, etc).

IMPORTANT: No Datastore should ever Panic! This is a cross-module interface,
and thus it should behave predictably and handle exceptional conditions with
proper error reporting. Thus, all Datastore calls may return errors, which
should be checked by callers.
*/
type Datastore interface {
	Read
	Write
	// Sync guarantees that any Put or Delete calls under prefix that returned
	// before Sync(prefix) was called will be observed after Sync(prefix)
	// returns, even if the program crashes. If Put/Delete operations already
	// satisfy these requirements then Sync may be a no-op.
	//
	// If the prefix fails to Sync this method returns an error.
	Sync(prefix Key) error
	io.Closer
}

// Write is the write-side of the Datastore interface.
type Write interface {
	// Put stores the object `value` named by `key`.
	//
	// The generalized Datastore interface does not impose a value type,
	// allowing various datastore middleware implementations (which do not
	// handle the values directly) to be composed together.
	//
	// Ultimately, the lowest-level datastore will need to do some value checking
	// or risk getting incorrect values. It may also be useful to expose a more
	// type-safe interface to your application, and do the checking up-front.
	Put(key Key, value []byte) error

	// Delete removes the value for given `key`. If the key is not in the
	// datastore, this method returns no error.
	Delete(key Key) error
}

// Read is the read-side of the Datastore interface.
type Read interface {
	// Get retrieves the object `value` named by `key`.
	// Get will return ErrNotFound if the key is not mapped to a value.
	Get(key Key) (value []byte, err error)

	// Has returns whether the `key` is mapped to a `value`.
	// In some contexts, it may be much cheaper only to check for existence of
	// a value, rather than retrieving the value itself. (e.g. HTTP HEAD).
	// The default implementation is found in `GetBackedHas`.
	Has(key Key) (exists bool, err error)

	// GetSize returns the size of the `value` named by `key`.
	// In some contexts, it may be much cheaper to only get the size of the
	// value rather than retrieving the value itself.
	GetSize(key Key) (size int, err error)

	// Query searches the datastore and returns a query result. This function
	// may return before the query actually runs. To wait for the query:
	//
	//   result, _ := ds.Query(q)
	//
	//   // use the channel interface; result may come in at different times
	//   for entry := range result.Next() { ... }
	//
	//   // or wait for the query to be completely done
	//   entries, _ := result.Rest()
	//   for entry := range entries { ... }
	//
	Query(q query.Query) (query.Results, error)
}
```

### flatfs
https://github.com/ipfs/go-ds-flatfs

```
// Put stores a key/value in the datastore.
//
// Note, that we do not guarantee order of write operations (Put or Delete)
// to the same key in this datastore.
//
// For example. i.e. in the case of two concurrent Put, we only guarantee
// that one of them will come through, but cannot assure which one even if
// one arrived slightly later than the other. In the case of a
// concurrent Put and a Delete operation, we cannot guarantee which one
// will win.
func (fs *Datastore) Put(key datastore.Key, value []byte) error {
	if !keyIsValid(key) {
		return fmt.Errorf("when putting '%q': %v", key, ErrInvalidKey)
	}

	fs.shutdownLock.RLock()
	defer fs.shutdownLock.RUnlock()
	if fs.shutdown {
		return ErrClosed
	}

	_, err := fs.doWriteOp(&op{
		typ: opPut,
		key: key,
		v:   value,
	})
	return err
}

// Delete removes a key/value from the Datastore. Please read
// the Put() explanation about the handling of concurrent write
// operations to the same key.
func (fs *Datastore) Delete(key datastore.Key) error {
	// Can't exist in datastore.
	if !keyIsValid(key) {
		return nil
	}

	fs.shutdownLock.RLock()
	defer fs.shutdownLock.RUnlock()
	if fs.shutdown {
		return ErrClosed
	}

	_, err := fs.doWriteOp(&op{
		typ: opDelete,
		key: key,
		v:   nil,
	})
	return err
}

func (fs *Datastore) Get(key datastore.Key) (value []byte, err error) {
	// Can't exist in datastore.
	if !keyIsValid(key) {
		return nil, datastore.ErrNotFound
	}

	_, path := fs.encode(key)
	data, err := readFile(path)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, datastore.ErrNotFound
		}
		// no specific error to return, so just pass it through
		return nil, err
	}
	return data, nil
}

func (fs *Datastore) Has(key datastore.Key) (exists bool, err error) {
	// Can't exist in datastore.
	if !keyIsValid(key) {
		return false, nil
	}

	_, path := fs.encode(key)
	switch _, err := os.Stat(path); {
	case err == nil:
		return true, nil
	case os.IsNotExist(err):
		return false, nil
	default:
		return false, err
	}
}

func (fs *Datastore) GetSize(key datastore.Key) (size int, err error) {
	// Can't exist in datastore.
	if !keyIsValid(key) {
		return -1, datastore.ErrNotFound
	}

	_, path := fs.encode(key)
	switch s, err := os.Stat(path); {
	case err == nil:
		return int(s.Size()), nil
	case os.IsNotExist(err):
		return -1, datastore.ErrNotFound
	default:
		return -1, err
	}
}

func (fs *Datastore) Query(q query.Query) (query.Results, error) {
	prefix := datastore.NewKey(q.Prefix).String()
	if prefix != "/" {
		// This datastore can't include keys with multiple components.
		// Therefore, it's always correct to return an empty result when
		// the user requests a filter by prefix.
		log.Warnw(
			"flatfs was queried with a key prefix but flatfs only supports keys at the root",
			"prefix", q.Prefix,
			"query", q,
		)
		return query.ResultsWithEntries(q, nil), nil
	}

	// Replicates the logic in ResultsWithChan but actually respects calls
	// to `Close`.
	b := query.NewResultBuilder(q)
	b.Process.Go(func(p goprocess.Process) {
		err := fs.walkTopLevel(fs.path, b)
		if err == nil {
			return
		}
		select {
		case b.Output <- query.Result{Error: errors.New("walk failed: " + err.Error())}:
		case <-p.Closing():
		}
	})
	go b.Process.CloseAfterChildren() //nolint

	// We don't apply _any_ of the query logic ourselves so we'll leave it
	// all up to the naive query engine.
	return query.NaiveQueryApply(q, b.Results()), nil
}





Put/Delete
os.Remove
os.Close
ioutil.TempFile
os.write
os.Sync
os.Stat
os.IsNotExist
os.Rename
os.OpenFile

Get
ioutil.ReadFile

Has/GetSize
os.Mkdir
os.IsExist

Query
os.Open
os.RemoveAll

```


### Blobfs


```
struct spdk_file {
	struct spdk_filesystem	*fs;
	struct spdk_blob	*blob;
	char			*name;
	uint64_t		length;
	bool                    is_deleted;
	bool			open_for_writing;
	uint64_t		length_flushed;
	uint64_t		length_xattr;
	uint64_t		append_pos;
	uint64_t		seq_byte_count;
	uint64_t		next_seq_offset;
	uint32_t		priority;
	TAILQ_ENTRY(spdk_file)	tailq;
	spdk_blob_id		blobid;
	uint32_t		ref_count;
	pthread_spinlock_t	lock;
	struct cache_buffer	*last;
	struct cache_tree	*tree;
	TAILQ_HEAD(open_requests_head, spdk_fs_request) open_requests;
	TAILQ_HEAD(sync_requests_head, spdk_fs_request) sync_requests;
	TAILQ_ENTRY(spdk_file)	cache_tailq;
};


struct spdk_filesystem {
	struct spdk_blob_store	*bs;
	TAILQ_HEAD(, spdk_file)	files;
	struct spdk_bs_opts	bs_opts;
	struct spdk_bs_dev	*bdev;
	fs_send_request_fn	send_request;

	struct {
		uint32_t		max_ops;
		struct spdk_io_channel	*sync_io_channel;
		struct spdk_fs_channel	*sync_fs_channel;
	} sync_target;

	struct {
		uint32_t		max_ops;
		struct spdk_io_channel	*md_io_channel;
		struct spdk_fs_channel	*md_fs_channel;
	} md_target;

	struct {
		uint32_t		max_ops;
	} io_target;
};


spdk_fs_open_file(struct spdk_filesystem *fs, struct spdk_fs_thread_ctx *ctx,const char *name, uint32_t flags, struct spdk_file **file)

spdk_fs_delete_file(struct spdk_filesystem *fs, struct spdk_fs_thread_ctx *ctx,const char *name)

spdk_fs_file_stat(struct spdk_filesystem *fs, struct spdk_fs_thread_ctx *ctx,const char *name, struct spdk_file_stat *stat)

spdk_fs_create_file(struct spdk_filesystem *fs, struct spdk_fs_thread_ctx *ctx, const char *name)


spdk_file_read(struct spdk_file *file, struct spdk_fs_thread_ctx *ctx，void *payload, uint64_t offset, uint64_t length)

spdk_file_write(struct spdk_file *file, struct spdk_fs_thread_ctx *ctx,void *payload, uint64_t offset, uint64_t length)

spdk_file_close(struct spdk_file *file, struct spdk_fs_thread_ctx *ctx)

spdk_fs_rename_file(struct spdk_filesystem *fs, struct spdk_fs_thread_ctx *ctx,const char *old_name, const char *new_name)

```

summary



 
 flatfs   |  blobfs  
 -------- | ---------
os.Open    	| spdk_fs_open_file
os.Remove	| spdk_fs_delete_file
os.Close	| spdk_file_close
ioutil.TempFile
os.write	| spdk_file_write
os.Stat		| spdk_fs_file_stat
os.IsNotExist
os.Rename	| spdk_fs_rename_file
os.OpenFile
ioutil.ReadFile	| spdk_file_read
os.Mkdir
os.IsExist
os.RemoveAll






