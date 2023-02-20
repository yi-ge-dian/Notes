# hw01-SQL

## 0. 准备工作

跟着官网的步骤一步一步做即可。

{% embed url="https://15445.courses.cs.cmu.edu/fall2022/homework1/" %}
官网地址
{% endembed %}

但是如果使用DataGrip 来连接 Windows 下的 WSL2 启动时，会报错误。

{% hint style="danger" %}
SQL错误 \[5]: \[SQLITE\_BUSY] The database file is locked (database is locked) \[SQLITE\_BUSY] The database file is locked (database is locked) \[SQLITE\_BUSY] The database file is locked (database is locked)
{% endhint %}

解决方法就是把数据库文件放到 Windows 下面来，不要放在 WSL2 的路径。

## 1. 记录步骤

