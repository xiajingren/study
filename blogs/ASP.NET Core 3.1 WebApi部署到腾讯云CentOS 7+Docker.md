# 一、准备
- 首先需要有一台CentOS服务器，安装最新版Docker，配置镜像加速等，安装方法网上很多，下面是一些相关指令：

`yum install -y yum-utils device-mapper-persistent-data lvm2`
`yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
`yum install docker-ce docker-ce-cli containerd.io`
`systemctl start docker`
`systemctl enable docker` 开机启动
配置阿里镜像加速：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519190327207-1538418757.png)

以下是我安装好的docker：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519170626495-1457697035.png)

- 准备一个可运行的.netcore webapi项目的发布后文件。
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519170748261-503752043.png)
这是我的dockerfile:
```
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1 AS runtime
EXPOSE 80
WORKDIR /app
COPY out/ ./
ENTRYPOINT ["dotnet", "NetCore3.1-WebApi-Demo.dll"]
```

- 准备一个远程连接工具，我使用的是putty
- 文件上传工具，我使用的是WinSCP，也可以搭建ftp等...

# 二、开始部署
- 使用putty登录到centos，创建一个测试目录webapitest：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519172045409-2033320704.png)
- 使用WinSCP登录到centos上传文件到服务器webapitest目录下：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519172629970-629840485.png)
- 上传完成后回到putty执行docker build：
`docker build -t webapitest .`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519190958949-1839166523.png)
看到Successfully说明build成功，然后执行docker images命令，查看镜像列表：
`docker images`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519191144198-192994314.png)
- 执行docker run命令，运行容器：
`docker run -d -p 5001:80 --name webapitestapp webapitest`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519191348860-988737596.png)
执行docker ps命令查看当前运行的容器：
`docker ps`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519191446594-2013890832.png)
- 使用浏览器访问测试：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200519191540204-1128356779.png)
测试访问正常，至此就完成了.netcore webapi在centos docker环境下的部署。