# 01 Windows下安装Ubuntu

1. 安装界面太小，按 alt+f7 加鼠标滚轮调动
2. 换源
   * 利用百度搜索 [https://mirror.tuna.tsinghua.edu.cn/help/ubuntu](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu)
   * sudo apt-get update
   * sudo apt-get upgrade
3. 虚拟机屏幕无法适应，要安装 vmware tools，可以手动，可以自动，自动如下（要先换源，不然由于维护软件包，无法安装）

```bash
sudo apt install open-vm-tools-desktop -y
sudo reboot
```

4. 虚拟机改代理，前面的数字为主机 ipconfig 下的 ip ，端口为本机代理端口

<figure><img src="../.gitbook/assets/image-20221130190231790.png" alt=""><figcaption><p>修改虚拟机代理</p></figcaption></figure>

5. 虚拟机终端代理命令

```bash
git config --global https.proxy https://ip:7890
git config --global http.proxy http://ip:7890
```

6. 远程连接时，虚拟机需要 ssh 服务

```bash
sudo apt-get install openssh-server
sudo /etc/init.d/ssh start
```
