# 安装 Neo4j

1. 安装 java 11

```bash
sudo apt-get install openjdk-11-jre
sudo apt-get install openjdk-11-jdk
```

2. 创建文件夹并进入

```bash
mkdir /usr/local/neo4j
cd /usr/local/neo4j
```

3. 安装社区版的 neo4j，注意切换版本

```bash
wget https://neo4j.com/artifact.php?name=neo4j-community-4.2.2-unix.tar.gz
```

4. 解压

```bash
tar -xzvf artifact.php\?name=neo4j-community-4.2.2-unix.tar.gz
```

5. 删除安装包

```bash
rm -rf artifact.php\?name=neo4j-community-4.2.2-unix.tar.gz
```

6. 进入文件夹

```bash
cd neo4j-community-4.2.2
```

7. 启动

```bash
bin/neo4j start
```

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

如果是访问服务器，开启下面四个

```apacheconf
# 允许远程访问
dbms.connectors.default_listen_address=0.0.0.0
# 开启bolt服务，默认端口7687
dbms.connector.bolt.listen_address=:7687
# 开启http服务，默认端口7474
dbms.connector.http.listen_address=:7474
# 开启https服务，默认端口7473
dbms.connector.https.listen_address=:7473
```

注意开启一下防火墙，不然端口可能访问不到。

Neo4j。http://服务器ip:7474/browser/，初次登录账号和密码均为neo4j，点击连接后，会出现修改密码的页面，自行修改密码即可。至此，安装Neo4j完成。

