---
description: 零部件老化，控制系统失灵，非线性作用
---

# Page 1

Fixed Percision Number

具有（可能）任意精度和比例的数字数据类型。当舍入误差不可接受时使用。

→ 示例：NUMERIC、DECIMAL

许多不同的实现。

→ 示例：使用附加元数据以精确的可变长度二进制表示形式存储。

→ 如果您放弃任意精度，可以降低成本。



<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Large Value

大多数DBMS不允许元组超过单个页面的大小。

为了存储大于页面的值，DBMS使用单独的溢出存储页面。

→ Postgres：TOAST（>2KB）

→ MySQL：溢出（页面大小>½）

→ SQL Server：溢出（>页面大小）

External Value Storage

有些系统允许您在外部文件中存储非常大的值。

被视为BLOB类型。

→ Oracle:BFILE数据类型

→ Microsoft:FILESTREAM数据类型

DBMS无法操作外部文件的内容。

→ 无耐久性保护。

→ 无事务保护



System Catalogs

DBMS在其内部目录中存储有关数据库的元数据。

→ 表、列、索引、视图

→ 用户，权限

→ 内部统计数据

几乎每个DBMS都将数据库的目录存储在其内部（即，作为表）。

→ 围绕元组包装对象抽象。

→ 用于“引导”目录表的专用代码。





您可以查询DBMS的内部INFORMATION\_SCHEMA目录以获取有关数据库的信息。

→ ANSI标准的只读视图集，提供有关数据库中所有表、视图、列和过程的信息

DBMS还具有非标准快捷方式来检索此信息

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

Conclusion

日志结构化存储是我们上节课讨论的面向页面的体系结构（面向元组）的一种替代方法。

存储管理器并非完全独立于DBMS的其余部分。









Database Workloads

On-Line Transaction Processing (OLTP)：联机事务处理

每次只读取/更新少量数据的快速操作

On-Line Analytical Processing (OLAP)：联机分析处理

读取大量数据以计算聚合的复杂查询

Hybrid Transaction + Analytical Processing：混合事务分析处理

同一个数据库实例中，既有oltp又有olap

图

Observation

关系模型没有指定DBMS必须将元组的所有属性一起存储在一个页面中。

这实际上可能不是某些工作负载的最佳布局…

OLTP：

→ 读取/更新与数据库中单个实体相关的少量数据的简单查询。

这通常是人们首先构建的应用程序

图

OLAP：

→ 读取跨越多个实体的数据库的大部分的复杂查询。

您可以在从OLTP应用程序收集的数据上执行这些工作负载。

Database Storage Models

DBMS可以以不同的方式存储元组，这对于OLTP或OLAP工作负载更好。

本学期到目前为止，我们一直在假设n元存储模型（也称为“行存储”）。





