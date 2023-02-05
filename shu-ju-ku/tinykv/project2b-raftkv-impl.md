# Project2B-RaftKV-Impl

## 框架

首先需要知道三个术语 Term，Peer，Region 的含义。

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption><p>Tiny架构图</p></figcaption></figure>

* `Store`：图中也称为 node，在代码中是 RaftStore。
* `Peer`：即图中每一个彩色的小方块，在一个 Store 中，所有的 peer 公用一个底层存储，在代码中是 PeerStorage
* `Region`：即图中所有相同颜色所串连起来的集合，也称为 Raft Group，

{% hint style="info" %}
* 一个 store 可以有多个 Region，每个 Region 之间的 key space 不重叠。（如 Region1 负责存储的 key 为 A-C，Region2 负责存储的 key 为 D-E）
* 一个 store 上可以有多个 Peer，每个 Peer 对应一个 Region。
* 一个 Region 可以跨越多个 store，可以由多个 Peer 组成
{% endhint %}

> 但在 Project2 中一个 node 只有一个 Peer，一个集群也只有一个 Region。

测试执行流程如下：

{% tabs %}
{% tab title="code" %}
```go
1. cfg := config.NewTestConfig()
2. cluster := NewTestCluster(nservers, cfg)
3. cluster.Start()
```
{% endtab %}

{% tab title="text" %}
1. 创建一个配置
2. 根据配置和创建的服务器数目，建立一个集群。
3. 启动集群
{% endtab %}
{% endtabs %}

启动的过程如下：

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>启动流程</p></figcaption></figure>

## Raft Store

它在节点启动的时候被创建，负责维护在该节点上所有的 region 和对应的 peer。

1. 进行初始化结构体的初始化。
2. 加载peers，如果底层存储着之前 peer 信息，那么根据存储的信息加载，否则就新建。
3. 将 peers 注册进 router
4. 开辟许多协程来启动 worker：RaftWorker，StoreWorker，SplitCheckWorker，RegionWorker，RaftLogGcWoker，SchedulerWorker
5. 新开辟一个协程，接收来自其他节点的 msg，然后根据 msg 里面的 region id 将其路由到指定 region 的 peer 上。同时也会将 peer 获取需要发送给其他节点的信息，发到其他的 RaftStore 上。

### Raft Woker

RaftWorker的作用是运行 raft commands 和 apply raft logs。

```go
func (rw *raftWorker) run(closeCh <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	var msgs []message.Msg
	for {
		msgs = msgs[:0]
		select {
		case <-closeCh:
			return
		case msg := <-rw.raftCh:
			msgs = append(msgs, msg)
		}
		pending := len(rw.raftCh)
		for i := 0; i < pending; i++ {
			msgs = append(msgs, <-rw.raftCh)
		}
		peerStateMap := make(map[uint64]*peerState)
		for _, msg := range msgs {
			peerState := rw.getPeerState(peerStateMap, msg.RegionID)
			if peerState == nil {
				continue
			}
			newPeerMsgHandler(peerState.peer, rw.ctx).HandleMsg(msg)
		}
		for _, peerState := range peerStateMap {
			newPeerMsgHandler(peerState.peer, rw.ctx).HandleRaftReady()
		}
	}
}
```

取一个 batch 的 msg 后，逐条使用 peerMsgHandler 处理消息。先通过 `HandleMsg()`，RaftStore 将收到的信息告诉该 peer，然后该 peer 进行一些列的处理，RaftStore 再通过 `HandleRaftReady()` 处理 RawNode 产生的 Ready。

### PeerMsgHandler

主要有两个方法，`HandleMsg()` 和 `HandleRaftReady()`

#### HandleMsg

* **MsgTypeRaftMessage**，来自外部接收的 msg，在接收 msg 前会进行一系列的检查，最后通过 RawNode 的 `Step()` 方法直接输入，不需要回复 proposal。
* **MsgTypeRaftCmd**，通常都是从 client 或自身发起的请求，比如 Admin 管理命令，read/write 等请求，需要回复 proposal。
* **MsgTypeTick**，驱动 RawNode 的 tick。
* **MsgTypeSplitRegion**，触发 region split，在 project 3B split 中会用到。
* **MsgTypeRegionApproximateSize**，修改自己的 ApproximateSize 属性，不用管。
* **MsgTypeGcSnap**，清理已经安装完的 snapshot。
* **MsgTypeStart**，启动 peer，新建的 peer 的时候需要发送这个请求启动 peer，在 Project3B split 中会遇到。

