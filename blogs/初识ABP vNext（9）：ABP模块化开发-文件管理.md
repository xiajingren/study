[TOC]

# 前言

在之前的章节中介绍过ABP扩展实体，当时在用户表扩展了用户头像字段，用户头像就涉及到文件上传和文件存储。文件上传是很多系统都会涉及到的一个基础功能，在ABP的模块化思路下，文件管理可以做成一个通用的模块，便于以后在多个项目中复用。单纯实现一个文件上传的功能并不复杂，本文就借着这个简单的功能来介绍一下ABP模块化开发的最基本步骤。



# 开始

## 创建模块

首先使用ABP CLI创建一个模块：`abp new Xhznl.FileManagement -t module --no-ui`

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910172558639-780117902.png)

创建完成后会得到如下文件：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910172811988-1656283782.png)

在主项目中添加对应模块的引用，Application=>Application，Domain=>Domain，HttpApi=>HttpApi 等等。例如：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910174159657-385790504.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910174252923-1500539398.png)

需要添加引用的项目：Application、Application.Contracts、Domain、Domain.Shared、EntityFrameworkCore、HttpApi、HttpApi.Client

手动添加这些引用比较麻烦，你可以搭建自己的私有NuGet服务器，把模块的包发布到私有NuGet上，然后通过NuGet来安装引用。两种方式各有优缺点，具体请参考[自定义现有模块](https://docs.abp.io/zh-Hans/abp/latest/Customizing-Application-Modules-Guide)，关于私有NuGet搭建可以参考：[十分钟搭建自己的私有NuGet服务器-BaGet](https://www.cnblogs.com/xhznl/p/13426918.html)。

然后给这些项目的模块类添加对应的依赖，例如：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910194927720-1698177383.png)

通过上面的方式引用模块，使用visual studio是无法编译通过的：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910174502648-1014353291.png)

需要在解决方案目录下，手动执行`dotnet restore`命令即可：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200910175401258-1508422500.png)

## 模块开发

接下来关于文件管理功能的开发，都在模块Xhznl.FileManagement中进行，它是一个独立的解决方案。初学ABP，下面就以尽量简单的方式来实现这个模块。

### 应用服务

模块开发通常从Domain层实体建立开始，但是这里先跳过。先在FileManagement.Application.Contracts项目添加应用服务接口和Dto。

modules\file-management\src\Xhznl.FileManagement.Application.Contracts\Files\IFileAppService.cs：

```csharp
public interface IFileAppService : IApplicationService
{
    Task<byte[]> GetAsync(string name);

    Task<string> CreateAsync(FileUploadInputDto input);
}
```

modules\file-management\src\Xhznl.FileManagement.Application.Contracts\Files\FileUploadInputDto.cs：

```csharp
public class FileUploadInputDto
{
    [Required]
    public byte[] Bytes { get; set; }

    [Required]
    public string Name { get; set; }
}
```

然后是FileManagement.Application项目，实现应用服务，先定义一个配置类。

modules\file-management\src\Xhznl.FileManagement.Application\Files\FileOptions.cs：

```csharp
public class FileOptions
{
    /// <summary>
    /// 文件上传目录
    /// </summary>
    public string FileUploadLocalFolder { get; set; }

    /// <summary>
    /// 允许的文件最大大小
    /// </summary>
    public long MaxFileSize { get; set; } = 1048576;//1MB

    /// <summary>
    /// 允许的文件类型
    /// </summary>
    public string[] AllowedUploadFormats { get; set; } = { ".jpg", ".jpeg", ".png", "gif", ".txt" };
}
```

modules\file-management\src\Xhznl.FileManagement.Application\Files\FileAppService.cs：

