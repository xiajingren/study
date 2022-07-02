[TOC]

# 前言

> NuGet是用于微软.NET（包括 .NET Core）开发平台的软件包管理器。NuGet能够令你在项目中添加、移除和更新引用的工作变得更加快捷方便。

通常使用NuGet都是官方的服务，但你有没有想过搭建自己的NuGet呢？在私有的NuGet上托管一些自己的类库，公司内部的类库等。。。搭建私有NuGet的方法有很多，比如NuGet.Server、ProGet、MyGet等等。本文使用的是BaGet，搭建过程也非常简单，下面进入正题。



# 开始

## 搭建BaGet

> BaGet是一个构建于ASP.NET Core 基础上的 NuGet V3 服务器的开源实现。

github地址：https://github.com/loic-sharma/BaGet

下载release包，我下载的是最新预览版，你也可以选择其他版本：

https://github.com/loic-sharma/BaGet/releases/download/v0.3.0-preview4/BaGet.zip

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803145835306-1464552057.png)

你可以按需要修改一下端口配置，默认是5000：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803150039208-1901086530.png)

在解压目录下打开命令行，执行：`dotnet BaGet.dll`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803150339570-1549904699.png)

浏览器访问：`http://localhost:8020/`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803150445407-1803809948.png)



这样，NuGet服务就搭建完成了，是不是很简单？

## 上传程序包

随便创建一个类库项目用于测试：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803151119353-1878230504.png)

右键项目，选择打包：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803151310795-1560461889.png)

打包完成会得到一个nupkg文件：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803151449410-2061280830.png)

当然，你也可以选择Release模式：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803163209042-827415192.png)

看一下Upload命令：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803151627897-403112516.png)

在上面打包目录下打开命令行执行：`dotnet nuget push -s http://localhost:8020/v3/index.json MyTestLibrary.1.0.0.nupkg`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803151838316-1387320016.png)

再次查看Packages：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803152049239-1394894748.png)

## 在vs中使用

在vs2019中打开：工具-选项-NuGet包管理器-程序包源。添加一个源，输入名称，源：http://localhost:8020/v3/index.json

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803153959710-285473415.png)

接下来就可以正常使用了：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803154615906-823903425.png)

## 其他

程序包的作者，说明，版本号等信息可以在这里修改：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803155418572-1384622153.png)

依赖项也完全不用担心：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200803160040430-1047110919.png)



# 最后

因为BaGet是基于ASP.NET Core开发，所以天生跨平台，你可以在windows，mac，linux或者docker中轻松部署。另外，BaGet也没有复杂的环境依赖，数据库默认Sqlite，很轻量，部署起来非常容易。

当然，本文一开始也提到，搭建私有NuGet的方式有很多，如有需要可以参考微软官方说明：https://docs.microsoft.com/zh-cn/nuget/hosting-packages/overview