这里需要实现 **MsgTypeRaftCmd**  的 `proposeRaftCommand()` 方法。该方法将 client 的请求包装成 entry 传递给 raft 层。

客户端会发送一个 msg（类型为 `*raft_cmdpb.RaftCmdRequest`）。其中，该结构中包含了 `Requests` 字段和 `AdminRequest` 字段，即两种命令请求。在 project2B 中，暂时只用到 Requests 字段。核心分为两步：&#x20;

1. 在请求发过来的时候，同时会附带一个回调函数，我们需要记录这个回调函数与本条日志的索引，等待 apply 这条记录到状态机执行完后，就回调它，并将返回值返回。简单来说，就是封装回调。
2. 将这个 Msg 序列化，通过 partA 的 rawnode 封装的 Propse 函数将消息传递进去。

<details>

<summary>proposals</summary>

当上层需要让 peer 执行某条命令时，发过来的请求类型是 RaftCmdRequest，需要执行的命令则全部在 Requests 字段中，但是这个格式并不能直接被 Raft 所执行，需要调用 <mark style="background-color:blue;">proposeRaftCommand()</mark> 方法把类型转化为 Raft 所能执行的数据格式。之后调用 partA 的 Step函数进行执行。

<mark style="color:red;">上层如何知道底层的这些命令真的被执行并且得到命令的执行结果呢？</mark>

proposals 是封装的回调结构体，是一个切片。这就是 callback 的作用。每当 peer 接收到 RaftCmdRequest 时，就会给里面的 Requests 一个 callback，然后封装成 proposal，其中 term 就为该 Requests 对应 entries 生成时的 term、index 就是该 Requests 当前最后一条日志+1

当 rawNode 返回一个 Ready 回去时，说明上述那些 entries 已经完成了同步，因此上层就可以通过 HandleRaftReady 对这些 entries 进行 apply（即执行底层读写命令）。

每执行完一次 apply，都需要对 proposals 中的相应 Index 的 proposal  调用 callback.Done()，表示这条命令已经完成了（如果是 Get 命令还会返回取到的 value），然后从中删除这个 proposal。&#x20;

</details>

#### HandleRaftReady()

1. 通过 `d.RaftGroup.HasReady()` 方法判断是否有新的 Ready，没有的话就什么都不用处理。
2. 如果有 Ready 先调用 `d.peerStorage.SaveReadyState(&ready)` 将 Ready 中需要持久化的内容保存到 badger。如果 Ready 中存在 snapshot，则应用它，后面会细说。
3. 然后调用 `d.Send(d.ctx.trans, ready.Messages)` 方法将 Ready 中的 Messages 发送出去。
4. Apply `ready.CommittedEntries` 中的 entry。
5. 调用 `d.RaftGroup.Advance(ready)` 方法推进 RawNode。



### PeerStorage

Part A 中使用的是 MemoryStorage  在内存中进行持久化。PeerStorage 使用 badger 做持久化。

```go
type PeerStorage struct {
    // others
    // Engine include two badger instance: Raft and Kv
    Engines *engine_util.Engines
    // others
}
```

Engines 里面有两个 badger 实例，各自存储了一些信息。不同类型的信息使用特殊的前后缀用作区分。

raftDB 存储

* raft log
  * key: region\_id + log\_idx
  * value: entry
* raft state
  * key: region\_id
  * value: RaftLocalState

kvDB 存储：

* data
* apply state
  * key: region\_id
  * value: RaftApplyState
* region state
  * key: region\_id
  * value: RegionLocalState

#### SaveReadyState()

这个方法主要是负责持久化 Ready 中的数据。

1. 判断是否有 Snapshot，如果有 Snapshot，先调用 `ApplySnapshot()` 方法应用。可以通过 `raft.isEmptySnap()` 方法判断是否存在 Snapshot。
2. 调用 `Append()` 方法将需要持久化的 entries 保存到 raftDB。
3. 保存 ready 中的 HardState 到 `ps.raftState.HardState`，注意先使用 `raft.isEmptyHardState()` 进行判空。
4. 持久化 RaftLocalState 到 raftDB。
5. 最后通过 `MustWriteToDB()` 方法写入。

