![](https://img2020.cnblogs.com/blog/610959/202101/610959-20210119172627251-1885769261.png)



# # 1

在桌面软件中，开机自启动是很常见的功能，在electron中也提供了很好的支持，以下是主要代码：

```js
//应用是否打包
if (app.isPackaged) {
  //设置开机启动
  app.setLoginItemSettings({
    openAtLogin: true
  });
}

//应用是否打包
if (app.isPackaged) {
  //获取是否开机启动
  const { openAtLogin } = app.getLoginItemSettings();
  return openAtLogin;
}
```

设置开机启动后，如果不稍加处理，用户一开电脑，就会弹出你的软件窗口，这样不太友好。正常来说某些软件只有用户手动打开软件时才弹出主窗口，开机启动的话，只收起到系统托盘中会更好一些。



# # 2

参考electron开机启动相关文档：[appsetloginitemsettingssettings-macos-windows](https://www.electronjs.org/docs/api/app#appsetloginitemsettingssettings-macos-windows)

## windows

在windows下，`setLoginItemSettings`方法有一个`args`参数，利用这个参数就可以达到目的，以下是主要代码：

```javascript
//设置开机启动
app.setLoginItemSettings({
  openAtLogin: true,
  args: ["--openAsHidden"],
});


//获取是否开机启动
const { openAtLogin } = app.getLoginItemSettings({
  args: ["--openAsHidden"],
});
return openAtLogin;
```

设置开机启动时，在`args`中传入`--openAsHidden`，这个字符串可以随便更改。获取开机启动时，也要在`args`中传入同样的字符串，不然获取不到正确的值。

然后在显示主窗口时，先判断一下`process.argv`中是否包含`--openAsHidden`，如果包含，说明是开机自动启动的，这时候不显示窗口；相反 如果不包含`--openAsHidden`的话，说明是用户手动启动软件，这时正常显示窗口就好了：

```javascript
win.once("ready-to-show", () => {
  if (process.argv.indexOf("--openAsHidden") < 0) 
      win.show();
});
```

## macOS

mac下没有`args`参数，可以通过`openAsHidden`来实现。以下是主要代码：

```javascript
//设置开机启动
app.setLoginItemSettings({
  openAtLogin: true,
  openAsHidden: true,
});


//获取是否开机启动
const { openAtLogin } = app.getLoginItemSettings();
return openAtLogin;
```

光设置`openAsHidden: true`还不行，也需要做一下判断：

```javascript
win.once("ready-to-show", () => {
  if (!app.getLoginItemSettings().wasOpenedAsHidden) 
      win.show();
});
```



# # 3

以上就是我正在使用的Electron开机启动时隐藏主窗口的方法，显示系统托盘就用`Tray`就行，比较简单。如有其他更好的方式，欢迎交流。。。



