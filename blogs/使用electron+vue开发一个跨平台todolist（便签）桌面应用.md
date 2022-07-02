# # 1

最近一直在使用electron开发桌面应用，对于一个web开发者来说，html+javascript+css的开发体验让我非常舒服。之前我一直简单的以为electron只是张网页加个壳，和那些号称跨平台的运行在手机上的webapp是一个套路。直到我真的需要开发一个跨平台桌面应用的时候，我又认真的尝试了一下electron，我开始意识到：这才是我理想中的跨平台桌面应用开发的最终形态，它简直太优秀了。



# # 2

在使用electron期间，我顺便写了一个简单的todolist（便签）应用，用于学习和尝试；项目地址：https://github.com/xiajingren/xhznl-todo-list 界面参考了小黄条便签。它目前的功能还非常简单，但是包含了很多我使用electron时遇到问题，这也是electron新手都很可能遇到的，也算是一个技术总结吧；比如：

- electron无边框透明窗口/拖拽/置顶/闪烁问题

- electron软件开机自启动
- electron软件单实例运行
- electron窗口的鼠标穿透/部分穿透

- electron软件打包
- electron软件自动更新（GitHub）
- electron中使用本地数据库
- electron中数据导出为excel文件
- 等等......



以下是项目README：



# # 3

## **xhznl-todo-list**

:sparkles:一个使用 electron + vue + electron-builder 开发的跨平台 todo-list 桌面应用

## 相关技术

[electron 9.x](https://github.com/electron/electron)

[vue 2.x](https://github.com/vuejs/vue)

[vue-cli-plugin-electron-builder](https://github.com/nklayman/vue-cli-plugin-electron-builder)

[electron-builder](https://github.com/electron-userland/electron-builder)

[lowdb](https://github.com/typicode/lowdb)

[exceljs](https://github.com/exceljs/exceljs)

[dayjs](https://github.com/iamkun/dayjs)

[Vue.Draggable](https://github.com/SortableJS/Vue.Draggable)

......

## 功能预览

![todo list](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119112545824-2097582470.png)

![done list](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119112713397-538446110.png)

![基本操作](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119112735298-1299088400.gif)

![数据导出](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119115050149-1015064685.gif)

![鼠标穿透](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119133816637-1160146572.gif)

![macOS](https://img2020.cnblogs.com/blog/610959/202011/610959-20201119120029136-279684705.png)

## 步骤

```
npm install
npm run electron:serve

npm run electron:build
```

下载 releases：https://github.com/xiajingren/xhznl-todo-list/releases

## 规划

- [x] todo/done 基本功能
- [x] 本地数据库存储
- [x] 软件自动更新
- [x] 数据导出为 excel
- [x] 开机启动
- [x] 鼠标穿透
- [ ] 窗口贴边自动收起
- [ ] ......




# # 4

在使用electron期间确实也遇到很多坑，其中大部分都是来自于electron编译nodejs模块。后续我可能整理一个关于electron的系列分享，介绍 [xhznl-todo-list](https://github.com/xiajingren/xhznl-todo-list) 的实现细节，欢迎关注。

GitHub：https://github.com/xiajingren/xhznl-todo-list 



