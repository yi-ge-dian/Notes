# VSCode远程连接服务器并免密登陆

* 客户端生成密钥

```bash
ssh-keygen -t rsa -C "1085266008@qq.com"
```

* VSCode 安装 Reomte 插件
* 打开配置文件

```
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
Host *
    HostName *
    User *
```

* 传密钥

```bash
ssh-copy-id -i .ssh/id_rsa.pub root@*
```
