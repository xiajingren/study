# 前言
这两天看了一下ABP，做个简单的学习记录。记录主要有以下内容：
1. 从官网创建并下载项目(.net core 3.x + vue)
2. 项目在本地成功运行
3. 新增实体并映射到数据库
4. 完成对新增实体的基本增删改查

ABP官网：https://aspnetboilerplate.com/
Github：https://github.com/aspnetboilerplate

# 创建项目
进入官网
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627100457868-170119983.png)

Get started，选择前后端技术栈，我这里就选.net core 3.x和vue。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627100647685-2090498988.png)

填写自己的项目名称，邮箱，然后点create my project就可以下载项目了。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627100918563-2094721135.png)

解压文件
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627102510273-1984153044.png)

# 运行项目
## 后端项目
首先运行后端项目，打开/aspnet-core/MyProject.sln
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627102114156-216837578.png)

改一下MyProject.Web.Host项目下appsettings.json的数据库连接字符串，如果本地安装了mssql，用windows身份认证，不改也行
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627103950359-1210972586.png)

数据库默认是使用mssql的，当然也可以改其他数据库。

将MyProject.Web.Host项目设置为启动项，打开程序包管理器控制台，默认项目选择DbContext所在的项目，也就是MyProject.EntityFrameworkCore。执行`update-database`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627104427236-1141298040.png)

数据库已成功创建：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627104650285-1366503271.png)

Ctrl+F5，不出意外，浏览器就会看到这个界面：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627104847414-338522314.png)

## 前端项目
后端项目成功运行了，下面运行一下前端项目，先要确保本机有nodejs环境并安装了vue cli，这个就不介绍了。

/vue目录下打开cmd执行：`npm install`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627105827060-1630520275.png)

install完成后执行：`npm run serve`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627110024817-53125415.png)

打开浏览器访问http://localhost:8080/，不出意外的话，会看到这个界面：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627110339033-46455551.png)

使用默认用户 admin/123qwe 登录系统：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627110508174-741570768.png)

至此，前后端项目都已成功运行。
那么基于abp的二次开发该从何下手呢，最简单的，比如要增加一个数据表，并且完成最基本CRUD该怎么做？

# 新增实体
实体类需要放在MyProject.Core项目下，我新建一个MyTest文件夹，并新增一个Simple类，随意给2个属性。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627113800717-635021028.png)
我这里继承了abp的Entity<int>类，Entity类有主键ID属性，这个泛型int是指主键的类型，不写默认就是int。abp还有一个比较复杂的FullAuditedEntity类型，继承FullAuditedEntity的话就有创建时间，修改时间，创建人，修改人，软删除等字段。这个看实际情况。
```
public class Simple : Entity<int>
{
    public string Name { get; set; }

    public string Details { get; set; }
}
```
修改MyProject.EntityFrameworkCore项目的/EntityFrameworkCore/MyProjectDbContext：
```
public class MyProjectDbContext : AbpZeroDbContext<Tenant, Role, User, MyProjectDbContext>
{
    /* Define a DbSet for each entity of the application */

    public DbSet<Simple> Simples { get; set; }

    public MyProjectDbContext(DbContextOptions<MyProjectDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Simple>(p =>
        {
            p.ToTable("Simples", "test");
            p.Property(x => x.Name).IsRequired(true).HasMaxLength(20);
            p.Property(x => x.Details).HasMaxLength(100);
        });
    }
}
```
然后就可以迁移数据库了，程序包管理器控制台执行：`add-migration mytest1`，`update-database`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627115048131-2123033570.png)
刷新数据库，Simples表已生成：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627115950665-1424626700.png)

# 实体的增删改查
进入MyProject.Application项目，新建一个MyTest文件夹
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627124056350-400123291.png)

## Dto
CreateSimpleDto，新增Simple数据的传输对象，比如ID，创建时间，创建人等字段，就可以省略
```
public class CreateSimpleDto
{
    public string Name { get; set; }

    public string Details { get; set; }
}
```
PagedSimpleResultRequestDto，分页查询对象
```
public class PagedSimpleResultRequestDto : PagedResultRequestDto
{
    /// <summary>
    /// 查询关键字
    /// </summary>
    public string Keyword { get; set; }
}
```
SimpleDto，这里跟CreateSimpleDto的区别就是继承了EntityDto，多了个ID属性
```
public class SimpleDto : EntityDto<int>
{
    public string Name { get; set; }

    public string Details { get; set; }
}
```
SimpleProfile，用来定义AutoMapper的映射关系清单
```
public class SimpleProfile : Profile
{
    public SimpleProfile()
    {
        CreateMap<Simple, SimpleDto>();
        CreateMap<SimpleDto, Simple>();
        CreateMap<CreateSimpleDto, Simple>();
    }
}
```
## Service
注意，类名参考abp的规范去命名。

ISimpleAppService，Simple服务接口。我这里继承IAsyncCrudAppService，这个接口中包含了增删改查的基本定义，非常方便。如果不需要的话，也可以继承IApplicationService自己定义
```
public interface ISimpleAppService : IAsyncCrudAppService<SimpleDto, int, PagedSimpleResultRequestDto, CreateSimpleDto, SimpleDto>
{

}
```
SimpleAppService，Simple服务，继承包含了增删改查的AsyncCrudAppService类，如果有需要的话可以override这些增删改查方法。也可以继承MyProjectAppServiceBase，自己定义。
```
public class SimpleAppService : AsyncCrudAppService<Simple, SimpleDto, int, PagedSimpleResultRequestDto, CreateSimpleDto, SimpleDto>, ISimpleAppService
{
    public SimpleAppService(IRepository<Simple, int> repository) : base(repository)
    {

    }

    /// <summary>
    /// 条件过滤
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    protected override IQueryable<Simple> CreateFilteredQuery(PagedSimpleResultRequestDto input)
    {
        return Repository.GetAll()
            .WhereIf(!input.Keyword.IsNullOrWhiteSpace(), a => a.Name.Contains(input.Keyword));
    }
}
```
## 接口测试
重新运行项目，不出意外的话，Swagger中就会多出Simple相关的接口。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627130744917-1655287996.png)

- Create

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131029796-1315680981.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131120431-831899305.png)

- Get

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131349678-286797860.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131413832-513887384.png)

- GetAll

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131547775-48419930.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131608590-91559163.png)

- Update

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131706912-822599511.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131729470-463138682.png)

- Delete

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131801785-1425833934.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200627131820860-673778571.png)

# 总结
ABP是一个优秀的框架，基于ABP的二次开发肯定会非常高效，但前提是需要熟练掌握ABP，弄清楚他的设计理念以及他的一些实现原理。

以后有时间的话再深入学习一下。文中如果有不妥之处欢迎指正。