[TOC]

# 前言

上一篇已经创建好了前后端项目，本篇开始编码部分。



# 开始

几乎所有的系统都绕不开登录功能，那么就从登录开始，完成用户登录以及用户菜单权限控制。

## 登录

首先用户输入账号密码点击登录，然后组合以下参数调用identityserver的`/connect/token`端点获取token：

```json
{
  grant_type: "password",
  scope: "HelloAbp",
  username: "",
  password: "",
  client_id: "HelloAbp_App",
  client_secret: "1q2w3e*"
}
```

这个参数来自ABP模板的种子数据：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812101059692-735307811.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812101202182-1379040031.png)

我使用的是password flow，这个flow无需重定向。如果你的网站应用只有一个的话，可以这么做，如果有多个的话建议采用其他oidc方式，把认证界面放到identityserver程序里，客户端重定向到identityserver去认证，这样其实更安全，并且你无需在每个客户端网站都做一遍登录界面和逻辑。。。

还有一点，严格来说不应该直接访问`/connect/token`端点获取token。首先应该从identityserver发现文档`/.well-known/openid-configuration`中获取配置信息，然后从`/.well-known/openid-configuration/jwks`端点获取公钥等信息用于校验token合法性，最后才是获取token。ABP的Angular版本就是这么做的，不过他是使用`angular-oauth2-oidc`这个库完成，我暂时没有找到其他的支持password flow的开源库，参考：https://github.com/IdentityModel/oidc-client-js/issues/234

前端想正常访问接口，首先需要在HttpApi.Host，IdentityServer增加跨域配置：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812102321207-1018751145.png)



![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812102152761-331590081.png)

前端部分需要修改的文件太多，下面只贴出部分主要代码，需要完整源码的可以去GitHub拉取。

src\store\modules\user.js：

```js
const clientSetting = {
  grant_type: "password",
  scope: "HelloAbp",
  username: "",
  password: "",
  client_id: "HelloAbp_App",
  client_secret: "1q2w3e*"
};
const actions = {
  // user login
  login({ commit }, userInfo) {
    const { username, password } = userInfo
    return new Promise((resolve, reject) => {
      clientSetting.username = username.trim()
      clientSetting.password = password
      login(clientSetting)
        .then(response => {
          const data = response
          commit('SET_TOKEN', data.access_token)
          setToken(data.access_token).then(() => {
            resolve()
          })
        })
        .catch(error => {
          reject(error)
        })
    })
  },

  // get user info
  getInfo({ commit }) {
    return new Promise((resolve, reject) => {
      getInfo()
        .then(response => {
          const data = response

          if (!data) {
            reject('Verification failed, please Login again.')
          }

          const { name } = data

          commit('SET_NAME', name)
          commit('SET_AVATAR', '')
          commit('SET_INTRODUCTION', '')
          resolve(data)
        })
        .catch(error => {
          reject(error)
        })
    })
  },

  setRoles({ commit }, roles) {
    commit('SET_ROLES', roles)
  },

  // user logout
  logout({ commit, dispatch }) {
    return new Promise((resolve, reject) => {
      logout()
        .then(() => {
          commit('SET_TOKEN', '')
          commit('SET_NAME', '')
          commit('SET_AVATAR', '')
          commit('SET_INTRODUCTION', '')
          commit('SET_ROLES', [])
          removeToken().then(() => {
            resetRouter()
            // reset visited views and cached views
            // to fixed https://github.com/PanJiaChen/vue-element-admin/issues/2485
            dispatch('tagsView/delAllViews', null, { root: true })

            resolve()
          })
        })
        .catch(error => {
          reject(error)
        })
    })
  },

  // remove token
  resetToken({ commit }) {
    return new Promise(resolve => {
      commit('SET_TOKEN', '')
      commit('SET_NAME', '')
      commit('SET_AVATAR', '')
      commit('SET_INTRODUCTION', '')
      commit('SET_ROLES', [])
      removeToken().then(() => {
        resolve()
      })
    })
  }
}
```

src\utils\auth.js：

```js
export async function setToken(token) {
  const result = Cookies.set(TokenKey, token);
  await store.dispatch("app/applicationConfiguration");
  return result;
}

export async function removeToken() {
  const result = Cookies.remove(TokenKey);
  await store.dispatch("app/applicationConfiguration");
  return result;
}
```

src\api\user.js：

```js
export function login(data) {
  return request({
    baseURL: "https://localhost:44364",
    url: "/connect/token",
    method: "post",
    headers: { "content-type": "application/x-www-form-urlencoded" },
    data: qs.stringify(data),
  });
}

export function getInfo() {
  return request({
    url: "/api/identity/my-profile",
    method: "get",
  });
}

export function logout() {
  return request({
    baseURL: "https://localhost:44364",
    url: "/api/account/logout",
    method: "get",
  });
}
```

src\utils\request.js：

```js
service.interceptors.request.use(
  (config) => {
    // do something before request is sent

    if (store.getters.token) {
      config.headers["authorization"] = "Bearer " + getToken();
    }
    return config;
  },
  (error) => {
    // do something with request error
    console.log(error); // for debug
    return Promise.reject(error);
  }
);

// response interceptor
service.interceptors.response.use(
  (response) => {
    const res = response.data;

    return res;
  },
  (error) => {
    console.log("err" + error); // for debug
    Message({
      message: error.message,
      type: "error",
      duration: 5 * 1000,
    });

    if (error.status === 401) {
      // to re-login
      MessageBox.confirm(
        "You have been logged out, you can cancel to stay on this page, or log in again",
        "Confirm logout",
        {
          confirmButtonText: "Re-Login",
          cancelButtonText: "Cancel",
          type: "warning",
        }
      ).then(() => {
        store.dispatch("user/resetToken").then(() => {
          location.reload();
        });
      });
    }

    return Promise.reject(error);
  }
);
```

