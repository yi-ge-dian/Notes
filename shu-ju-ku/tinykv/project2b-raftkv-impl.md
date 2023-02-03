# Project2-RaftKV-B

## 项目要求

项目文档：[https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md](https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md)

该项目要求我们实现一个基于 raft 的高可用 kv 服务器。主要有三个部分：

1. 实现基本的 Raft 算法
2. 在 Raft 之上建立一个容错的 KV 服务器
3. 增加对 raftlog GC 和快照的支持

第二部分就是 2B 所要完成的部分。首先需要知道三个术语 Term，Peer，Region 的含义。图里面每一个彩色的小方格代表一个 peer。

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>Tiny架构图</p></figcaption></figure>

* `Store`：图中也称为 node，在代码中是 RaftStore。
* `Peer`：即图中每一个不同颜色的方块，在一个 Store 中，所有的 peer 公用一个底层存储，在代码中是 PeerStorage
* `Region`：即图中所有相同颜色所串连起来的集合，也称为 Raft Group，

{% hint style="info" %}
* 一个 store 可以有多个 Region，每个 Region 之间的 key space 不重叠。（如 Region1 负责存储的 key 为 A-C，Region2 负责存储的 key 为 D-E）
* 一个 store 上可以有多个 Peer，每个 Peer 对应一个 Region。
* 一个 Region 可以跨越多个 store，可以由多个 Peer 组成
{% endhint %}

> 但在 Project2 中一个 node 只有一个 Peer，一个集群也只有一个 Region。

