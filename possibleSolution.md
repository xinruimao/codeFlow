

## IPFS go-datastore

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

## 将Blobstore & BlobFS作为leveldb的后端支撑
spdk已经有过将Blobfs & Blobstore作为rocksdb的后端存储支撑的成功案例，而Rocksdb是Facebook公司在Leveldb基础之上开发的一个嵌入式K-V系统，故此方案直观上是可行的，也具有参考。

## Blobstore & BlobFS

Blobstore是位于SPDK bdev之上的Blob管理层，用于与用户态文件系统Blobstore Filesystem （BlobFS）集成，从而代替传统的文件系统，支持更上层的服务，如数据库MySQL、K-V存储引擎Rocksdb以及分布式存储系统Ceph、Cassandra等。

以Rocksdb为例，通过BlobFS作为Rocksdb的存储后端的优势在于，I/O经由BlobFS与Blobstore下发到bdev，随后由SPDK用户态driver写入磁盘。整个I/O流从发起到落盘均在用户态操作，完全bypass内核。此外，可以充分利用SPDK所提供的异步、无锁化、Zero Copy、轮询等机制，大幅度减少额外的系统开销。它们之间的关系如下所示(以NVMe bdev为例):


<div align=center><img width="600" height="300" src="https://github.com/xinruimao/codeFlow/blob/main/images/image.png"/></div>


## 直接将Blobfs作为ipfs的datastore的一种实现
从ipfs的datastore中可以看到，ipfs中采用了两种不同的 datastore 实现来存储数据，分别是 `flatfs.datastore` 和 `leveldb.datastore`，可以增加`Blobfs.datastore`。


<div align=center><img width="450" height="300" src="https://github.com/xinruimao/codeFlow/blob/main/images/Untitled%20Diagram.png"/></div>















