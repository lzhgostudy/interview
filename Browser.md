# 1. 浏览器加载页面的流程
![浏览器加载页面流程.png](https://upload-images.jianshu.io/upload_images/16548108-68f47fd2777d92e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 首先，开源浏览器一般以8k每块下载html页面。然后解析页面生成DOM树，
- 遇到css标签或JS脚本标签就新起线程去下载他们，并继续构建DOM。
- 下载完后解析CSS为CSS规则树，浏览器结合CSS规则树和DOM树生成Render Tree。

> 注意：构建CSS Object Model（CSSOM)会阻塞JavaScript的执行。JavaScript的执行也会阻塞DOM的构建。JavaScript下载后可以通过DOM API修改DOM，通过CSSOM API修改样式作用域Render Tree。每次修改会造成Render Tree的重新布局和重绘。只要修改DOM或修改了元素的形状或大小，就会触发Reflow，单纯修改元素的颜色只需Repaint一下（调用操作系统Native GUI的API绘制）。


# 2. 介绍下缓存

所有的性能优化中，缓存是最重要也是最直接有效的。
缓存分为强缓存和协商缓存

![流程图](https://user-gold-cdn.xitu.io/2020/2/27/17085a970e66cf41?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

缓存机制相关的字段都是在请求和响应上

## 强缓存

在缓存有效期内，客户端直接读取本地资源，强缓存返回的状态码是 200

## Expires

在`http1.0`中使用，表示资源失效的具体时间点`Expires: Sat, 09 Jun 2018 08:13:56 GMT`，若是访问器和本地时间不一致，可能就会出现问题，在现在http1.1中换成了`max-age`，为了兼容也可以加上。