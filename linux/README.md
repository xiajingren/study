# linux

## vm 安装centos 7

准备centos 7 iso 镜像文件，vm 新建虚拟机

![image-20210810164129944](README.assets/image-20210810164129944.png)

![image-20210810165054812](README.assets/image-20210810165054812.png)

右键虚拟机设置

![image-20210810165730855](README.assets/image-20210810165730855.png)

开启虚拟机，进入安装

![image-20210810165856151](README.assets/image-20210810165856151.png)

![image-20210810172008837](README.assets/image-20210810172008837.png)

![image-20210810170912274](README.assets/image-20210810170912274.png)

![image-20210810171853795](README.assets/image-20210810171853795.png)

**配置网络：**

修改 /etc/sysconfig/network-scripts/ifcfg-ens33

```bash
cd /etc/sysconfig/network-scripts/

ls

vim ifcfg-ens33

#
TYPE=Ethernet
BOOTPROTO=static
DEVICE=ens33
ONBOOT=yes
IPADDR=172.18.3.60
NETMASK=255.255.254.0
GATEWAY=172.18.2.1
DNS1=8.8.8.8
#

# 重启网络服务
service network restart
```

![image-20210810175742473](README.assets/image-20210810175742473.png)

![image-20210810182047470](README.assets/image-20210810182047470.png)

![image-20210810182210181](README.assets/image-20210810182210181.png)

## 常用命令

### 文件权限

![image-20210811094613567](README.assets/image-20210811094613567.png)

r = 4 (2^2)

w = 2(2^1)

x 1 1(2^0)

修改文件权限 `chmod 764 1.txt`（等价于`chmod u=rwx,g=rw,o=r 1.txt`）

tips: 7=4+2+1(rwx)  6=4+2(rw) 4=4(r)

![image-20210811100518922](README.assets/image-20210811100518922.png)



