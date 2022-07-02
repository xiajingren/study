[TOC]

# 前言

在前两节中介绍了ABP模块开发的基本步骤，试着实现了一个简单的文件管理模块；功能很简单，就是基于本地文件系统来完成文件的读写操作，数据也并没有保存到数据库，所以之前只简单使用了应用服务，并没有用到领域层。而在DDD中领域层是非常重要的一层，其中包含了实体，聚合根，领域服务，仓储等等，复杂的业务逻辑也应该在领域层来实现。本篇来完善一下文件管理模块，将文件记录保存到数据库，并使用ABP BLOB系统来完成文件的存储。


# 开始
## 聚合根

首先从实体模型开始，建立File实体。按照DDD的思路，这里的File应该是一个**聚合根**。

\modules\file-management\src\Xhznl.FileManagement.Domain\Files\File.cs：

```csharp
public class File : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public virtual Guid? TenantId { get; protected set; }

    [NotNull]
    public virtual string FileName { get; protected set; }

    [NotNull]
    public virtual string BlobName { get; protected set; }

    public virtual long ByteSize { get; protected set; }

    protected File() { }

    public File(Guid id, Guid? tenantId, [NotNull] string fileName, [NotNull] string blobName, long byteSize) : base(id)
    {
        TenantId = tenantId;
        FileName = Check.NotNullOrWhiteSpace(fileName, nameof(fileName));
        BlobName = Check.NotNullOrWhiteSpace(blobName, nameof(blobName));
        ByteSize = byteSize;
    }
}
```

在DbContext中**添加DbSet**

\modules\file-management\src\Xhznl.FileManagement.EntityFrameworkCore\EntityFrameworkCore\IFileManagementDbContext.cs：

```csharp
public interface IFileManagementDbContext : IEfCoreDbContext
{
    DbSet<File> Files { get; }
}
```

\modules\file-management\src\Xhznl.FileManagement.EntityFrameworkCore\EntityFrameworkCore\FileManagementDbContext.cs：

```csharp
public class FileManagementDbContext : AbpDbContext<FileManagementDbContext>, IFileManagementDbContext
{
    public DbSet<File> Files { get; set; }

    ......
}

```

**配置实体**

\modules\file-management\src\Xhznl.FileManagement.EntityFrameworkCore\EntityFrameworkCore\FileManagementDbContextModelCreatingExtensions.cs：

```csharp
public static void ConfigureFileManagement(
    this ModelBuilder builder,
    Action<FileManagementModelBuilderConfigurationOptions> optionsAction = null)
{
    ......

    builder.Entity<File>(b =>
    {
        //Configure table & schema name
        b.ToTable(options.TablePrefix + "Files", options.Schema);

        b.ConfigureByConvention();

        //Properties
        b.Property(q => q.FileName).IsRequired().HasMaxLength(FileConsts.MaxFileNameLength);
        b.Property(q => q.BlobName).IsRequired().HasMaxLength(FileConsts.MaxBlobNameLength);
        b.Property(q => q.ByteSize).IsRequired();
    });
}
```

## 仓储

ABP为每个聚合根或实体提供了 **默认的通用(泛型)仓储** ，其中包含了标准的CRUD操作，注入`IRepository<TEntity, TKey>`即可使用。通常来说默认仓储就够用了，有特殊需求时也可以自定义仓储。

定义**仓储接口**

\modules\file-management\src\Xhznl.FileManagement.Domain\Files\IFileRepository.cs：

```csharp
public interface IFileRepository : IRepository<File, Guid>
{
    Task<File> FindByBlobNameAsync(string blobName);
}
```

**仓储实现**

\modules\file-management\src\Xhznl.FileManagement.EntityFrameworkCore\Files\EfCoreFileRepository.cs：

```csharp
public class EfCoreFileRepository : EfCoreRepository<IFileManagementDbContext, File, Guid>, IFileRepository
{
    public EfCoreFileRepository(IDbContextProvider<IFileManagementDbContext> dbContextProvider) : base(dbContextProvider)
    {
    }

    public async Task<File> FindByBlobNameAsync(string blobName)
    {
        Check.NotNullOrWhiteSpace(blobName, nameof(blobName));

        return await DbSet.FirstOrDefaultAsync(p => p.BlobName == blobName);
    }
}
```

**注册仓储**

\modules\file-management\src\Xhznl.FileManagement.EntityFrameworkCore\EntityFrameworkCore\FileManagementEntityFrameworkCoreModule.cs：

```csharp
public class FileManagementEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<FileManagementDbContext>(options =>
        {
            options.AddRepository<File, EfCoreFileRepository>();
        });
    }
}
```

## 领域服务

定义**领域服务接口**

\modules\file-management\src\Xhznl.FileManagement.Domain\Files\IFileManager.cs：

```csharp
public interface IFileManager : IDomainService
{
    Task<File> FindByBlobNameAsync(string blobName);

    Task<File> CreateAsync(string fileName, byte[] bytes);

    Task<byte[]> GetBlobAsync(string blobName);
}
```

