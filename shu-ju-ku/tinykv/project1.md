# Project1

### 项目要求

原文地址：



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

该项目有两大部分：

1. 实现一个单机的存储引擎。
2. 实现原始键/值服务处理程序。

来自官方的一些提示还是比较重要，把他放到这里：

> * 你应该使用 [badger.Txn](https://godoc.org/github.com/dgraph-io/badger#Txn) 来实现 `Reader` 函数，因为 badger 提供的事务处理程序可以提供键和值的一致快照。
> * Badger 没有给出对列族的支持。 engine\_util 包（`kv/util/engine_util`）通过给键添加前缀来模拟列族。例如，一个属于特定列族 cf 的 `key` 被存储为 `${cf}_${key}`。它封装了 `badger` 以提供对 CF 的操作，还提供了许多有用的辅助函数。所以你应该通过 `engine_util` 提供的方法进行所有读写操作。请阅读 `util/engine_util/doc.go` 以了解更多。
> * TinyKV fork 了 `badger` 原始版本，并进行了一些修正，所以需要使用 `github.com/Connor1996/badger` 而不是 `github.com/dgraph-io/badger`。
> * 别忘了为 badger.Txn 调用 `Discard()`，并在丢弃前关闭所有迭代器。

### 如何实现

该项目的架构图如下：

![1673185888981](https://cdn.staticaly.com/gh/yi-ge-dian/blog-image-hosting@master/TinyKV/1673185888981.4beu8728fta0.webp)

#### StandAloneStorage

Server 结构体包含了 storage 接口，需要为 StandAloneStorage 实现 Storage 的所有接口。

```go
type Storage interface {
   Start() error
   Stop() error
   Write(ctx *kvrpcpb.Context, batch []Modify) error
   Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

上来定义结构体的时候，可能会参考 RaftStorage 定义成这样，因为 engine\_util.Engines 已经完成了对 badger 的封装。

```go
type StandAloneStorage struct {
	engines *engine_util.Engines
}
```

但其实我们只需要 **badger.DB** 就已经够了，如果使用 engines 会创建两个 DB，测试时也需要再关闭。会浪费一些时间，我在修改之后从12s 变为了 8s。

```go
type StandAloneStorage struct {
   db *badger.DB
}
```

> 题外话，badgerDB 有什么厉害的地方，为什么要用它，而不用其他的数据库。

这里有两篇文章关于 bardgerDB 的讲解。

* badger 简介：<mark style="color:blue;">http://note.iawen.com/note/graph/badger\_base</mark>
* badger 一个高性能的LSM K/V store：https://colobu.com/2017/10/11/badger-a-performant-k-v-store/

简单来说，Badger 是为 Dgraph 而生的, 是 Dgraph 底层数据存储引擎。

Dgraph 是开源的，可扩展的，分布式的，低延迟图形数据库，采用 Go 语言书写。Dgraph 开始的时候，的确采用的是 RocksDB，但RocksDB 是 C++语言写的，使用时需要通过 Go 调用 Cgo。而 Cgo中又有一些缺点：

* Go profiler 无法分析和监测 Cgo 代码段里的问题, 所有工具链都不起作用
* 当涉及到 Cgo 时, 轻量级的 goroutine 会变成昂贵的 pthreadCgo
* 造成了内存泄漏

于是着手进行了 BadgerDB 这个底层存储引擎的开发，并进行对 SSD 的优化。并于 2016 年发表了论文：[WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

**Reader**

接下来就可以完成 StandAloneStorage 的书写。构造函数，缺少参数补充即可。Start函数，没什么可做的。Stop 函数关闭数据库。Reader 函数，需要我们返回一个实现了 StorageReader 接口的 Reader，这里就叫 StandAloneStorageReader。Write 函数需要对传过来的数据进行处理。

StorageReader 接口如下：

```go
type StorageReader interface {
   // When the key doesn't exist, return nil for the value
   GetCF(cf string, key []byte) ([]byte, error)
   IterCF(cf string) engine_util.DBIterator
   Close()
}
```

根据提示，StandAloneStorageReader 结构体需要包含一个 **badger.Txn**，并且实现上面的三个方法，直接模仿 region\_reader 实现即可。

```go
type StandAloneStorageReader struct {
   txn *badger.Txn
}
```

获取badger.Txn的函数engine\_util并未给出，需要直接调用badger.DB.NewTransaction函数。

```go
func (db *DB) NewTransaction(update bool) *Txn
```

update 为真表示 Put/Delete 两个写操作，为假表示 Get/Scan 两个读操作。这里直接传入 false 即可。

**Write**

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

但其实，engine\_util 下面的这两个函数 PutCF 和 DeleteCF 也是用了事务，只不过无需让我们实现。

#### Server

这一部分需要完成四个函数 RawGet、RawPut、RawDelete、RawScan。同时利用上面完成的 Reader 结构体 和 Write 函数。

Get 和 Scan 需要使用 Reader。Put 和 Delete 需要使用 Write。

```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error)
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error)
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error)
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error)
```

RawScan 比较难以理解，从指定的 key 出发向后寻找 limit 个 KVPair 并将其返回。大致步骤如下：

1. 获取到 reader，记得defer
2. 通过reader 的 IterCF 函数，拿到一个 key 的迭代器。记得 defer
3. 通过迭代器的 Seek 函数进行定位。
4. 开始循环，等于 limit 时停止。每次循环开始时都要通过 Valid 函数校验迭代器是否有效。通过 Item 函数获得 DBItem。进而获得 key 和 value。每次循环结束时迭代器要调用 Next 函数进行移动。

### 通过

![pass1](https://cdn.staticaly.com/gh/yi-ge-dian/blog-image-hosting@master/TinyKV/pass1.13qsot4dpva8.webp)
