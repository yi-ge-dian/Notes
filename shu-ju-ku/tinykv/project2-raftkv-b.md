# Project2-RaftKV-B

## 项目要求

项目文档：[https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md](https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md)

该项目要求我们实现一个基于 raft 的高可用 kv 服务器。主要有三个部分：

1. 实现基本的 Raft 算法
2. 在 Raft 之上建立一个容错的 KV 服务器
3. 增加对 raftlog GC 和快照的支持

第二部分就是 2B 所要完成的部分。首先需要知道三个术语 Term，Peer，Region 的含义。图里面每一个彩色的小方格代表一个 peer。

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>Tiny架构图</p></figcaption></figure>

* `Store`：每一个节点叫做一个 store，也就是一个节点上面只有一个 Store。代码里面叫 RaftStore，后面统一使用 RaftStore 代称。
* `Peer`：一个 RaftStore 里面会包含多个 peer，一个 RaftStore 里面的所有 peer 公用同一个底层存储，也就是多个 peer 公用同一个 badger 实例。
* `Region`：一个 Region 叫做一个 Raft group，一个 region 包含多个 peer，这些 peer 散落在不同的 RaftStore 上，Region 处理的是某一个范围的数据。

{% hint style="info" %}
一个 Region 在一个 RaftStore 上最多只有一个 peer。

因为一个 region 里面所有 peer 的数据是一致的，如果在同一个 RaftStore 上面有两个一样的 peer，则会毫无意义，不能增加容灾性。
{% endhint %}

