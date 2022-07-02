[TOC]

# 前言

上一篇提到了ABP功能管理（特征管理），它来自ABP的FeatureManagement模块，ABP官方文档貌似还没有这个模块的相关说明，但是个人感觉这个模块非常实用，下面就简单介绍一个特征管理的基本应用。



# 开始

在租户管理中，有一个“管理功能”按钮，默认是没有数据的，界面上也没有地方维护。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200902161359148-1307148214.png)

特征管理简单来说就是在同一套系统中为不同的租户提供一些差异化的功能。比如免费用户，提供的是基础功能，VIP用户则会多一些高级功能。

## 定义特征

在Application.Contracts项目中添加Features文件夹。

src\Xhznl.HelloAbp.Application.Contracts\Features\HelloAbpFeatures.cs：

```csharp
public class HelloAbpFeatures
{
    public const string GroupName = "HelloAbp";

    public const string SocialLogins = GroupName + ".SocialLogins";
    public const string UserCount = GroupName + ".UserCount";
}
```

src\Xhznl.HelloAbp.Application.Contracts\Features\HelloAbpFeatureDefinitionProvider.cs：

```csharp
public class HelloAbpFeatureDefinitionProvider : FeatureDefinitionProvider
{
    public override void Define(IFeatureDefinitionContext context)
    {
        var group = context.AddGroup(HelloAbpFeatures.GroupName);

        group.AddFeature(HelloAbpFeatures.SocialLogins, "true", L("Feature:SocialLogins")
            , valueType: new ToggleStringValueType());
        group.AddFeature(HelloAbpFeatures.UserCount, "10", L("Feature:UserCount")
            , valueType: new FreeTextStringValueType(new NumericValueValidator(1, 1000)));
    }

    private static LocalizableString L(string name)
    {
        return LocalizableString.Create<HelloAbpResource>(name);
    }
}
```

以上代码添加了2个特征：SocialLogins，UserCount。

SocialLogins（社交登录），valueType为ToggleStringValueType，意味着它是个勾选框，默认值为"true"。

UserCount（用户数量），valueType为FreeTextStringValueType，意味着它是个输入框，默认值为"10"。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200902175143473-290258241.png)

现在可以为不同租户设置不同的特征值。

## 应用特征

特征值定义好了，接下来就是如何应用了，首先看一下用户数量如何控制。

### 用户数量

目前用户是通过`/identity/users`接口来添加的，那么我们重写这个接口对应的服务方法就好了。关于重写服务可以参考：[重写服务](https://docs.abp.io/zh-Hans/abp/latest/Customizing-Application-Modules-Overriding-Services)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903114748250-736690648.png)

对应的ABP源码在：abp\modules\identity\src\Volo.Abp.Identity.Application\Volo\Abp\Identity\IdentityUserAppService.cs中。

在我们的Application项目中添加一个服务类继承`IdentityUserAppService`，重写`CreateAsync`方法，使用`FeatureChecker`获取到特征值，然后做个用户数量校验即可。

src\Xhznl.HelloAbp.Application\Identity\HelloIdentityUserAppService.cs：

```csharp
[RemoteService(IsEnabled = false)]
[Dependency(ReplaceServices = true)]
[ExposeServices(typeof(IIdentityUserAppService), typeof(IdentityUserAppService))]
public class HelloIdentityUserAppService : IdentityUserAppService, IHelloIdentityUserAppService
{
    private readonly IStringLocalizer<HelloAbpResource> _localizer;

    public HelloIdentityUserAppService(IdentityUserManager userManager,
        IIdentityUserRepository userRepository,
        IIdentityRoleRepository roleRepository,
        IStringLocalizer<HelloAbpResource> localizer) : base(userManager, userRepository, roleRepository)
    {
        _localizer = localizer;
    }

    public override async Task<IdentityUserDto> CreateAsync(IdentityUserCreateDto input)
    {
        var userCount = (await FeatureChecker.GetOrNullAsync(HelloAbpFeatures.UserCount)).To<int>();
        var currentUserCount = await UserRepository.GetCountAsync();
        if (currentUserCount >= userCount)
        {
            throw new UserFriendlyException(_localizer["Feature:UserCount.Maximum", userCount]);
        }

        return await base.CreateAsync(input);
    }
}
```

下面可以将某租户的用户数量设置一下，测试是否有效果：

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903151521350-701776470.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903151333237-231611813.png)

这样，就实现了对不同租户用户数量的限制。

### 社交登录

特征值也可以在前端使用，在`/abp/application-configuration`中就可以获取到。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903152805234-582385026.png)

拿到特征值，前端也可以做一些差异化功能，比如这里的是否支持社交登录。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903154249927-186656135.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903154232397-191624765.png)



----

关于Feature就简单介绍到这里，本项目源码放在：https://github.com/xiajingren/HelloAbp 

另外非常感谢热心小伙@[jonny-xhl](https://github.com/jonny-xhl)给添加的设置模块（来自EasyAbp的[Abp.SettingUi](https://github.com/EasyAbp/Abp.SettingUi)）。

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903155640781-1377849112.png)

![](https://img2020.cnblogs.com/blog/610959/202009/610959-20200903155721372-28113862.png)



# 最后

本文只是对Feature的最基本介绍，关于Feature，还有很多实用的API方法，基于Feature可以满足很多定制化需求，想深入了解的话可以看下Abp.FeatureManagement源码。

感谢@jonny-xhl的pr。

