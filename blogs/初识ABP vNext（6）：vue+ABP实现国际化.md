[TOC]

# 前言

上一篇介绍了ABP扩展实体，并且在前端部分新增了身份认证管理和租户管理的菜单，在实现这两个功能模块前，先来解决一下界面文字国际化的问题。



# 开始

国际化（简称 I18N），本地化（简称 L10N）；这两者的目的都是用于让你的应用程序支持多个国家和区域的语言，它们看起来很相似，但是有一些细微的区别，本文不对此进行深入探讨，有兴趣的可以自行搜索。ABP后端支持的是本地化，而vue-element-admin支持的是国际化，使用**vue-i18n**实现；本文默认它两者是一回事。

前面的章节中，已经大概分析了vue+ABP国际化的实现思路。我们可以在后端实现国际化，然后vue从后端获取国际化文本，展示到界面中；另一种方式是直接在前端部分实现国际化。在前端实现就很简单，直接在vue-element-admin的`src\lang\`目录下配置相应的文本，然后界面使用i18n的`$t()`方法渲染就可以了，这个不多做介绍。本文只探讨第一种实现方式。



## 语言选项

首先，语言选项列表需要根据后端配置得到。

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815153213605-1308548692.png)

在后端修改支持的语言类型，这里就只支持中文和英文2种吧，其他的注释掉。

src\Xhznl.HelloAbp.HttpApi.Host\HelloAbpHttpApiHostModule.cs：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815154714009-655262813.png)

请求`abp/application-configuration`接口：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815155118711-186493352.png)

此时返回的localization.languages属性只有2个语言了，然后只需要把这个数据绑定到界面上就好了。语言切换用的是一个公共组件 src\components\LangSelect\index.vue：

```vue
<template>
  <el-dropdown
    trigger="click"
    class="international"
    @command="handleSetLanguage"
  >
    <div>
      <svg-icon class-name="international-icon" icon-class="language" />
    </div>
    <el-dropdown-menu slot="dropdown">
      <el-dropdown-item
        v-for="item in languages"
        :key="item.cultureName"
        :disabled="language === item.cultureName"
        :command="item.cultureName"
      >
        {{ item.displayName }}
      </el-dropdown-item>
    </el-dropdown-menu>
  </el-dropdown>
</template>

<script>
export default {
  data() {
    return {
      languages: this.$store.getters.abpConfig.localization.languages
    };
  },
  computed: {
    language() {
      return this.$store.getters.language;
    }
  },
  methods: {
    handleSetLanguage(lang) {
      //this.$i18n.locale = lang
      this.$store.dispatch("app/setLanguage", lang);
      this.$store.dispatch("app/applicationConfiguration").then(() => {
        this.$message({
          message: "Switch Language Success",
          type: "success"
        });
      });
    }
  }
};
</script>
```

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815164003204-797109963.png)

## 语言切换

语言切换时，需要再次调用`app/applicationConfiguration`接口，更新本地化文本。

src\utils\request.js：

```js
// request interceptor
service.interceptors.request.use(
  config => {
    // do something before request is sent
    config.headers['accept-language'] = store.getters.language

    if (store.getters.token) {
      config.headers['authorization'] = 'Bearer ' + getToken()
    }
    return config
  },
  error => {
    // do something with request error
    console.log(error) // for debug
    return Promise.reject(error)
  }
)
```

src\store\modules\app.js：

```js
const actions = {
  。。。。。。
    
  applicationConfiguration({ commit }) {
    return new Promise((resolve, reject) => {
      applicationConfiguration()
        .then(response => {
          const data = response;
          commit("SET_ABPCONFIG", data);

          const language = data.localization.currentCulture.cultureName;
          const values = data.localization.values;
          setLocale(language, values);

          resolve(data);
        })
        .catch(error => {
          reject(error);
        });
    });
  }
};
```

src\lang\index.js：

```js
import Vue from "vue";
import VueI18n from "vue-i18n";
import Cookies from "js-cookie";
import elementEnLocale from "element-ui/lib/locale/lang/en"; // element-ui lang
import elementZhLocale from "element-ui/lib/locale/lang/zh-CN"; // element-ui lang

Vue.use(VueI18n);

const messages = {
  en: {
    ...elementEnLocale
  },
  "zh-Hans": {
    ...elementZhLocale
  }
};

export function getLanguage() {
  const chooseLanguage = Cookies.get("language");
  if (chooseLanguage) return chooseLanguage;

  // if has not choose language
  const language = (
    navigator.language || navigator.browserLanguage
  ).toLowerCase();
  const locales = Object.keys(messages);
  for (const locale of locales) {
    if (language.indexOf(locale) > -1) {
      return locale;
    }
  }
  return "en";
}
export function setLocale(language, values) {
  i18n.mergeLocaleMessage(language, values);
  i18n.locale = language;
}
const i18n = new VueI18n({
  // set locale
  // options: en | zh | es
  locale: getLanguage(),
  // set locale messages
  messages
});

export default i18n;
```

将后端返回的文本设置到vue-i18n中，就可以使用了。这跟直接在前端做国际化有一点区别就是，后者的文本信息是写在前端，vue-i18n可以直接使用。而这里只是把文本信息改到后端，从后端获取后再设置到i18n中，本质是一样的。

修改后端的配置文本：

src\Xhznl.HelloAbp.Domain.Shared\Localization\HelloAbp\zh-Hans.json：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815165949871-1279611621.png)

src\Xhznl.HelloAbp.Domain.Shared\Localization\HelloAbp\en.json：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815171435896-1074821742.png)

localization.values返回：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815170306687-904958250.png)

接下来只需要把界面上对应的文本使用vue-i18n的`$t()`方法渲染就好了，比如：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815170753017-375998060.png)

前端需要改动的地方比较多，但都是类似的修改。。。直接看效果：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815172950076-1564810273.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815173009829-1257301151.png)

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815172852552-1192780957.gif)

### 注意

因为`app/applicationConfiguration`接口只有在刷新页面、登录、退出、切换语言等操作的时候才会去调用，所以不用担心请求频繁。

其实上面有一部分本地化文本还是放在了前端：ElementUI自带的文本。因为ABP的本地化json格式只能有一级，key/value：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815174018143-1208829445.png)

文本只能写在texts属性中，key/value形式，不支持多层级。

而vue-i18n是支持多层级的：

![](https://img2020.cnblogs.com/blog/610959/202008/610959-20200815174421595-1027246779.png)

所以ElementUI的这部分文本还是放在前端了。



# 最后

本篇关于vue+ABP实现国际化就介绍完了。。。其实还是有点繁琐的，要配置的比较多，不知道有没有更好的方法，欢迎评论交流。。。


