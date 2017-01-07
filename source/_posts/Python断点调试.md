
---
title: Python断点调试
tag: 工具 python
---


很多项目是用python写构建脚本的，比如微信最近开源的终端跨平台组件 [Mars](https://github.com/Tencent/mars) 
本文将以mars为例简单介绍下如何用[PyCharm](https://www.jetbrains.com/pycharm/)对python进行断点调试。

---

### 导入代码
open整个mars项目，切换合适的python版本，mars需要python2.7版本。
![](http://ww1.sinaimg.cn/large/8b331ee1gw1fbifkuh496j20h81f0q9j.jpg)
![](http://ww2.sinaimg.cn/large/8b331ee1gw1fbifke5anpj20xm0du0xi.jpg)

### 打断点
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbifnm18q9j20gk04gt94.jpg)

### Debug it
![](http://ww1.sinaimg.cn/large/8b331ee1gw1fbifo65ncyj20po0v6wm7.jpg)

当代码中需要input时，切换到Console窗口输入
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbifq7bsxlj20nm0e4jvb.jpg)

![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbifqp5rcaj211q0lutf5.jpg)

用PyCharm调试跟Android Studio一样，毕竟都是一家公司的产品。

---