[TOC]

# 前言

上一篇介绍了ABP的一些基础知识，本篇继续介绍ABP的启动模板。使用ABP CLI命令就可以得到这个启动模板，其中包含了一些基础功能模块，你可以基于这个模板来快速开发。



# 开始

首先ABP CLI的安装以及基本指令这些就不说了，官网上写的很清楚。目前ABP的前端部分只支持ASP.NET Core MVC / Razor Pages和Angular，移动端支持React Native。

初学者建议跟着官网https://docs.abp.io/zh-Hans/abp/latest/Tutorials/Part-1?UI=MVC这个指引做一遍，体验一下ABP开发的基本流程，虽然ABP开发流程几乎都标准化了，照着官网的流程编写代码就能完成一个功能的开发，但是这个过程有些繁琐，容易出错。这里推荐一个开源项目：https://github.com/EasyAbp/AbpHelper.GUI，这是一个ABP帮助工具，你只需要创建一个实体，剩下的代码它都可以帮你生成。这个项目是https://github.com/EasyAbp下的一个子项目，EasyAbp是国内ABP爱好者创建的，里面还有很多开箱即用的模块，可以关注一下。。。

## AbpHelper

使用AbpHelper来完成官网的例子非常容易，首先创建项目解决方案：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808172208668-117306134.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808172259139-1144630104.png)

AbpHelper提供了图形化配置，自动帮我们执行ABP CLI指令：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808173003735-727456081.png)

执行完成后，打开解决方案，先启动Acme.BookStore.DbMigrator项目来初始化数据库：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808173402021-1502894791.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808173329310-1112891760.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808173506095-726154508.png)

然后就可以启动Acme.BookStore.Web项目，这是APB启动模板的默认界面：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808174015503-514476808.png)

接下来，在Acme.BookStore.Domain项目中创建Book实体，我直接从官网上复制代码。

```csharp
public class Book : AuditedAggregateRoot<Guid>
{
    public string Name { get; set; }

    public BookType Type { get; set; }

    public DateTime PublishDate { get; set; }

    public float Price { get; set; }

    protected Book()
    {
    }
    public Book(Guid id, string name, BookType type, DateTime publishDate, float price)
        : base(id)
    {
        Name = name;
        Type = type;
        PublishDate = publishDate;
        Price = price;
    }
}
```

在Acme.BookStore.Domain.Shared项目中添加枚举类BookType：

```
public enum BookType
{
    Undefined,
    Adventure,
    Biography,
    Dystopia,
    Fantastic,
    Horror,
    Science,
    ScienceFiction,
    Poetry
}
```

第一次使用需要安装一下AbpHelper CLI：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808174709856-692014386.png)

选择Generate CRUD，填入实体名称和解决方案路径，然后Execute即可：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200808223929839-1900603445.png)

生成代码时可能会报这个错（如果没装ef tools）：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809122533017-1199719100.png)

这时安装一下ef tools就好了，`dotnet tool install -g dotnet-ef`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809122703623-1534963738.png)

代码生成完后，运行Acme.BookStore.Web项目：

使用默认用户 admin/1q2w3E* 登录系统，给admin角色分配BookStore相关权限：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809142215488-1866268604.png)

然后就可以看到book菜单了，包括基本的增删改查界面：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809142402665-1762942163.png)

至此就完成了一个基本功能的开发，AbpHelper确实很方便，他还有CLI版本，直接命令行操作。

## 模块安装

ABP的模块化可以实现插件式的开发，你可以预先构建一些通用的模块，比如日志模块，用户模块等等，当你以后需要时就可以直接安装到项目中。有一些由ABP社区开发和维护的开源免费的应用程序模块，我们可以直接使用；比如我要使用官方的Blogging模块，Blogging是用于创建精美的博客。

同样使用AbpHelper来安装：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809173458658-1627766935.png)

安装过程出了点小问题，提示找不到DbContext。。。不过没关系，自己执行一下迁移命令就行。。。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809222058906-744846658.png)

Acme.BookStore.Web项目设为启动项，默认项目为Acme.BookStore.EntityFrameworkCore.DbMigrations，然后执行：

`Add-Migration AddedBlogging`

`Update-DataBase`

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809222800652-1157485564.png)

接下来再次运行Acme.BookStore.Web项目，为admin角色配置博客相关的权限：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809192431041-540693524.png)

然后就就可以看到博客的相关功能：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809223056171-1206249551.png)

Swagger：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200809224947937-578664104.png)

当然，这些模块不一定完全符合你的要求，你可能需要稍作修改，ABP也允许你扩展实体，重写服务包括重写用户界面，你可以很方便的修改。这些后面再介绍，包括如何去开发这种模块。。。



# 最后

EasyAbp上也有很多开源模块，地址是：https://github.com/EasyAbp/EasyAbpGuide，目前这些模块的UI部分都只支持MVC/Razor Pages，不支持Angular之类的。。。当然模块不一定非要UI，一些Framework级别的模块就不需要UI。基础部分就写到这里，主要还是需要认真看下官网，然后自己动手练习一下。下一篇将进入vue+ABP实战部分。

