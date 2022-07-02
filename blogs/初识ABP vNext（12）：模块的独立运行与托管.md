[TOC]

# 前言

很久没更新这个系列。。。之前的章节中讲到ABP的模块是可以独立运行的，但是没有介绍具体怎么操作，本篇就来讨论一下模块如何独立运行，以及一些托管方式。本人也是处于摸索阶段，如有不对欢迎指出。




# 开始
## 模块运行

首先需要生成模块的数据库，修改`HttpApi.Host`和`IdentityServer`项目的`appsettings.json`数据库连接字符串配置。

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029112031658-1016178286.png)

\modules\file-management\host\Xhznl.FileManagement.HttpApi.Host\appsettings.json：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029113717428-546963758.png)

\modules\file-management\host\Xhznl.FileManagement.IdentityServer\appsettings.json：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029113739190-1920028055.png)

这样会生成2个数据库，如果你只需要一个数据库的话，就把`FileManagement`的那行配置去掉就好了。

打开程序包管理器控制台，默认项目选择`IdentityServer`，执行`update-database`

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029114515532-434053586.png)

执行完成会生成Main数据库，其中是一些ABP的基础表。

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029114704548-1409262071.png)

继续将默认项目设置为`HttpApi.Host`执行`add-migration Initial`  `update-database`

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029115522213-2002189719.png)

执行完成会生成Module数据库，其中是你模块的相关表。

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029121619379-1941514745.png)

此时这两个项目就可以正常运行了。

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029122409085-256684635.png)

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029122421448-1862298157.png)

项目中可能有多个模块相互协作，如果将各个模块独立运行的话，不可能每个模块都创建一个Main数据库，所以部分ABP的通用模块的数据库表就用同一个就好了。

\modules\file-management\host\Xhznl.FileManagement.HttpApi.Host\appsettings.json：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029130757297-1385112816.png)

\modules\file-management\host\Xhznl.FileManagement.IdentityServer\appsettings.json：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029130820846-870785924.png)

## 动态 C# API 客户端

当有多个独立部署的模块时，可能需要做一些网关之类的来统一入口，模块之间的相互调用也比较麻烦，本篇暂不讨论。下面介绍一下如何使用ABP的动态C# API客户端来调用远程模块。

> ABP可以自动创建C# API 客户端代理来调用远程HTTP服务(REST APIS).通过这种方式,你不需要通过 `HttpClient` 或者其他低级的HTTP功能调用远程服务并获取数据.

前面的章节中，在主项目中将模块的`Application`层和`Domain`层的大部分项目都引用了一遍，那种方式是单体部署的情况，模块和主项目托管在同一个进程里。

下面使用C# API客户端来代理远程模块。

首先删除项目中模块的引用和`DependsOn`

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029190838112-2059454260.png)

然后在你需要调用模块的项目中，添加模块的`HttpApi.Client`项目的依赖即可。比如我这里的`Xhznl.HelloAbp.HttpApi.Host`项目：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029191326754-462511767.png)

然后`DependsOn`：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029191536899-408332689.png)

然后在`appsettings.json`中添加远程服务的地址配置：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029191720509-1722374039.png)

其中的`FileManagement`这个名称是来自模块的`HttpApi.Client`项目中的定义：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029191901621-2117657607.png)

接下来就可以像使用本地方法一样去使用远程服务了，因为`HttpApi.Client`是依赖于`Application.Contracts`项目的，所以你模块的所有服务接口都可以在这里使用，直接注入即可（前提是你的服务需要实现`IRemoteService`），ABP会自动帮你完成Http的远程调用。随便找个地方测试一下：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029192852827-709797232.png)

接下来是模块项目，最好配合ABP的自动API控制器一起使用，如果你是自定义路由的话，可能会出现一些`Could not found remote action`的奇怪错误。

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029192224672-613511655.png)

Auth服务地址也注意一下：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029194403458-1084837062.png)

下面给两个项目打上断点，测试一下流程是否正确：

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029194635682-187731539.png)

![](https://img2020.cnblogs.com/blog/610959/202010/610959-20201029194720013-787059653.png)

可以看到，请求已经正常流转到模块项目中。

上面有些乱，总结一下重点：

1. 添加`HttpApi.Client`引用
2. 添加`RemoteServices`地址配置
3. 注入服务接口进行使用

如果想托管模块的所有API，那么只需要再添加模块的`HttpApi`依赖即可。托管方式非常灵活，具体可以参考：[模块化架构最佳实践 & 约定](https://docs.abp.io/zh-Hans/abp/latest/Best-Practices/Module-Architecture)



# 最后

本篇就到这里。。。。。。

