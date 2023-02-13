# 03-Storage 1

## 1. Course Outline

课程大纲如下：

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p>outline</p></figcaption></figure>

左侧是课程讲解目录，右侧是数据库执行过程。

## 2. Storage

我们将关注一个 "面向磁盘" 的DBMS架构，它假定数据库的主要存储位置是在非易失性磁盘上。 越接近CPU，存储就越快，容量越小，也更贵。

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>ARCHITECTURE</p></figcaption></figure>

DBMS 的组件负责管理非易失性和易失性存储之间的数据移动。

{% hint style="danger" %}
这课不讨论NVMe SSD -- non-volatile memory express.
{% endhint %}

