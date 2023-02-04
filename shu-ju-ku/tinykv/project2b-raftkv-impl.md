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
5. 新开辟一个协程，接收来自其他节点的 msg，然后根据 msg 里面的 region id 将其路由到指定 region 的 peer 上。同时也会将 peer 获取需要发送给其他节点的信息，发到其他的 RaftStore 上

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

* raftDB
  * raft log
    * key: region\_id + log\_idx
    * value: entry
  * raft state
    * key: region\_id
    * value: RaftLocalState
* kvDB
  * data
  * apply state
    * key: region\_id
    * value: RaftApplyState
  * region state
    * key: region\_id
    * value: RegionLocalState

1.
