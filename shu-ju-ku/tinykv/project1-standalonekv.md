# Project1-StandaloneKV-CN

项目文档：[https://github.com/talent-plan/tinykv/blob/course/doc/project1-StandaloneKV.md](https://github.com/talent-plan/tinykv/blob/course/doc/project1-StandaloneKV.md)

在这个项目中，我们将会实现一个单机的、支持列族的 KV 存储 [gRPC](https://so.csdn.net/so/search?q=gRPC\&spm=1001.2101.3001.7020) 服务。单机意味着只有一个节点，而不是一个分布式系统。列族（CF, Column family）是一个类似 key 命名空间的术语，即同一个 key 在不同列族中的值是不同的。你可以简单地将多个 CF 视为独立的小型数据库。CF 在 Project4 中被用来支持事务模型，那时你会知道为什么 TinyKV 需要支持 CF。

该服务支持四个基本操作：Put/Delete/Get/Scan，它维护了一个简单的 key/value Pairs 的数据库，其中 key 和 value 都是字符串。`Put`：替换数据库中指定 CF 的某个 key 的 value。`Delete`：删除指定 CF 的 key 的 value。`Get`：获取指定 CF 的某个 key 的当前值。`Scan`：获取指定 CF 的一系列 key的 current value。

该项目可以分为2个步骤，包括:

1. 实现一个单机的存储引擎。
2. 实现原始的 key/value 服务处理程序。

## 代码

`gRPC` 服务在 `kv/main.go` 中被初始化，它包含一个 `tinykv.Server`，它提供了名为 `TinyKv` 的 `gRPC` 服务 。它由 `proto/proto/tinykvpb.proto` 中的 [protocol-buffer](https://developers.google.com/protocol-buffers) 定义，rpc 请求和响应的细节被定义在 `proto/proto/kvrpcpb.proto` 中。

一般来说，不需要改变 proto 文件，因为所有必要的字段都已经被定义了。但如果仍然需要改变，可以修改 proto 文件并运行 `make proto` 来更新 `proto/pkg/xxx/xxx.pb.go` 中生成的 go 相关代码。

此外，`Server` 依赖于一个 `Storage`，这是一个需要为单机存储引擎实现的接口，位于 `kv/storage/standalone_storage/standalone_storage.go` 中。一旦 `StandaloneStorage` 中实现了接口 `Storage`，就可以用它实现 `Server` 的原始 `key/value` 服务。

### 实现单机存储引擎

第一个任务是实现一个 [badger](https://github.com/dgraph-io/badger) key/value API 的包装器。gRPC 服务器的服务依赖于一个在`kv/storage/storage.go` 中定义的 `Storage`。在这种情况下，单机的存储引擎只是 badger key/value API的一个封装器，它由两个方法提供：

```go
type Storage interface {
    // Other stuffs
    Write(ctx *kvrpcpb.Context, batch []Modify) error
    Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

`Write` 应该提供一种方式，将一系列的修改应用到内部状态，在这种情况下，内部状态是一个 badger 的实例。

`Reader` 应该返回一个支持键/值的点取和扫描快照操作的 `StorageReader`。

并且你现在不需要考虑 `kvrpcpb.Context`，它在接下来的项目中才会使用。

{% hint style="info" %}
* 你应该使用 [badger.Txn](https://godoc.org/github.com/dgraph-io/badger#Txn) 来实现 <mark style="background-color:blue;">Reader</mark> 函数，因为 badger 提供的事务处理程序可以提供键和值的一致快照。
* Badger 没有给出对列族的支持。 engine\_util 包（<mark style="background-color:blue;">kv/util/engine\_util</mark>）通过给键添加前缀来模拟列族。例如，一个属于特定列族 cf 的 `key` 被存储为 <mark style="background-color:blue;">${cf}\_${key}</mark>。它封装了 <mark style="background-color:blue;">badger</mark> 以提供对 CF 的操作，还提供了许多有用的辅助函数。所以你应该通过 <mark style="background-color:blue;">engine\_util</mark> 提供的方法进行所有读写操作。请阅读 <mark style="background-color:blue;">util/engine\_util/doc.go</mark> 以了解更多。
* TinyKV fork 了 <mark style="background-color:blue;">badger</mark> 原始版本，并进行了一些修正，所以需要使用 <mark style="background-color:blue;">github.com/Connor1996/badger</mark> 而不是 <mark style="background-color:blue;">github.com/dgraph-io/badger</mark>。
* 别忘了为 <mark style="background-color:blue;">badger.Txn</mark> 调用 <mark style="background-color:blue;">Discard()</mark>，并在丢弃前关闭所有迭代器。
{% endhint %}

### 实现服务处理程序

这个项目的最后一步是使用已实现的存储引擎来建立原始键/值服务处理程序，包括RawGet/RawScan/RawPut/RawDelete。处理程序已经为你定义好了，你只需要在 `kv/server/raw_api.go` 中填上实现。一旦完成，记得运行 `make project1` 以通过测试。