在实现领域服务之前，先来安装一下ABP Blob系统核心包，因为我要使用blob来存储文件，`Volo.Abp.BlobStoring`包是必不可少的。

### BLOB存储

BLOB(binary large object)：大型二进制对象；关于BLOB可以参考 [BLOB 存储](https://docs.abp.io/zh-Hans/abp/latest/Blob-Storing) ，这里不多介绍。

安装`Volo.Abp.BlobStoring`，在Domain项目目录下执行：`abp add-package Volo.Abp.BlobStoring`

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923215317430-1613070966.png)

`Volo.Abp.BlobStoring`是BLOB的核心包，它仅包含BLOB的一些基本抽象，想要BLOB系统正常工作，还需要为它配置一个提供程序；这个提供程序暂时不管，将来由模块的具体使用者去提供。这样的好处是模块不依赖特定存储提供程序，使用者可以随意的指定存储到阿里云，Azure，或者文件系统等等。。。



**领域服务实现**

\modules\file-management\src\Xhznl.FileManagement.Domain\Files\FileManager.cs：

```csharp
public class FileManager : DomainService, IFileManager
{
    protected IFileRepository FileRepository { get; }
    protected IBlobContainer BlobContainer { get; }

    public FileManager(IFileRepository fileRepository, IBlobContainer blobContainer)
    {
        FileRepository = fileRepository;
        BlobContainer = blobContainer;
    }

    public virtual async Task<File> FindByBlobNameAsync(string blobName)
    {
        Check.NotNullOrWhiteSpace(blobName, nameof(blobName));

        return await FileRepository.FindByBlobNameAsync(blobName);
    }

    public virtual async Task<File> CreateAsync(string fileName, byte[] bytes)
    {
        Check.NotNullOrWhiteSpace(fileName, nameof(fileName));

        var blobName = Guid.NewGuid().ToString("N");

        var file = await FileRepository.InsertAsync(new File(GuidGenerator.Create(), CurrentTenant.Id, fileName, blobName, bytes.Length));

        await BlobContainer.SaveAsync(blobName, bytes);

        return file;
    }

    public virtual async Task<byte[]> GetBlobAsync(string blobName)
    {
        Check.NotNullOrWhiteSpace(blobName, nameof(blobName));

        return await BlobContainer.GetAllBytesAsync(blobName);
    }
}
```

## 应用服务

接下来修改一下应用服务，应用服务通常没有太多业务逻辑，其调用领域服务来完成业务。

**应用服务接口**

\modules\file-management\src\Xhznl.FileManagement.Application.Contracts\Files\IFileAppService.cs：

```csharp
public interface IFileAppService : IApplicationService
{
    Task<FileDto> FindByBlobNameAsync(string blobName);

    Task<string> CreateAsync(FileDto input);
}
```

**应用服务实现**

\modules\file-management\src\Xhznl.FileManagement.Application\Files\FileAppService.cs：

```csharp
public class FileAppService : FileManagementAppService, IFileAppService
{
    protected IFileManager FileManager { get; }

    public FileAppService(IFileManager fileManager)
    {
        FileManager = fileManager;
    }

    public virtual async Task<FileDto> FindByBlobNameAsync(string blobName)
    {
        Check.NotNullOrWhiteSpace(blobName, nameof(blobName));

        var file = await FileManager.FindByBlobNameAsync(blobName);
        var bytes = await FileManager.GetBlobAsync(blobName);

        return new FileDto
        {
            Bytes = bytes,
            FileName = file.FileName
        };
    }

    [Authorize]
    public virtual async Task<string> CreateAsync(FileDto input)
    {
        await CheckFile(input);

        var file = await FileManager.CreateAsync(input.FileName, input.Bytes);

        return file.BlobName;
    }

    protected virtual async Task CheckFile(FileDto input)
    {
        if (input.Bytes.IsNullOrEmpty())
        {
            throw new AbpValidationException("Bytes can not be null or empty!",
                new List<ValidationResult>
                {
                    new ValidationResult("Bytes can not be null or empty!", new[] {"Bytes"})
                });
        }

        var allowedMaxFileSize = await SettingProvider.GetAsync<int>(FileManagementSettings.AllowedMaxFileSize);//kb
        var allowedUploadFormats = (await SettingProvider.GetOrNullAsync(FileManagementSettings.AllowedUploadFormats))
            ?.Split(",", StringSplitOptions.RemoveEmptyEntries);

        if (input.Bytes.Length > allowedMaxFileSize * 1024)
        {
            throw new UserFriendlyException(L["FileManagement.ExceedsTheMaximumSize", allowedMaxFileSize]);
        }

        if (allowedUploadFormats == null || !allowedUploadFormats.Contains(Path.GetExtension(input.FileName)))
        {
            throw new UserFriendlyException(L["FileManagement.NotValidFormat"]);
        }
    }
}
```