```csharp
public class FileAppService : FileManagementAppService, IFileAppService
{
    private readonly FileOptions _fileOptions;

    public FileAppService(IOptions<FileOptions> fileOptions)
    {
        _fileOptions = fileOptions.Value;
    }

    public Task<byte[]> GetAsync(string name)
    {
        Check.NotNullOrWhiteSpace(name, nameof(name));

        var filePath = Path.Combine(_fileOptions.FileUploadLocalFolder, name);

        if (File.Exists(filePath))
        {
            return Task.FromResult(File.ReadAllBytes(filePath));
        }

        return Task.FromResult(new byte[0]);
    }

    [Authorize]
    public Task<string> CreateAsync(FileUploadInputDto input)
    {
        if (input.Bytes.IsNullOrEmpty())
        {
            throw new AbpValidationException("Bytes can not be null or empty!",
                new List<ValidationResult>
                {
                    new ValidationResult("Bytes can not be null or empty!", new[] {"Bytes"})
                });
        }

        if (input.Bytes.Length > _fileOptions.MaxFileSize)
        {
            throw new UserFriendlyException($"File exceeds the maximum upload size ({_fileOptions.MaxFileSize / 1024 / 1024} MB)!");
        }

        if (!_fileOptions.AllowedUploadFormats.Contains(Path.GetExtension(input.Name)))
        {
            throw new UserFriendlyException("Not a valid file format!");
        }

        var fileName = Guid.NewGuid().ToString("N") + Path.GetExtension(input.Name);
        var filePath = Path.Combine(_fileOptions.FileUploadLocalFolder, fileName);

        if (!Directory.Exists(_fileOptions.FileUploadLocalFolder))
        {
            Directory.CreateDirectory(_fileOptions.FileUploadLocalFolder);
        }

        File.WriteAllBytes(filePath, input.Bytes);

        return Task.FromResult("/api/file-management/files/" + fileName);
    }
}
```

服务实现很简单，就是基于本地文件系统的读写操作。

下面是FileManagement.HttpApi项目，添加控制器，暴露服务API接口。

modules\file-management\src\Xhznl.FileManagement.HttpApi\Files\FileController.cs：

```csharp
[RemoteService]
[Route("api/file-management/files")]
public class FileController : FileManagementController
{
    private readonly IFileAppService _fileAppService;

    public FileController(IFileAppService fileAppService)
    {
        _fileAppService = fileAppService;
    }

    [HttpGet]
    [Route("{name}")]
    public async Task<FileResult> GetAsync(string name)
    {
        var bytes = await _fileAppService.GetAsync(name);
        return File(bytes, MimeTypes.GetByExtension(Path.GetExtension(name)));
    }

    [HttpPost]
    [Route("upload")]
    [Authorize]
    public async Task<JsonResult> CreateAsync(IFormFile file)
    {
        if (file == null)
        {
            throw new UserFriendlyException("No file found!");
        }

        var bytes = await file.GetAllBytesAsync();
        var result = await _fileAppService.CreateAsync(new FileUploadInputDto()
        {
            Bytes = bytes,
            Name = file.FileName
        });
        return Json(result);
    }

}
```

### 运行模块

ABP的模板是可以独立运行的，在FileManagement.HttpApi.Host项目的模块类FileManagementHttpApiHostModule配置FileOptions：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911132722192-254591885.png)

修改FileManagement.HttpApi.Host和FileManagement.IdentityServer项目的数据库连接配置，然后启动这2个项目，不出意外的话可以看到如下界面。

FileManagement.HttpApi.Host：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911133218273-1796636936.png)

FileManagement.IdentityServer：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911133231215-1422063602.png)

现在你可以使用postman来测试一下File的2个API，当然也可以编写单元测试。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911135241123-1212104739.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911135304435-1298819144.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911135517597-557954077.png)

### 单元测试

更好的方法是编写单元测试，关于如何做好单元测试可以参考ABP源码，下面只做一个简单示例：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911140231014-126474242.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911140410237-1879670126.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911140453574-1542444913.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911140605769-598857386.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911140922024-290909767.png)



## 模块使用

模块测试通过后，回到主项目。模块引用，模块依赖前面都已经做好了，现在只需配置一下FileOptions，就可以使用了。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911144501836-1823056167.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200911144331274-1016096113.png)

目前FileManagement.Domain、FileManagement.Domain.Shared、FileManagement.EntityFrameworkCore这几个项目暂时没用到，项目结构也不是固定的，可以根据自己实际情况来调整。



# 最后

本文的模块示例比较简单，只是完成了一个文件上传和显示的基本功能，关于实体，数据库，领域服务，仓储之类的都暂时没用到。但是相信可以通过这个简单的例子，感受到ABP插件式的开发体验，这是一个好的开始，更多详细内容后面再做介绍。本文参考了ABP blogging模块的文件管理，关于文件存储，ABP中也有一个BLOB系统可以了解一下。



