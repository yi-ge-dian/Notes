# TinyKV

项目文档：[https://github.com/talent-plan/tinykv](https://github.com/talent-plan/tinykv)

## TinyKV 课程

TinyKV 课程建立了一个具有 Raft 共识算法的键值存储系统。它的灵感来自 [MIT 6.824](https://pdos.csail.mit.edu/6.824/) 和[TiKV](https://github.com/tikv/tikv) 项目。

在完成本课程后，你将对实现一个水平可扩展的、高可用的、支持分布式事务的键值存储服务具有一定的知识。同时，你将对 TiKV 架构和实现有更好的理解。

### 课程大纲

整个项目一开始只有一个 key-value server 和一个 scheduler server 的代码框架 - 你需要一步步地完成它的核心逻辑。

* [Standalone KV](https://github.com/talent-plan/tinykv/blob/course/doc/project1-StandaloneKV.md)
  * 实现一个单机的存储引擎
  * 实现原始键值服务处理程序。
* [Raft KV](https://github.com/talent-plan/tinykv/blob/course/doc/project2-RaftKV.md)
  * 实现基本的 Raft 算法。&#x20;
  * 在 Raft 之上建立一个容错的 KV 服务器。
  * 增加对 Raft 日志垃圾收集和快照的支持。
* [Multi-raft KV](https://github.com/talent-plan/tinykv/blob/course/doc/project3-MultiRaftKV.md)
  * 对 Raft 算法实现成员变更和领导力变更。
  * 在 Raft 存储上实现配置变更和 Region 分割。
  * 实现一个基本的调度器。
* [Transaction](https://github.com/talent-plan/tinykv/blob/course/doc/project4-Transaction.md)
  * 实现多版本并发控制层。
  * 实现 `KvGet`、`KvPrewrite`和 `KvCommit` 请求的处理程序。
  * 实现 `KvScan`、`KvCheckTxnStatus`、`KvBatchRollback` 和 `KvResolveLock` 请求的处理程序。

### 代码结构

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>TinyKV 代码结构</p></figcaption></figure>

类似于 TiDB + TiKV + PD 这种将存储和计算分离的架构，TinyKV 只专注于分布式数据库系统的存储层。如果你对 SQL 层也感兴趣，请看  [TinySQL](https://github.com/tidb-incubator/tinysql)。除此之外，还有一个叫做TinyScheduler 的组件作为整个 TinyKV 集群的中心控制，它从 TinyKV 的心跳中收集信息。之后，TinyScheduler 可以生成调度任务并将任务分配给 TinyKV 的实例。所有的实例都通过RPC进行通信。

整个项目的代码目录如下：

* `kv` 包含键值存储的实现。&#x20;
* `raft` 包含了 Raft 共识算法的实现。&#x20;
* `scheduler` 包含了 TinyScheduler 的实现，它负责管理 TinyKV 节点和生成时间戳。&#x20;
* `proto` 包含了节点和进程之间所有通信的实现，使用 gRPC 的协议缓冲区。其包含了 TinyKV 使用的协议定义，以及生成的你可以使用的 Go 代码。&#x20;
* `log` 包含了根据级别输出日志的工具。

### 阅读清单

我们为分布式存储系统的知识提供了一个[阅读清单](https://github.com/talent-plan/tinykv/blob/course/doc/reading\_list.md)。虽然不是所有的书都与本课程高度相关，但它们可以帮助你构建这个领域的知识体系。

另外，我们鼓励你阅读 TiKV 和 PD 的设计概述，以便对你要建立的东西有一个总体印象。

* TiKV，数据存储的设计（[英文](https://en.pingcap.com/blog/tidb-internal-data-storage)，[中文](https://pingcap.com/zh/blog/tidb-internal-1)）
* PD，调度的设计（[英文](https://en.pingcap.com/blog/tidb-internal-scheduling)，[中文](https://pingcap.com/zh/blog/tidb-internal-3)）

### 从源码构建 TinyKV

#### 前置条件

* `git` ：TinyKV 的源代码以 git 仓库的形式托管在 GitHub 上。为了使用 git 仓库，请[安装 git](https://git-scm.com/downloads)。
* `go`: TinyKV是一个 Go 项目。要从源代码中构建 TinyKV，请[安装](https://golang.org/doc/install)版本大于或等于 1.13 的 go。

#### 克隆

克隆源代码到你的开发机器上。

```bash
git clone https://github.com/tidb-incubator/tinykv.git
```

#### 构建

从源代码构建 TinyKV。

```bash
cd tinykv
make
```

它将 `tinykv-server` 和 `tinyscheduler-server` 的二进制文件构建到 `bin dir`。

### 用 TinySQL 来运行TinyKV

1. 按照其文档获取 `tinysql-server`。&#x20;
2. 把 `tinyscheduler-server`、`tinykv-server` 和 `tinysql-server` 的二进制文件放到一个目录中。&#x20;
3. 在二进制文件目录下，运行以下命令：

```bash
mkdir -p data
./tinyscheduler-server
./tinykv-server -path=data
./tinysql-server --store=tikv --path="127.0.0.1:2379"
```

现在你可以用一个官方的 MySQL 客户端连接到数据库：

```bash
mysql -u root -h 127.0.0.1 -P 4000
```

### 贡献

非常感谢任何反馈和贡献。如果你想加入开发，请查看 [issues](https://github.com/tidb-incubator/tinykv/issues)。