## 菜单权限

vue-element-admin的菜单权限是使用用户角色来控制的，我们不需要role。前面分析过，通过`/api/abp/application-configuration`接口的auth.grantedPolicies字段，与对应的菜单路由绑定，就可以实现权限控制了。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812105328542-1868756016.png)

src\permission.js：

```js
router.beforeEach(async (to, from, next) => {
  // start progress bar
  NProgress.start();

  // set page title
  document.title = getPageTitle(to.meta.title);

  let abpConfig = store.getters.abpConfig;
  if (!abpConfig) {
    abpConfig = await store.dispatch("app/applicationConfiguration");
  }

  if (abpConfig.currentUser.isAuthenticated) {
    if (to.path === "/login") {
      // if is logged in, redirect to the home page
      next({ path: "/" });
      NProgress.done(); // hack: https://github.com/PanJiaChen/vue-element-admin/pull/2939
    } else {
      //user name
      const name = store.getters.name;

      if (name) {
        next();
      } else {
        try {
          // get user info
          await store.dispatch("user/getInfo");

          store.dispatch("user/setRoles", abpConfig.currentUser.roles);
            
          const grantedPolicies = abpConfig.auth.grantedPolicies;

          // generate accessible routes map based on grantedPolicies
          const accessRoutes = await store.dispatch(
            "permission/generateRoutes",
            grantedPolicies
          );

          // dynamically add accessible routes
          router.addRoutes(accessRoutes);

          // hack method to ensure that addRoutes is complete
          // set the replace: true, so the navigation will not leave a history record
          next({ ...to, replace: true });
        } catch (error) {
          // remove token and go to login page to re-login
          await store.dispatch("user/resetToken");
          Message.error(error || "Has Error");
          next(`/login?redirect=${to.path}`);
          NProgress.done();
        }
      }
    }
  } else {
    if (whiteList.indexOf(to.path) !== -1) {
      // in the free login whitelist, go directly
      next();
    } else {
      // other pages that do not have permission to access are redirected to the login page.
      next(`/login?redirect=${to.path}`);
      NProgress.done();
    }
  }
});
```

src\store\modules\permission.js：

```js
function hasPermission(grantedPolicies, route) {
  if (route.meta && route.meta.policy) {
    const policy = route.meta.policy;
    return grantedPolicies[policy];
  } else {
    return true;
  }
}

export function filterAsyncRoutes(routes, grantedPolicies) {
  const res = [];

  routes.forEach((route) => {
    const tmp = { ...route };
    if (hasPermission(grantedPolicies, tmp)) {
      if (tmp.children) {
        tmp.children = filterAsyncRoutes(tmp.children, grantedPolicies);
      }
      res.push(tmp);
    }
  });

  return res;
}

const state = {
  routes: [],
  addRoutes: [],
};

const mutations = {
  SET_ROUTES: (state, routes) => {
    state.addRoutes = routes;
    state.routes = constantRoutes.concat(routes);
  },
};

const actions = {
  generateRoutes({ commit }, grantedPolicies) {
    return new Promise((resolve) => {
      let accessedRoutes = filterAsyncRoutes(asyncRoutes, grantedPolicies);
      commit("SET_ROUTES", accessedRoutes);
      resolve(accessedRoutes);
    });
  },
};
```

src\router\index.js：

```js
export const asyncRoutes = [
  {
    path: '/permission',
    component: Layout,
    redirect: '/permission/page',
    alwaysShow: true, // will always show the root menu
    name: 'Permission',
    meta: {
      title: 'permission',
      icon: 'lock',
      policy: 'AbpIdentity.Roles'
    },
    children: [
      {
        path: 'page',
        component: () => import('@/views/permission/page'),
        name: 'PagePermission',
        meta: {
          title: 'pagePermission',
          policy: 'AbpIdentity.Roles'
        }
      },
      {
        path: 'directive',
        component: () => import('@/views/permission/directive'),
        name: 'DirectivePermission',
        meta: {
          title: 'directivePermission',
          policy: 'AbpIdentity.Roles'
        }
      },
      {
        path: 'role',
        component: () => import('@/views/permission/role'),
        name: 'RolePermission',
        meta: {
          title: 'rolePermission',
          policy: 'AbpIdentity.Roles'
        }
      }
    ]
  },

  。。。。。。

  // 404 page must be placed at the end !!!
  { path: '*', redirect: '/404', hidden: true }
]
```

因为菜单太多了，就拿其中的一个“权限测试页”菜单举例，将它与AbpIdentity.Roles绑定测试。

## 运行测试

运行前后端项目，使用默认账号admin/1q2w3E*登录系统：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812114347246-1031799810.png)

正常的话就可以进入这个界面了：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812115855889-335942807.png)

目前可以看到“权限测试页”菜单，因为现在还没有设置权限的界面，所以我手动去数据库把这条权限数据删除，然后测试一下：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812120437552-549508790.png)

但是手动去数据库改这个表的话会有很长一段时间的缓存，在redis中，暂时没去研究这个缓存机制，正常通过接口修改应该不会这样。。。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812152715218-633964945.png)

我手动清理了redis，运行结果如下：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200812152943841-275384445.png)



# 最后

本篇实现了前端部分的登录和菜单权限控制，但是还有很多细节问题需要处理。比如右上角的用户头像，ABP的默认用户表中是没有头像和用户介绍字段的，下篇将完善这些问题，还有删除掉vue-element-admin多余的菜单。

