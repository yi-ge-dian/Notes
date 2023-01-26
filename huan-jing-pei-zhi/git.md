# git

* 生成密钥的命令

```bash
ssh-keygen -t rsa -C "1085266008@qq.com"
```

* 设置邮箱和用户名

```bash
git config --global user.name "yi-ge-dian"
git config --global user.email "1085266008@qq.com"
```

* windows 下上传密钥到服务器

{% code overflow="wrap" %}
```bash
Get-Content $ewnv:USERPROFILE\.ssh\id_rsa.pub | ssh dongwenlong@192.168.140.128 "cat >> .ssh/authorized_keys" 
```
{% endcode %}

* git 远程分支回滚

```bash
git reset --hard xxxx(commit-id)
git push -f origin master(branch-name) 
```
