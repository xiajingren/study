[TOC]

# 前言

上一篇介绍了ABP的启动模板以及AbpHelper工具的基本使用，这一篇将进入项目实战部分。因为目前ABP的官方模板只支持MVC和Angular，MVC的话咱.NET开发人员来写还可以，专业前端估计很少会用这个。。。Angular我本人不熟，所以选择vue来做UI。



# 开始

我使用[vue-element-admin](https://github.com/PanJiaChen/vue-element-admin)来作为模板，这个项目貌似很多人用，选择他的[i18n](https://github.com/PanJiaChen/vue-element-admin/tree/i18n)分支，因为我需要国际化功能。在开始编码前，需要先分析几个重要问题：

- 用户登录/token
- 用户权限控制
- 应用程序本地化/语言切换

好在ABP模板提供了Angular版本，我们可以参考Angular版本来做。

## 登录

因为ABP的授权模块是使用IdentityServer4，所以IdentityServer4的一些默认端点在ABP里也是同样有效的，可以参考下[IdentityServer4官网](https://identityserver4.readthedocs.io/)。运行ABP模板项目，看一下IdentityServer4发现文档，uri是：`/.well-known/openid-configuration`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810163749436-980878802.png)

可以看到token端点是`/connect/token`，这是IdentityServer4默认的，通过这个端点就可以登录用户获取token。后面请求接口时，把token放在HTTP Header的authorization字段即可。

## 权限

进入ABP的`/swagger`界面：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810165447203-1444129284.png)

ABP内置了一个`/api/abp/application-configuration`接口，它用于返回本地化文本，权限和一些系统设置信息。看一下数据格式：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810170048204-401146819.png)

在**auth.policies**字段中包含了系统的所有权限，**auth.grantedPolicies**字段则包含了当前用户所拥有的权限，因为我现在没登录所以是空的。通过这两个字段就可以和vue-element-admin的菜单权限对应起来，实现权限控制。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810171247311-1143536186.png)

**currentUser**字段表示当前用户信息，没登录时就是空的，**isAuthenticated**为false，这个字段也可以作为用户是否登录（token是否有效）的判断依据。

## 本地化

本地化对于大部分的小型系统可能都用不上，不过ABP作为一个优秀且全面的框架，必然会支持本地化功能。同样的，本地化信息也可以通过`/api/abp/application-configuration`接口来获取：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810171826125-307001118.png)

**localization.languages**字段表示系统所支持的语言类型，前端的语言切换选项就可以使用这个字段。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810172528544-1773818737.png)

**localization.values**字段就是本地化的文本信息了，你在后端配置的本地化文本都可以从这里获取到，通过这个字段结合vue-element-admin的国际化功能，就可以让你的系统支持多语言。vue-element-admin的国际化方案是通过 [vue-i18n](https://github.com/kazupon/vue-i18n)来实现，你也可以直接在前端部分来做国际化，如果你只有一个前端应用的话，但是在后端做复用性更好一些。

语言切换时只需要把对应的语言名称放到HTTP Header的accept-language字段就行。

## 创建项目

在开始编码前，先创建好前后端的模板项目。

### ABP

这里直接用Abp CLI命令来创建解决方案吧：

```powershell
abp new "Xhznl.HelloAbp" 
-t app 
-u none --separate-identity-server 
-m none 
-d ef -cs "Server=localhost;User Id=sa;Password=Password@2020;Database=HelloAbp;MultipleActiveResultSets=true"
```

创建一个名为"Xhznl.HelloAbp"的解决方案，使用app作为模板，不需要UI，并且将Identity Server应用程序与API host应用程序分开，使用Entity Framework Core作为数据库提供程序，并指定连接字符串。创建完成后会得到一个aspnet-core文件夹。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811094435880-828476414.png)

项目结构如下，因为指定了`--separate-identity-server`参数，所以多了个IdentityServer项目，如果不指定的话它是集成在HttpApi.Host中的。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811105932795-339687787.png)

通常小型系统没必要把Identity Server应用程序与API host应用程序分开，会增加运维成本，这里只是为了演示分布式部署的情况，为后面的微服务做准备。

ABP还支持为每个模块单独配置数据库（如果你不需要分库，可以忽略以下内容），修改DbMigrator、IdentityServer项目的appsettings.json配置文件：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811160628452-1933296949.png)

在ConnectionStrings中添加AbpIdentityServer配置，为Identity Server配置独立的数据库连接字符串，不配置的话默认使用Default配置。AbpIdentityServer这个key是来自ABP的IdentityServer模块中的一个常量，具体请参考源码。

在开发环境光定义连接字符串还不够，因为HelloAbpIdsDB数据库还不存在，需要使用EF Core Code Frist迁移系统创建和维护这个数据库。新建一个项目：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812171158565-132408323.png)

步骤比较多，具体流程请参考官网：[数据库迁移](https://docs.abp.io/zh-Hans/abp/latest/Entity-Framework-Core-Migrations#使用多个数据库)，这里就不重复介绍了，你也可以选择不分库。

完成以上步骤，最终会生成2个数据库，并且包含了一些默认的种子数据。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811163403948-1392973109.png)

然后验证一下HttpApi.Host和IdentityServer项目是否可以正常运行，前提是你电脑需要有sqlserver，redis。

HttpApi.Host：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811172838923-1221899843.png)

IdentityServer：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200811172903263-416062701.png)

### vue-element-admin

vue-element-admin的基本使用就不介绍了，相信很多人见过这个，不了解的可以自己去搜索学习一下。

去GitHub下载[i18n](https://github.com/PanJiaChen/vue-element-admin/tree/i18n)分支的代码，或者直接用git clone命令。

在项目根目录下执行指令：

安装依赖：`npm install`

启动项目：`npm run dev`

启动正常的话可以看到这个界面：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200810190011711-1997461720.png)



# 最后

本篇先做准备工作，下一篇将从登录功能开始编码实现。。。代码已上传至GitHub：https://github.com/xiajingren/HelloAbp 欢迎star。


