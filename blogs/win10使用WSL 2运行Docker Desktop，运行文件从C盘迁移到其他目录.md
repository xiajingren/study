# 前言
前几天重装系统，把系统升到了Windows 10 2004，然后在安装Docker Desktop（2.3.0.3版本）时发现跟以前不太一样了。现在Docker Desktop默认使用WSL 2来运行，而不是以前的Hyper-V。

# WSL
WSL：适用于 Linux 的 Windows 子系统。

- 什么是适用于 Linux 的 Windows 子系统？

> 适用于 Linux 的 Windows 子系统可让开发人员按原样运行 GNU/Linux 环境 - 包括大多数命令行工具、实用工具和应用程序 - 且不会产生虚拟机开销。

- 什么是 WSL 2？

> WSL 2 是适用于 Linux 的 Windows 子系统体系结构的一个新版本，它支持适用于 Linux 的 Windows 子系统在 Windows 上运行 ELF64 Linux 二进制文件。 它的主要目标是提高文件系统性能，以及添加完全的系统调用兼容性。

安装完后试了一下，最明显的感觉就是开启docker的速度大大提升！！！
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623194114380-1616116229.png)

但是以前设置镜像位置的功能不见了：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623194352720-365413236.png)
看官网说明，原来，启用WSL后，docker运行数据都在WSL发行版中，文件位置都只能由WSL管理！

安装docker后，docker会自动创建2个发行版：
- docker-desktop
- docker-desktop-data

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623195500964-442325184.png)

WSL发行版默认都是安装在C盘，在%LOCALAPPDATA%/Docker/wsl目录
docker的运行数据、镜像文件都存在%LOCALAPPDATA%/Docker/wsl/data/ext4.vhdx中，这对C盘空间紧张的人非常不友好。。。

# WSL发行版迁移
网上查了一下wsl发行版迁移，几乎都是说使用LxRunOffline.exe

经过我试验，LxRunOffline.exe确实可以迁移自己安装的发行版，却迁移不了docker自动创建的2个发行版！

最后只能去github提了个issues：https://github.com/docker/for-win/issues/7348

下面是操作方法：
1. 首先关闭docker

2. 关闭所有发行版：
`wsl --shutdown`

3. 将docker-desktop-data导出到D:\SoftwareData\wsl\docker-desktop-data\docker-desktop-data.tar（注意，原有的docker images不会一起导出）
`wsl --export docker-desktop-data D:\SoftwareData\wsl\docker-desktop-data\docker-desktop-data.tar`

4. 注销docker-desktop-data：
`wsl --unregister docker-desktop-data`

5. 重新导入docker-desktop-data到要存放的文件夹：D:\SoftwareData\wsl\docker-desktop-data\：
`wsl --import docker-desktop-data D:\SoftwareData\wsl\docker-desktop-data\ D:\SoftwareData\wsl\docker-desktop-data\docker-desktop-data.tar --version 2`

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623202849041-113421930.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623202919822-213119905.png)

只需要迁移docker-desktop-data一个发行版就行，另外一个不用管，它占用空间很小。

完成以上操作后，原来的%LOCALAPPDATA%/Docker/wsl/data/ext4.vhdx就迁移到新目录了：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200623204037008-1551744168.png)
重启docker，这下不用担心C盘爆满了！

参考：
https://docs.microsoft.com/zh-cn/windows/wsl/
https://docs.docker.com/docker-for-windows/wsl/