# ipfs add codeFlow

the design of storage
```
     +---------+
     |   MFS   |   File system.
     +---------+
     |  UnixFS |   Files and directories.
     +---------+
     |   DAG   |   Nodes.
     +---------+
     |  Block  |   Stream of bits.
     +---------+
```
<img src="https://docs.google.com/drawings/d/e/2PACX-1vS_n1FvSu6mdmSirkBrIIEib2gqhgtatD9awaP2_WdrGN4zTNeg620XQd9P95WT-IvognSxIIdCM5uE/pub?w=1000&amp;h=760">

# Code Flow
## Concepts
- [Files](https://github.com/ipfs/docs/issues/133)

--- 
**Try this yourself**
> 
> ```
> # Convert a file to the IPFS format.
> echo "Hello World" > new-file
> ipfs add new-file
> added QmWATWQ7fVPP2EFGu71UkfnqhYXDYH566qy47CnJDgvs8u new-file
> 12 B / 12 B [=========================================================] 100.00%
>
> # Add a file to the MFS.
> NEW_FILE_HASH=$(ipfs add new-file -Q)
> ipfs files cp /ipfs/$NEW_FILE_HASH /new-file
> 
> # Get information from the file in MFS.
> ipfs files stat /new-file
> # QmWATWQ7fVPP2EFGu71UkfnqhYXDYH566qy47CnJDgvs8u
> # Size: 12
> # CumulativeSize: 20
> # ChildBlocks: 0
> # Type: file
> 
> # Retrieve the contents.
> ipfs files read /new-file
> # Hello World
> ```

## Code Flow

**[`UnixfsAPI.Add()`](https://github.com/ipfs/go-ipfs/blob/v0.4.18/core/coreapi/unixfs.go#L31)** - *Entrypoint into the `Unixfs` package*

The `UnixfsAPI.Add()` acts on the input data or files, to build a _merkledag_ node (in essence it is the entire tree represented by the root node) and adds it to the _blockstore_.
Within the function, a new `Adder` is created with the configured `Blockstore` and __DAG service__`. 

- **[`adder.AddAllAndPin(files)`](https://github.com/ipfs/go-ipfs/blob/v0.4.18/core/coreunix/add.go#L403)** - *Entrypoint to the `Add` logic*
    encapsulates a lot of the underlying functionality that will be investigated in the following sections. 

    Our focus will be on the simplest case, a single file, handled by `Adder.addFile(file files.File)`. 

  - **`adder.addFileNode(path string, file files.Node, toplevel bool)`** - *Create the _DAG_ and add to `MFS`*
      -   `adder.maybePauseForGC()`
      -   `adder.addFileNode`
      -   `adder.maybePauseForGC()`
      -   `adder.PinRoot(rn)`
      -   `adder.dagService.Add(adder.ctx, root)`

### go-merkledag

在代码 `https://github.com/ipfs/go-merkledag` 中：

`merkledag.go` 实现了此接口


	var _ ipld.LinkGetter = &dagService{}
	var _ ipld.NodeGetter = &dagService{}
	var _ ipld.NodeGetter = &sesGetter{}
	var _ ipld.DAGService = &dagService{}


看下其中的方法实现:


	// Add adds a node to the dagService, storing the block in the BlockService
	func (n *dagService) Add(ctx context.Context, nd ipld.Node) error {
	    ....
		return n.Blocks.AddBlock(nd)
	}
	
	// Get retrieves a node from the dagService, fetching the block in the BlockService
	func (n *dagService) Get(ctx context.Context, c *cid.Cid) (ipld.Node, error) {
	    ...
		b, err := n.Blocks.GetBlock(ctx, c)
	    ...
		return ipld.Decode(b)
	}


我们可以看到`DagService`的方法最终调用的是`BlockService`中的方法

## IPFS BlockService

代码位置 `https://github.com/ipfs/go-blockservice`
BlockService 添加block 最终调用 `blockstore` 来添加 block，并且放入到 bitswap 中，获取 block 时从 blockstore 中获取，如果没找到则从 bitswap 中获取。

	
	// BlockService is a hybrid block datastore. It stores data in a local
	// datastore and may retrieve data from a remote Exchange.
	// It uses an internal `datastore.Datastore` instance to store values.
	type BlockService interface {
		io.Closer
		BlockGetter
		Blockstore() blockstore.Blockstore
	
		// Exchange returns a reference to the underlying exchange (usually )
		Exchange() exchange.Interface
	
		// AddBlock puts a given block to the underlying datastore
		AddBlock(o blocks.Block) error
	
		// AddBlocks adds a slice of blocks at the same time using batching
		// capabilities of the underlying datastore whenever possible.
		AddBlocks(bs []blocks.Block) error
	
		// DeleteBlock deletes the given block from the blockservice.
		DeleteBlock(o *cid.Cid) error
	}
	
	
	// 将 block 添加进 blockstore
	func (s *blockService) AddBlock(o blocks.Block) error {
		c := o.Cid()
		// hash 校验
		err := verifcid.ValidateCid(c)
	    
		if s.checkFirst {
			if has, err := s.blockstore.Has(c); has || err != nil {
				return err
			}
		}
	    //添加进blockstore
		if err := s.blockstore.Put(o); err != nil {
			return err
		}
	
		if err := s.exchange.HasBlock(o); err != nil {
			// TODO(#4623): really an error?
			return errors.New("blockservice is closed")
		}
	
		return nil
	}
	
	
	func getBlock(ctx context.Context, c *cid.Cid, bs blockstore.Blockstore, f exchange.Fetcher) (blocks.Block, error) {
	     // hash 检查
		err := verifcid.ValidateCid(c)
	
		block, err := bs.Get(c)
	
		if err == blockstore.ErrNotFound && f != nil {
			log.Debug("Blockservice: Searching bitswap")
			blk, err := f.GetBlock(ctx, c)
			if err != nil {
				if err == blockstore.ErrNotFound {
					return nil, ErrNotFound
				}
				return nil, err
			}
			return blk, nil
		}
	
		if err == blockstore.ErrNotFound {
			return nil, ErrNotFound
		}
	
		return nil, err
	}
	

### Blockstore

// Blockstore wraps a Datastore block-centered methods and provides a layer of abstraction which allows to add different caching strategies.
     
     func (bs *blockstore) Put(block blocks.Block) error {
          k := dshelp.CidToDsKey(block.Cid())

          // Has is cheaper than Put, so see if we already have it
          exists, err := bs.datastore.Has(k)
          if err == nil && exists {
               return nil // already stored.
          }
          return bs.datastore.Put(k, block.RawData())
     }

### IPFS go-datastore

代码位置 `https://github.com/ipfs/go-datastore`

datastore 是数据存储和数据库访问的通用抽象层。 它设计了一套简单的API，它允许无需更改代码即可无缝交换数据。 因此可以实现不同的 datastore 来处理不同强度要求的数据。

datastore.go 接口中定义了对block的增删查改方法

	type Datastore interface {
		Put(key Key, value []byte) error
		Get(key Key) (value []byte, err error)
		Has(key Key) (exists bool, err error)
		Delete(key Key) error
		Query(q query.Query) (query.Results, error)
	}


在代码 `go-ipfs-config\datastore.go` 中我们可以看到 ipfs 中采用了两种不同的 datastore 实现来存储数据，分别是 `flatfs.datastore` 和 `leveldb.datastore`


	return Datastore{
		StorageMax:         "10GB",
		StorageGCWatermark: 90, // 90%
		GCPeriod:           "1h",
		BloomFilterSize:    0,
		Spec: map[string]interface{}{
			"type": "mount",
			"mounts": []interface{}{
				map[string]interface{}{
					"mountpoint": "/blocks",
					"type":       "measure",
					"prefix":     "flatfs.datastore",
					"child": map[string]interface{}{
						"type":      "flatfs",
						"path":      "blocks",
						"sync":      true,
						"shardFunc": "/repo/flatfs/shard/v1/next-to-last/2",
					},
				},
				map[string]interface{}{
					"mountpoint": "/",
					"type":       "measure",
					"prefix":     "leveldb.datastore",
					"child": map[string]interface{}{
						"type":        "levelds",
						"path":        "datastore",
						"compression": "none",
					},
				},
			},
		},
	}



#### flatfs datastore

代码位置 `https://github.com/ipfs/go-ds-flatfs`

调用操作系统 api 实现 datastore 存储

	
	func (fs *Datastore) Put(key datastore.Key, value []byte) error {
	    ...
		for i := 1; i <= putMaxRetries; i++ {
			err = fs.doWriteOp(&op{
				typ: opPut,
				key: key,
				v:   value,
			})
		}
	    ...
		return err
	}
	
	func (fs *Datastore) doWriteOp(oper *op) error {
	    ...
		err := fs.doOp(oper)
	    ...
		return err
	}
	
	func (fs *Datastore) doOp(oper *op) error {
		switch oper.typ {
		case opPut:
			return fs.doPut(oper.key, oper.v)
		case opDelete:
			return fs.doDelete(oper.key)
		case opRename:
			return fs.renameAndUpdateDiskUsage(oper.tmp, oper.path)
		default:
			panic("bad operation, this is a bug")
		}
	}
	
	//最终调用存储文件的方法
	func (fs *Datastore) doPut(key datastore.Key, val []byte) error {
	
		dir, path := fs.encode(key)
		if err := fs.makeDir(dir); err != nil {
			return err
		}
	
		tmp, err := ioutil.TempFile(dir, "put-")
		if err != nil {
			return err
		}
		closed := false
		removed := false
		defer func() {
			if !closed {
				// silence errcheck
				_ = tmp.Close()
			}
			if !removed {
				// silence errcheck
				_ = os.Remove(tmp.Name())
			}
		}()
	
		if _, err := tmp.Write(val); err != nil {
			return err
		}
		if fs.sync {
			if err := syncFile(tmp); err != nil {
				return err
			}
		}
		if err := tmp.Close(); err != nil {
			return err
		}
		closed = true
	
		err = fs.renameAndUpdateDiskUsage(tmp.Name(), path)
		if err != nil {
			return err
		}
		removed = true
	
		if fs.sync {
			if err := syncDir(dir); err != nil {
				return err
			}
		}
		return nil
	}
	
	func (fs *Datastore) encode(key datastore.Key) (dir, file string) {
		noslash := key.String()[1:]
		dir = filepath.Join(fs.path, fs.getDir(noslash))
		file = filepath.Join(dir, noslash+extension)
		return dir, file
	}
	
	func (fs *Datastore) Get(key datastore.Key) (value []byte, err error) {
		_, path := fs.encode(key)
		data, err := ioutil.ReadFile(path)
		if err != nil {
			if os.IsNotExist(err) {
				return nil, datastore.ErrNotFound
			}
			return nil, err
		}
		return data, nil
	}


#### leveldb datastore

代码位置 `https://github.com/ipfs/go-ds-leveldb`

采用 leveldb 存储
	
	type datastore struct {
		DB   *leveldb.DB
		path string
	}
	
	//初始化 leveldb 实例
	func NewDatastore(path string, opts *Options) (*datastore, error) {
	    ...
		var db *leveldb.DB
		db, err = leveldb.Open(storage.NewMemStorage(), &nopts)
	
		return &datastore{
			DB:   db,
			path: path,
		}, nil
	}
	
	func (d *datastore) Put(key ds.Key, value []byte) (err error) {
		return d.DB.Put(key.Bytes(), value, nil)
	}
	
	func (d *datastore) Get(key ds.Key) (value []byte, err error) {
		val, err := d.DB.Get(key.Bytes(), nil)
		if err != nil {
			if err == leveldb.ErrNotFound {
				return nil, ds.ErrNotFound
			}
			return nil, err
		}
		return val, nil
	}

### summary
![](https://github.com/xinruimao/codeFlow/blob/main/images/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)


## IPFS repo 定义

IPFS 节点的数据对象都存储在本地的 `repo` 中（类似于git）。根据所使用的存储介质不同，有不同的`repo`实现。ipfs 节点使用[fs-repo](https://github.com/ipfs/specs/blob/master/repo/fs-repo)。

常见的`repo`实现有：

+ [fs-repo](https://github.com/ipfs/specs/blob/master/repo/fs-repo)  - 存储于操作系统的文件系统
+ [mem-repo](https://github.com/ipfs/specs/blob/master/repo/mem-repo) - 存储于内存
+ [s3-repo](https://github.com/ipfs/specs/blob/master/repo/mem-repo)  - 存储于 amazon s3


## IPFS repo 组成


Repo 存储了一组 IPLD 对象，分别表示：

+ keys      - 加密密钥，包括节点的标识
+ config    - 节点配置
+ datastore - 本地存储的数据和索引数据
+ logs      - 调试的事件日志
+ hooks     - 脚本在预定义的时间运行（尚未实现）

--
	
	tree ~/.ipfs
		
	.ipfs/
	├── api             <--- running daemon api addr
	├── blocks/         <--- objects stored directly on disk
	│   └── aa          <--- prefix namespacing like git
	│       └── aa      <--- N tiers
	├── config          <--- config file (json or toml)
	├── hooks/          <--- hook scripts
	├── keys/           <--- cryptographic keys
	│   ├── id.pri      <--- identity private key
	│   └── id.pub      <--- identity public key
	├── datastore/      <--- datastore
	├── logs/           <--- 1 or more files (log rotate)
	│   └── events.log  <--- can be tailed
	├── repo.lock       <--- mutex for repo
	└── version         <--- version file

### blocks


blocks 包含表示本地存储的所有IPFS对象的原始数据，无论是固定（pinned）还是缓存（cached）的数据。 blocks 由 datastore 控制。 例如，它可以将它存储在 leveldb 中，也可以存储在 git 中。

在默认情况下，所有 block 存储于 fs-datastore 中。



### keys

keys 目录包含节点有权访问的所有密钥。 密钥以其 hash 命名，扩展名描述它们的密钥类型。格式为 id.{pub，sec}

	
	<key>.pub is a public key
	<key>.pri is a private key
	<key>.sym is a symmetric secret key


### datastore

datastore 目录包含用于存储在 leveldb 中用来操作 IPFS 节点的数据。 如果用户改为使用 boltdb datastore，则该目录将命名为boltdb。 因此，每个数据库的数据文件不会发生冲突。
这个目录将来可以能考虑改为 leveldb 命名。

<details>
  <summary>show code</summary>
  <pre><code> 
     System.out.println("Hello");
  </code></pre>
</details>















































