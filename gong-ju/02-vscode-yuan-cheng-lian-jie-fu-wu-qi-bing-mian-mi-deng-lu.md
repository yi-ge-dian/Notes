# 02 VSCode远程连接服务器并免密登陆

1. VSCode 安装 Reomte 插件
2. 打开配置文件，并书写配置文件

```
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
Host *（名字，无所谓）
    HostName *（服务器IP）
    User *（用户名）
```

{% hint style="info" %}
到这里可以通过密码进行远程登录
{% endhint %}

3. 客户端生成密钥

```bash
ssh-keygen -t rsa -C "1085266008@qq.com"
```

4. 上传密钥到远端服务器

```bash
ssh-copy-id -i .ssh/id_rsa.pub username@ip
```
