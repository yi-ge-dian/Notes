# Project2-Raft-CN

项目文档：[https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md](https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md)

Raft 是一种共识算法，其设计理念是易于理解。你可以在 [Raft 网站](https://raft.github.io/) 上阅读关于 Raft 本身的材料，Raft的可视化交互，以及其他的资源，包括 [Raft 的论文](https://raft.github.io/raft.pdf)。

在这个项目中，你将基于 raft 实现一个高可用 kv 服务器，这不仅需要你实现 Raft 算法，还需要你实际使用它，并给你带来更多的挑战，比如用 `badger` 管理 raft 的持久化状态，为快照消息添加流控制等。

该项目有3个部分需要你去做，包括。

* 实现基本的 Raft 算法
* 在 Raft 之上建立一个容错的KV服务器
* 增加对 raftlog GC 和快照的支持

## Part A

### 代码

在这一部分，你将实现基本的 raft 算法。你需要实现的代码在 `raft/`。在 `raft/`里面，有一些代码框架和测试案例。你要在这里实现的 raft 算法有一个精心设计的与上层应用程序的接口。此外，它使用一个逻辑时钟（这里命名为 tick）而不是物理时钟，来测量选举和心跳是否超时。也就是说，不要在 Raft 模块本身设置一个计时器，上层应用程序负责通过调用 `RawNode.Tick()` 来推进逻辑时钟。除此之外，消息的发送和接收以及其他事情都是异步处理的，何时真正做这些事情也是由上层应用决定的（更多细节见下文）。例如，Raft 不会在等待任何请求消息的响应时阻塞。

在实现之前，请先查看这部分的提示。另外，你应该粗略地看一下 proto 文件 `proto/proto/eraftpb.proto`。Raft 发送和接收消息以及相关的结构都是在那里定义的，你将使用它们来实现。注意，与 Raft 论文不同的是，它将 Heartbeat 和 AppendEntries 分为不同的消息，以使逻辑更加清晰。

这部分可分为三个步骤，包括：

* 领导选举
* 日志复制
* Raw node 接口

### 实现 Raft 算法

`raft/raft.go` 中的 `raft.Raft` 提供了 Raft 算法的核心，包括消息处理、驱动逻辑时钟等。关于更多的实现指南，请查看 `raft/doc.go` ，其中包含了概要设计和这些 MessageTypes 负责的内容。

#### **领导选举**

为了实现领导选举，你可能想从 `raft.Raft.tick()` 开始，它被用来通过一个 tick 驱动内部逻辑时钟，从而触发选举超时或心跳超时。你现在不需要关心消息的发送和接收逻辑。如果你需要发送消息，只需将其推送到 `raft.Raft.msgs` ，Raft 收到的所有消息将被传递到 `raft.Raft.Step()`。测试代码将从 `raft.Raft.msgs` 获取消息，并通过 `raft.Raft.Step()` 传递响应消息。`raft.Raft.Step()` 是消息处理的入口，你应该处理像 `MsgRequestVote`、`MsgHeartbeat` 这样的消息及其响应。也请实现 test stub 函数，并让它们被正确调用，如`raft.Raft.becomeXXX`，当 Raft 的角色改变时，它被用来更新 Raft 的内部状态。

你可以运行 `make project2aa` 来测试实现，并在这部分的最后看到一些提示。

#### **日志复制**

为了实现日志复制，你可能想从处理发送方和接收方的 `MsgAppend` 和 `MsgAppendResponse` 开始。查看 `raft/log.go` 中的 `raft.RaftLog`，这是一个辅助结构体，可以帮助你管理 raft 日志，在这里你还需要通过 `raft/storage.go` 中定义的 `Storage` 接口与上层应用进行交互，以获得日志条目和快照等持久化数据。

你可以运行 `make project2ab` 来测试实现，并在这部分的最后看到一些提示。

### 实现 Raw node 接口

`raft/rawnode.go` 中的 `raft.RawNode` 是我们与上层应用程序交互的接口，`raft.RawNode` 包含 `raft.Raft` 并提供一些封装函数，如 `RawNode.Tick()` 和 `RawNode.Step()`。它还提供了 `RawNode.Propose()` 来让上层应用提出新的 raft 日志。

另一个重要的结构 `Ready` 也在这里定义。在处理消息或推进逻辑时钟时，`raft.Raft` 可能需要与上层应用进行交互，比如：

* 向其他 peers 发送消息
* 将日志条目进行持久化存储
* 将 term、commit index 和 vote 等状态持久划存储
* 将 committed 的日志条目 apply 于状态机
* 等等

但是这些交互并不是立即发生的，相反，它们被封装在 `Ready` 中，并由 `RawNode.Ready()` 返回给上层应用。这取决于上层应用程序何时调用 `RawNode.Ready()` 并处理它。在处理完返回的 `Ready` 后，上层应用程序还需要调用一些函数，如 `RawNode.Advance()` 来更新 `raft.Raft` 的内部状态，如 applied index、stabled log index 等。

你可以运行 `make project2ac` 来测试实现，运行 `make project2a` 来测试整个A部分。

{% hint style="info" %}
* 在 <mark style="background-color:blue;">raft.Raft</mark>、<mark style="background-color:blue;">raft.RaftLog</mark>、<mark style="background-color:blue;">raft.RawNode</mark> 和 <mark style="background-color:blue;">eraftpb.proto</mark> 上添加任何你需要的状态。
* 测试文件假设，第一次启动的 Raft 应该有 term 0。
* 测试文件假设，新当选的领导应该在其任期内附加一个 noop 日志项。
* 测试文件假设，一旦领导者推进其 commit index，它将通过 <mark style="background-color:blue;">MessageType\_MsgAppend</mark> 消息广播提交索引。
* 测试文件假设，没有为本地消息：<mark style="background-color:blue;">MessageType\_MsgHup</mark>、<mark style="background-color:blue;">MessageType\_MsgBeat</mark> 和 <mark style="background-color:blue;">MessageType\_MsgPropose</mark> 设置 term。
* 在领导者和非领导者之间，追加的日志项是相当不同的，有不同的来源、检查和处理，要注意这一点。
* 不要忘了选举超时在 peers 之间应该是不同的。
* <mark style="background-color:blue;">rawnode.go</mark> 中的一些封装函数可以用 <mark style="background-color:blue;">raft.Step(local message)</mark> 实现。
* 当启动一个新的 raft 时，从 <mark style="background-color:blue;">Storage</mark> 中获取最后的稳定状态来初始化 <mark style="background-color:blue;">raft.Raft</mark> 和 <mark style="background-color:blue;">raft.RaftLog</mark>。
{% endhint %}

## Part B

在这一部分中，你将使用 Part A 中实现的 Raft 模块建立一个容错的 KV 存储服务。服务将是一个复制状态机，由几个使用 Raft 进行复制的 KV 服务器组成。KV 服务应该继续处理客户的请求，只要大多数的服务器是活的并且可以通信，尽管有其他的故障或网络分区发生。

在 Project1 中，你已经实现了一个独立的 KV 服务器，所以你应该已经熟悉了 KV 服务器的 API 和 Storage 接口。

在介绍代码之前，你需要先了解三个术语：`Store`、`Peer` 和 `Region`，它们定义在`proto/proto/metapb.proto` 中。

* Store 代表 tinykv-server 的一个实例
* Peer 代表运行在 Store 上的 Raft 节点
* Region 是 Peer 的集合，也叫 Raft 组

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>region 示意图</p></figcaption></figure>

为了简单起见，对于 project2，一个 Store 上 只有一个 Peer，一个集群中只有一个 Region。所以你现在不需要考虑 Region 的范围。多个 Region 将在 project3 进一步引入。

### 代码

首先，你应该看看位于 `kv/storage/raft_storage/raft_server.go` 的 `RaftStorage` 代码，它也实现了 `Storage` 接口。与 `StandaloneStorage` 直接从底层引擎写入或读取不同，它首先将每个写入或读取请求发送到 Raft，然后在 Raft 提交请求后对底层引擎进行实际写入和读取。通过这种方式，它可以保持多个存储之间的一致性。

`RaftStorage` 主要创建一个 `Raftstore` 来驱动 Raft。当调用 Reader 或 Write 函数时，它实际上是通过通道向 raftstore 发送一个定义在 `proto/proto/raft_cmdpb.proto` 中的 `RaftCmdRequest`，其中有四种基本的命令类型（Get/Put/Delete/Snap）（该通道的接收者是 `raftWorker` 的 `raftCh`），在 Raft 提交和应用该命令后返回响应。而 `Write` 和 `Read` 函数的 `kvrpc.Context` 参数现在很有用，它从客户端的角度携带了 Region 信息，并作为 `RaftCmdRequest` 的头传递。也许这些信息是错误的或陈旧的，所以 raftstore 需要检查它们并决定是否提出请求。

然后，就来到了 TinyKV 的核心 —— raftstore。这个结构有点复杂，你可以阅读一下 TiKV 的参考资料，让你对这个设计有更好的了解。

* [https://pingcap.com/blog-cn/the-design-and-implementation-of-multi-raft/#raftstore](https://pingcap.com/blog-cn/the-design-and-implementation-of-multi-raft/#raftstore) （中文）
* [https://pingcap.com/blog/design-and-implementation-of-multi-raft/#raftstore](https://pingcap.com/blog/design-and-implementation-of-multi-raft/#raftstore) （英文）

raftstore 的入口是 `Raftstore`，见 `kv/raftstore/raftstore.go`。它启动了一些 worker 来异步处理特定的任务，但大部分现在并不用，所以你可以直接忽略它们。你所需要关注的是 `raftWorker`。(kv/raftstore/raft\_worker.go)

整个过程分为两部分：

1. raft worker 轮询 `raftCh` 以获得消息，这些消息包括驱动 Raft 模块的基本 tick 和作为 Raft 日志项的 Raft 命令。
2. 它从 Raft 模块获得并处理 ready，包括发送 raft 消息、持久化状态、将提交的日志项应用到状态机。一旦应用，将响应返回给客户。

### 实现 PeerStorage

peer storage 是通过 Part A 中的 `Storage` 接口进行交互，但是除了 raft 日志之外，peer storage 还管理着其他持久化的元数据，这对于重启后恢复到一致的状态机非常重要。此外，在 `proto/proto/raft_serverpb.proto` 中定义了三个重要状态。

* RaftLocalState：用于存储当前 Raft hard state 和 Last Log Index。
* RaftApplyState。用于存储 Raft applied 的 Last Log Index 和一些 truncated Log 信息。
* RegionLocalState。用于存储 Region 信息和该 Store 上的 Peer State。Normal表示该 peer 是正常的，Tombstone表示该 peer 已从 Region 中移除，不能加入Raft 组。

这些状态被存储在两个badger实例中：raftdb 和 kvdb。

* raftdb 存储 raft 日志和 `RaftLocalState`。
* kvdb 在不同的列族中存储键值数据，`RegionLocalState` 和 `RaftApplyState`。你可以把 kvdb 看作是Raft论文中提到的状态机。

格式如下，在 `kv/raftstore/meta` 中提供了一些辅助函数，并通 `writebatch.SetMeta()` 将其保存到 badger。

| Key                | KeyFormat                          | Value            | DB   |
| ------------------ | ---------------------------------- | ---------------- | ---- |
| raft\_log\_key     | 0x01 0x02 region\_id 0x01 log\_idx | Entry            | raft |
| raft\_state\_key   | 0x01 0x02 region\_id 0x02          | RaftLocalState   | raft |
| apply\_state\_key  | 0x01 0x02 region\_id 0x03          | RaftApplyState   | kv   |
| region\_state\_key | 0x01 0x03 region\_id 0x01          | RegionLocalState | kv   |

> 你可能想知道为什么 TinyKV 需要两个 badger 实例。实际上，它只能使用一个badger 来存储 Raft 日志和状态机数据。分成两个实例只是为了与TiKV的设计保持一致。

这些元数据应该在 `PeerStorage` 中创建和更新。当创建 PeerStorage 时，见`kv/raftstore/peer_storage.go` 。它初始化这个 Peer 的 RaftLocalState、RaftApplyState，或者在重启的情况下从底层引擎获得之前的值。注意，RAFT\_INIT\_LOG\_TERM 和 RAFT\_INIT\_LOG\_INDEX 的值都是5（只要大于1），但不是0。之所以不设置为0，是为了区别于 peer 在更改 conf 后被动创建的情况。你现在可能还不太明白，所以只需记住它，细节将在 project3b 中描述，当你实现 conf change 时。
