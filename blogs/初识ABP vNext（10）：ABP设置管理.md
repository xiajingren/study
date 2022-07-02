[TOC]

# 前言

上一篇介绍了ABP模块化开发的基本步骤，完成了一个简单的文件上传功能。通常的模块都有一些自己的配置信息，比如上篇讲到的`FileOptions`类，其中配置了文件的上传目录，允许的文件大小和允许的文件类型。配置信息可以通过[Configuration](https://docs.abp.io/zh-Hans/abp/latest/Configuration)（配置）和[Options](https://docs.abp.io/zh-Hans/abp/latest/Options)（选项）来完成，ABP还提供了另一种更灵活的方式： [Settings](https://docs.abp.io/zh-Hans/abp/latest/Settings)（设置），本篇就来介绍一下ABP的设置管理。



# 开始

回顾一下上篇的`FileOptions`：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200912151714042-292001813.png)

首先定义了一个`FileOptions`类，其中包含了几个配置，然后在需要的地方中注入`IOptions<FileOptions>`就可以使用这些信息了。

当然，模块启动时可以做一些配置修改，比如：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200912151951119-1559743263.png)

无论是配置文件还是这种代码形式的配置，都是程序层面的修改；有些配置不太适合这样做，比如这里的`AllowedMaxFileSize`和`AllowedUploadFormats`，它们应该在应用界面上，可以让管理员自行修改。下面就来改造一下程序。

## 定义设置

使用设置之前需要先定义它，不同的模块可以拥有不同的设置。

modules\file-management\src\Xhznl.FileManagement.Domain\Settings\FileManagementSettingDefinitionProvider.cs：

```csharp
public class FileManagementSettingDefinitionProvider : SettingDefinitionProvider
{
    public override void Define(ISettingDefinitionContext context)
    {
        /* Define module settings here.
         * Use names from FileManagementSettings class.
         */

        context.Add(new SettingDefinition(
            FileManagementSettings.AllowedMaxFileSize,
            "1024",
            L("DisplayName:FileManagement.AllowedMaxFileSize"),
            L("Description:FileManagement.AllowedMaxFileSize")
            )
                .WithProperty("Group1", "File")
                .WithProperty("Group2", "Upload")
                .WithProperty("Type", "number"),

            new SettingDefinition(
                FileManagementSettings.AllowedUploadFormats,
                ".jpg,.jpeg,.png,.gif,.txt",
                L("DisplayName:FileManagement.AllowedUploadFormats"),
                L("Description:FileManagement.AllowedUploadFormats")
            )
                .WithProperty("Group1", "File")
                .WithProperty("Group2", "Upload")
                .WithProperty("Type", "text")
            );
    }

    private static LocalizableString L(string name)
    {
        return LocalizableString.Create<FileManagementResource>(name);
    }
}
```

以上代码定了了2个配置：`AllowedMaxFileSize`和`AllowedUploadFormats`，设置了它们的默认值、名称和详细说明。因为本项目使用了EasyAbp的SettingUi模块，所以会有一些`Group1`，`Group2`之类的字段，具体介绍可以参考[Abp.SettingUi](https://github.com/EasyAbp/Abp.SettingUi)

## 使用设置

想读取设置信息，只需注入`ISettingProvider`即可。因为父类`ApplicationService`中已经注入，所以这里直接使用`SettingProvider`就好。获取到配置，然后就可以做一些逻辑处理，比如判断上传文件的大小和格式是否合法：

```csharp
public class FileAppService : FileManagementAppService, IFileAppService
{
    ......

    [Authorize]
    public virtual async Task<string> CreateAsync(FileUploadInputDto input)
    {
        var allowedMaxFileSize = await SettingProvider.GetAsync<int>(FileManagementSettings.AllowedMaxFileSize);//kb
        var allowedUploadFormats = (await SettingProvider.GetOrNullAsync(FileManagementSettings.AllowedUploadFormats))
            ?.Split(",", StringSplitOptions.RemoveEmptyEntries);

        if (input.Bytes.Length > allowedMaxFileSize * 1024)
        {
            throw new UserFriendlyException(L["FileManagement.ExceedsTheMaximumSize", allowedMaxFileSize]);
        }

        if (allowedUploadFormats == null || !allowedUploadFormats.Contains(Path.GetExtension(input.Name)))
        {
            throw new UserFriendlyException(L["FileManagement.NotValidFormat"]);
        }

        ......
    }
}
```

前端设置界面：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200914173633744-166856438.png)

下面可以随便修改下设置，进行测试：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200914173746754-653827974.png)



# 最后

本篇内容较少，希望对你有帮助。代码已上传至 https://github.com/xiajingren/HelloAbp ，欢迎star。

