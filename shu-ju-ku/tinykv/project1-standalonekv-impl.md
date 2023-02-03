# Project1-StandaloneKV-Impl

## 框架

该项目需要基于 **badgerDB** 实现一个单机的、支持 Column Family 的 KV存储 [gRPC](https://grpc.io/docs/guides/) 服务。[Column family](https://en.wikipedia.org/wiki/Standard\_column\_family)（下文将缩写为 CF）是一个类似于键命名空间的术语，即同一键在不同列族中的值是不一样的。

```go
const (
   CfDefault string = "default"
   CfWrite   string = "write"
   CfLock    string = "lock"
)

func KeyWithCF(cf string, key []byte) []byte {
   return append([]byte(cf+"_"), key...)
}
```

例如默认的 CF 为 "default"，那么该 CF 下的名为 "apple" 的 Key，实际存储为 "default\_apple" 这个字符串。可以简单地将多个列族视为独立的小型数据库。它是用来支持 project 4 中的事务模型的，现在还用不到，感兴趣的可以来看这篇文章：[Percolator](https://tidb.net/blog/b6d840f2?utm\_source=tidb-community\&utm\_medium=referral\&utm\_campaign=repost)

<details>

<summary>badgerDB</summary>

### badgerDB 有什么厉害的地方，为什么要用它，而不用其他的数据库。

这里有两篇文章关于 bardgerDB 的讲解。

* [badger 简介](http://note.iawen.com/note/graph/badger\_base)
* [badger 一个高性能的LSM K/V store](https://colobu.com/2017/10/11/badger-a-performant-k-v-store/)

简单来说，Badger 是为 Dgraph 而生的, 是 Dgraph 底层数据存储引擎。

Dgraph 是开源的，可扩展的，分布式的，低延迟图形数据库，采用 Go 语言书写。Dgraph 开始的时候，的确采用的是 RocksDB，但RocksDB 是 C++语言写的，使用时需要通过 Go 调用 Cgo。而 Cgo中又有一些缺点：

* Go profiler 无法分析和监测 Cgo 代码段里的问题, 所有工具链都不起作用
* 当涉及到 Cgo 时, 轻量级的 goroutine 会变成昂贵的 pthreadCgo
* 造成了内存泄漏

于是着手进行了 BadgerDB 这个底层存储引擎的开发，并进行对 SSD 的优化。并于 2016 年发表了论文：[WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

</details>

该项目的架构图如下：

<figure><img src="../../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>架构图</p></figcaption></figure>

## StandAloneStorage

1. Server 结构体包含了 storage 接口，需要为 StandAloneStorage 实现 Storage 的所有接口。

```go
type Storage interface {
   Start() error
   Stop() error
   Write(ctx *kvrpcpb.Context, batch []Modify) error
   Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

2. 上来定义结构体的时候，可能会参考 RaftStorage 定义成下面这样，这是因为 engine\_util.Engines 已经完成了对 badger 的封装。

```go
type StandAloneStorage struct {
	engines *engine_util.Engines
}
```

3. 但其实我们只需要 **badger.DB** 就已经够了，如果使用 engines 会创建两个 DB，测试时也需要再关闭。会浪费一些时间，我在修改之后从12s 变为了 8s。

```go
type StandAloneStorage struct {
   db *badger.DB
}
```

### **Reader**

1. 接下来就可以完成 StandAloneStorage 的书写。构造函数，缺少参数补充即可。Start函数，没什么可做的。Stop 函数关闭数据库。Reader 函数，需要我们返回一个实现了 StorageReader 接口的 Reader，这里就叫 StandAloneStorageReader。Write 函数需要对传过来的数据进行处理。
2. StorageReader 接口如下：

```go
type StorageReader interface {
   // When the key doesn't exist, return nil for the value
   GetCF(cf string, key []byte) ([]byte, error)
   IterCF(cf string) engine_util.DBIterator
   Close()
}
```

3. 根据提示，StandAloneStorageReader 结构体需要包含一个 **badger.Txn**，并且实现上面的三个方法，直接模仿 region\_reader 实现即可。

```go
type StandAloneStorageReader struct {
   txn *badger.Txn
}
```

4. 获取badger.Txn的函数engine\_util并未给出，需要直接调用badger.DB.NewTransaction函数。

```go
func (db *DB) NewTransaction(update bool) *Txn
```

update 为真表示 Put/Delete 两个写操作，为假表示 Get/Scan 两个读操作。这里直接传入 false 即可。

### **Write**

```go
type Modify struct {
   Data interface{}
}

type Put struct {
   Key   []byte
   Value []byte
   Cf    string
}

type Delete struct {
   Key []byte
   Cf  string
}
```

Modify 表示一个 Put 还是 Delete 操作，直接通过断言确定是 Put 还是 Delete，进而调用 engine\_util 提供的 PutCF 和 DeleteCF 两个函数。

```go
for _, v := range batch {
   switch v.Data.(type) {
   case storage.Put:
      // PutCF
   case storage.Delete:
      // DeleteCF
   }
}
```

但其实，engine\_util 下面的这两个函数 PutCF 和 DeleteCF 在底层也是调用了事务。

## Server

1. 这一部分需要完成四个函数 RawGet、RawPut、RawDelete、RawScan。同时利用上面完成的 Reader 结构体 和 Write 函数。
2. Get 和 Scan 需要使用 Reader。Put 和 Delete 需要使用 Write。

```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error)
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error)
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error)
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error)
```

3. RawScan 比较难以理解，从指定的 key 出发向后寻找 limit 个 KVPair 并将其返回。步骤如下：

{% tabs %}
{% tab title="text" %}
1. 获取到 reader，记得defer
2. 通过reader 的 IterCF 函数，拿到一个 key 的迭代器。记得 defer
3. 通过迭代器的 Seek 函数进行定位。
4. 开始循环，等于 limit 时停止。每次循环开始时都要通过 Valid 函数校验迭代器是否有效。通过 Item 函数获得 DBItem。进而获得 key 和 value。每次循环结束时迭代器要调用 Next 函数进行移动。
{% endtab %}

{% tab title="code" %}
```go
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
	// Hint: Consider using reader.IterCF
	limit := req.GetLimit()
	startKey := req.GetStartKey()
	storageReader, _ := server.storage.Reader(req.Context)
	defer storageReader.Close()
	iterator := storageReader.IterCF(req.GetCf())
	defer iterator.Close()
	iterator.Seek(startKey)
	response := &kvrpcpb.RawScanResponse{
		Error: "",
	}
	var i uint32 = 0
	for ; i < limit; i++ {
		if !iterator.Valid() {
			break
		}
		item := iterator.Item()
		key := item.Key()
		value, err := item.Value()
		if err != nil {
			response.Error = ""
			return response, err
		}
		response.Kvs = append(response.Kvs, &kvrpcpb.KvPair{Key: key, Value: value})
		iterator.Next()
	}
	return response, nil
}
```
{% endtab %}
{% endtabs %}

## 通过

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>PASS</p></figcaption></figure>
