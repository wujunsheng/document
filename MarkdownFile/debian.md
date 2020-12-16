# Debian 10 
## 常用基础指令
> apt-get update

> apt-get install xxx

> apt-get remove xxx
---
## IP更改
1. 进入修改文件
    > nano /etc/network/interfaces

    > vim /etc/network/interfaces
2. 修改文件内容
    ```shell
    auto ens33
    iface ens33 inet dhcp
    # iface eth0 inet static
    # address 192.168.199.230
    # netmask 255.255.255.0
    # gateway 192.168.199.1
    ```
---
## root ssh权限
1. 安装ssh
    > apt install ssh
2. 修改配置文件目录
    > vim /etc/ssh/sshd_config
3. 修改
    > PermitRootLogin yes
4. 重启服务
    > systemctl restart ssh