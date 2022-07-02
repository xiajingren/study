[TOC]

# 前言

上一篇介绍了vue+ABP国际化的基本实现，本篇开始功能模块的开发，首先完成ABP模板自带的身份认证管理模块和租户管理模块。同样的，参考ABP的Angular版本来做。



# 开始

功能模块的开发往往是最容易的，但是要处理好每个细节就不容易了。就拿这里的身份认证管理模块来说，看似很简单，因为后端接口都是ABP模板里现成的，前端部分无非就是写界面，调接口，绑数据；但是看一下ABP Angular版本的代码，就会发现他其实是有很多细节方面的处理的。

回到vue，因为前端部分的代码文件太多，下面只列出一些需要注意的细节，其他的像vue组件、表格、表单、数据绑定、接口请求之类的其实都差不多就不说了。



## 按钮级权限

前面章节中实现了菜单权限的控制，按钮权限的道理也是一样的。判断abpConfig.auth.grantedPolicies是否包含某个权限，然后在组件中使用`v-if`渲染就好了。

src\utils\abp.js：

```js
export function checkPermission(policy) {
  const abpConfig = store.getters.abpConfig;
  if (abpConfig.auth.grantedPolicies[policy]) {
    return true;
  } else {
    return false;
  }
}
```

src\views\identity\roles.vue：

```vue
<el-button
  class="filter-item"
  style="margin-left: 10px;"
  type="primary"
  icon="el-icon-edit"
  @click="handleCreate"
  v-if="checkPermission('AbpIdentity.Roles.Create')"
>
  {{ $t("AbpIdentity['NewRole']") }}
</el-button>
```

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819211013473-2066041972.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819211132013-331134211.png)

## 身份认证管理

角色和用户的增删改查就不说了，这里要注意一下权限管理。用户和角色都需要用到权限管理，在ABP Angular版中是一个独立的permission-management模块。我这里也把他作为一个公用组件，根据providerName来区分，"R"是角色权限，"U"是用户权限。

### R/U权限

他们有一点区别，用户权限可能来自于角色权限，所以用户中的权限需要显示是来自哪个providerName和providerKey，如果来自其他provider则disabled，不可以修改。

src\views\identity\components\permission-management.vue：

```vue
<el-form label-position="top">
  <el-tabs tab-position="left">
    <el-tab-pane
      v-for="group in permissionData.groups"
      :key="group.name"
      :label="group.displayName"
    >
      <el-form-item :label="group.displayName">
        <el-tree
          ref="permissionTree"
          :data="transformPermissionTree(group.permissions)"
          :props="treeDefaultProps"
          show-checkbox
          check-strictly
          node-key="name"
          default-expand-all
        />
      </el-form-item>
    </el-tab-pane>
  </el-tabs>
</el-form>
```

```js
transformPermissionTree(permissions, name = null) {
  let arr = [];
  if (!permissions || !permissions.some(v => v.parentName == name))
    return arr;
  const parents = permissions.filter(v => v.parentName == name);
  for (let i in parents) {
    let label = '';
    if (this.permissionsQuery.providerName == "R") {
      label = parents[i].displayName;
    } else if (this.permissionsQuery.providerName == "U") {
      label =
        parents[i].displayName +
        " " +
        parents[i].grantedProviders.map(provider => {
          return `${provider.providerName}: ${provider.providerKey}`;
        });
    }
    arr.push({
      name: parents[i].name,
      label,
      disabled: this.isGrantedByOtherProviderName(
        parents[i].grantedProviders
      ),
      children: this.transformPermissionTree(permissions, parents[i].name)
    });
  }
  return arr;
},
isGrantedByOtherProviderName(grantedProviders) {
  if (grantedProviders.length) {
    return (
      grantedProviders.findIndex(
        p => p.providerName !== this.permissionsQuery.providerName
      ) > -1
    );
  }
  return false;
}
```

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819213617660-603341209.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819213550493-1474456755.png)

### 权限刷新

还有一个细节问题，如果正在修改的权限影响到了当前用户，如何立即生效。

src\views\identity\components\permission-management.vue：

```js
updatePermissions(this.permissionsQuery, { permissions: tempData }).then(
  () => {
    this.dialogPermissionFormVisible = false;
    this.$notify({
      title: this.$i18n.t("HelloAbp['Success']"),
      message: this.$i18n.t("HelloAbp['SuccessMessage']"),
      type: "success",
      duration: 2000
    });
    fetchAppConfig(
      this.permissionsQuery.providerKey,
      this.permissionsQuery.providerName
    );
  }
);
```

src\utils\abp.js：

```js
function shouldFetchAppConfig(providerKey, providerName) {
  const currentUser = store.getters.abpConfig.currentUser;

  if (providerName === "R")
    return currentUser.roles.some(role => role === providerKey);

  if (providerName === "U") return currentUser.id === providerKey;

  return false;
}
export function fetchAppConfig(providerKey, providerName) {
  if (shouldFetchAppConfig(providerKey, providerName)) {
    store.dispatch("app/applicationConfiguration").then(abpConfig => {
      resetRouter();

      store.dispatch("user/setRoles", abpConfig.currentUser.roles);

      const grantedPolicies = abpConfig.auth.grantedPolicies;

      // generate accessible routes map based on grantedPolicies
      store
        .dispatch("permission/generateRoutes", grantedPolicies)
        .then(accessRoutes => {
          // dynamically add accessible routes
          router.addRoutes(accessRoutes);
        });

      // reset visited views and cached views
      //store.dispatch("tagsView/delAllViews", null, { root: true });
    });
  }
}
```

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819221633742-548994024.gif)



----

还有很多需要注意的，比如`isStatic===true`的角色不可以删除，并且不可以修改名称；新增用户和编辑用户的密码校验规则需要区别对待；保存权限是差异保存。等等。。。有条件的可以看一下ABP的Angular代码。

## 租户管理

基本功能界面都差不多。。。但是这里有一个”管理功能“的选项，默认是显示”没有可用的功能“：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200821223216201-1877139954.png)

这玩意在界面上没地方添加，也没地方删除，但是这个功能相当实用。它来自ABP的FeatureManagement模块，也称为”特征管理“，这个后面再做介绍。

### 租户切换

完成了租户管理，那么登录时也应该可以切换租户。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200824160701611-540824840.png)

切换租户比较简单，就是根据输入的租户名称获取到租户ID，然后调用`/abp/application-configuration`接口，把租户ID放到请求Header的**__tenant**字段中即可，之后的请求中也需要这个参数，不传的话就是默认的宿主端。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200824161016597-973050504.png)

其实ABP后端是可以配置是否启用多租户的，这里也可以根据后端配置来显示或者隐藏租户切换的按钮。跟ABP模板相比，登录界面还缺少一个注册入口，后面再加上吧。



## 效果

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819225612137-1997525523.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819225627520-1459996490.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819225759445-2050487330.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200819225844482-1130287626.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200821223023332-2042194059.png)



# 最后

前端部分的模块开发就不再详细介绍了，主题还是ABP。进行到这里，ABP模板自带的前端部分功能就差不多完成了，需要代码的可以去 https://github.com/xiajingren/HelloAbp 拉取，后面我再把文件整理一下，弄一个干净的vue版本。