**API控制器**

最后记得将服务接口暴露出去，我这里是自己编写Controller，你也可以使用ABP的自动API控制器来完成，请参考 [ 自动API控制器](https://docs.abp.io/zh-Hans/abp/latest/API/Auto-API-Controllers)

\modules\file-management\src\Xhznl.FileManagement.HttpApi\Files\FileController.cs：

```csharp
[RemoteService]
[Route("api/file-management/files")]
public class FileController : FileManagementController
{
    protected IFileAppService FileAppService { get; }

    public FileController(IFileAppService fileAppService)
    {
        FileAppService = fileAppService;
    }

    [HttpGet]
    [Route("{blobName}")]
    public virtual async Task<FileResult> GetAsync(string blobName)
    {
        var fileDto = await FileAppService.FindByBlobNameAsync(blobName);
        return File(fileDto.Bytes, MimeTypes.GetByExtension(Path.GetExtension(fileDto.FileName)));
    }

    [HttpPost]
    [Route("upload")]
    [Authorize]
    public virtual async Task<JsonResult> CreateAsync(IFormFile file)
    {
        if (file == null)
        {
            throw new UserFriendlyException("No file found!");
        }

        var bytes = await file.GetAllBytesAsync();
        var result = await FileAppService.CreateAsync(new FileDto()
        {
            Bytes = bytes,
            FileName = file.FileName
        });
        return Json(result);
    }
}
```

## 单元测试

针对以上内容做一个简单的测试，首先为Blob系统配置一个提供程序。

我这里使用最简单的文件系统来储存，所以需要安装`Volo.Abp.BlobStoring.FileSystem`。在Application.Tests项目目录下执行：`abp add-package Volo.Abp.BlobStoring.FileSystem`

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923223117152-2113769594.png)

**配置默认容器**

\modules\file-management\test\Xhznl.FileManagement.Application.Tests\FileManagementApplicationTestModule.cs：

```csharp
[DependsOn(
    typeof(FileManagementApplicationModule),
    typeof(FileManagementDomainTestModule),
    typeof(AbpBlobStoringFileSystemModule)
    )]
public class FileManagementApplicationTestModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        Configure<AbpBlobStoringOptions>(options =>
        {
            options.Containers.ConfigureDefault(container =>
            {
                container.UseFileSystem(fileSystem =>
                {
                    fileSystem.BasePath = "D:\\my-files";
                });
            });
        });

        base.ConfigureServices(context);
    }
}
```

**测试用例**

\modules\file-management\test\Xhznl.FileManagement.Application.Tests\Files\FileAppService_Tests.cs：

```csharp
public class FileAppService_Tests : FileManagementApplicationTestBase
{
    private readonly IFileAppService _fileAppService;

    public FileAppService_Tests()
    {
        _fileAppService = GetRequiredService<IFileAppService>();
    }

    [Fact]
    public async Task Create_FindByBlobName_Test()
    {
        var blobName = await _fileAppService.CreateAsync(new FileDto()
        {
            FileName = "微信图片_20200813165555.jpg",
            Bytes = await System.IO.File.ReadAllBytesAsync(@"D:\WorkSpace\WorkFiles\杂项\图片\微信图片_20200813165555.jpg")
        });
        blobName.ShouldNotBeEmpty();

        var fileDto = await _fileAppService.FindByBlobNameAsync(blobName);
        fileDto.ShouldNotBeNull();
        fileDto.FileName.ShouldBe("微信图片_20200813165555.jpg");
    }
}
```

**运行测试**

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200924130852505-278784871.png)

测试通过，blob也已经存入D:\\my-files：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200924131054161-1906173417.png)

## 模块引用

下面回到主项目，前面的章节中已经介绍过，模块的引用依赖都已经添加完成，下面就直接从数据库迁移开始。

\src\Xhznl.HelloAbp.EntityFrameworkCore.DbMigrations\EntityFrameworkCore\HelloAbpMigrationsDbContext.cs：

```csharp
public class HelloAbpMigrationsDbContext : AbpDbContext<HelloAbpMigrationsDbContext>
{
    public HelloAbpMigrationsDbContext(DbContextOptions<HelloAbpMigrationsDbContext> options)
        : base(options)
    {

    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        ......
            
        builder.ConfigureFileManagement();
        
        ......
    }
}
```

打开程序包管理器控制台，执行以下命令：

`Add-Migration "Added_FileManagement"`

`Update-Database`

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923231607988-1470111310.png)

此时数据库已经生成了File表：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923231839341-235042846.png)

还有记得在HttpApi.Host项目配置你想要的blob提供程序。

最后结合前端测试一下吧：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923232348431-2045452685.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200923232306418-1872529958.png)

# 最后
以上就是本人所理解的abp模块开发一个相对完整的流程，还有些概念后面再做补充。因为这个例子比较简单，文中有些环节是不必要的，需要结合实际情况去取舍。代码地址：https://github.com/xiajingren/HelloAbp

