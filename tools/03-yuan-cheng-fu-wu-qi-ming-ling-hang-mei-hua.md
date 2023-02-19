# 03 远程服务器命令行美化

1. update 和 upgrade

```bash
sudo apt-get update
sudo apt-get upgrade
```

2. 安装 zsh

```bash
sudo apt-get install zsh
```

3. 验证是否成功

```bash
zsh --version
```

4. 查看当前shell

```bash
echo $SHELL
```

5. 安装 oh-my-zsh

```bash
sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
```

6. 更换主题

```bash
vim ~/.zshrc
```

* ZSH\_THEME="agnoster"&#x20;

7. 配置生效

```bash
source ~/.zshrc
```
