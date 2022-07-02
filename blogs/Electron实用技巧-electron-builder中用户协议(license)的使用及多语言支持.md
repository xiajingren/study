![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210119172627251-1885769261.png)



# # 1

通常pc软件的安装过程中，会加入用户协议，如：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121134128251-182644407.png)

下面介绍一下使用`electron-builder`打包应用，如何加入license。首先参考官网介绍：windows：[nsis](https://www.electron.build/configuration/nsis)，macOS：[dmg](https://www.electron.build/configuration/dmg)



# # 2

官网上关于license配置说明写的不是很详细，下面是我实践总结出的正确的姿势：

最简单的方法是在你的项目`/build`目录下新建`license.text`文件，然后正常打包就可以了，无需其他设置。

注意，这里有一个中文乱码的问题，如果只考虑windows系统的话，编码可以选择`ANSI`，就不会乱码了。

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121140003945-973477417.png)

但是`ANSI`在macOS下是不行的，所以更推荐的方案是使用 “带有BOM的UTF-8“，这样在windows，macOS下都可以使用。

![image-20210122100244716](https://img2020.cnblogs.com/blog/610959/202101/610959-20210122100247162-711641686.png)

`/build`是`electron-builder`默认资源目录，也可以修改，比如我这里是`public`目录：

```javascript
directories: {
  buildResources: "./public",
}
```

这样`license.text`文件就放在`/public`目录下即可。

如果没有多语言需求的话，这样就结束了，windows，macOS通用。



# # 3

如果要支持多语言，只需修改license文件名添加对应的语言代码后缀，如：license_xxx.txt。关于语言代码官网给出的参考是[language code to name](https://github.com/meikidd/iso-639-1/blob/master/src/data.js)，这里有个错误，中文对应的是`zh`，实际上简体中文应该写`zh_CN`。

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121142715809-691470971.png)

下面在我的`/public`目录下新建`license_en.txt`和`license_zh_CN.txt`：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121142825014-1321215860.png)

为了测试多语言，我增加一个语言选择配置`displayLanguageSelector`（正常不建议使用这个配置，默认跟随系统语言）：

```javascript
nsis: {
  oneClick: false,
  allowToChangeInstallationDirectory: true,
  
  displayLanguageSelector: true,
},
```

打包后安装，选择语言：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121145057363-765641431.png)

英文：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121123008383-964224700.png)

中文：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210121131044803-769099593.png)

macOS：

![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210122102006296-1128643070.png)

