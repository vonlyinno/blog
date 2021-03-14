# H5与APP通信

以下内容纯学习总结文，参考文章都列在了附录。

# 行业方案

Hybrid App，俗称混合应用，即混合了 Native技术 与 Web技术 进行开发的移动应用。现在比较流行的混合方案主要有三种，主要是在UI渲染机制上的不同：

- 1、**基于 WebView UI 的基础方案**，市面上大部分主流 App 都有采用，例如微信JS-SDK，通过 JSBridge 完成 H5 与 Native 的双向通讯，从而赋予H5一定程度的原生能力。
- 2、**基于 Native UI 的方案**，例如 React-Native、Weex。在赋予 H5 原生API能力的基础上，进一步通过 JSBridge 将js解析成的虚拟节点树(Virtual DOM)传递到 Native 并使用原生渲染。
- 3、近期比较流行的**小程序方案**，通过更加定制化的 JSBridge，并使用双 WebView 双线程的模式隔离了JS逻辑与UI渲染，形成了特殊的开发模式，加强了 H5 与 Native 混合程度，提高了页面性能及开发体验。

# 原理

Hybrid App的本质，其实是在原生的 App 中，使用 **WebView** 作为容器直接承载 Web页面。

## webview

> webview，一个基于webkit的引擎 ，用来展示网页的 view 组件，该组件是你运行自己的浏览器或者在你的线程中展示线上内容的基础。使用 webkit 渲染引擎来展示，并且支持前进后退等基于浏览历史，放大缩小，等更多功能。
简单来说 WebView 是手机中内置了一款高性能 webkit 内核浏览器，在 SDK 中封装的一个组件。不过没有提供地址栏和导航栏，只是单纯的展示一个网页界面。

### [webview初始化](https://tech.meituan.com/2017/06/09/webviewperf.html)

当App首次打开时，默认是并不初始化浏览器内核的；只有当创建WebView实例的时候，才会创建WebView的基础框架。

所以与浏览器不同，**App中打开WebView的第一步并不是建立连接，而是启动浏览器内核**。

![20210314153604](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/20210314153604.png)

### 有哪几种？

**IOS**

- UIWebView。在ios中有自己的浏览器组件，他就是UIWebView。UIWebView是iOS上对WebKit的封装，WebKit是渲染引擎，UIWebView是渲染引擎和JS引擎的组合。
- WKWebView。从iOS8起，Apple推出了wkwebview，Safari默认使用wkwebview。目的是给出一个新的高性能的 Web View 解决方案，摆脱过去 UIWebView 的老旧笨重特别是内存占用量巨大的问题。（节省内存、滚动时懒加载的图片也可以实时渲染）

**Android**

- 系统webview。
    - 系统webview，是默认的webview，及Google的Android system webview，它自带于手机rom中，所有依赖系统webview的应用都调用这个webview。
        - 在Android4.4以前，webview是Android webkit 浏览器内核，很多HTML5标准语法不支持。Android4.4起，变成了chromium内核。
    - 从HBuilderX 2.5.3起，支持使用腾讯的x5内核。

        **[使用X5内核能解决什么问题：](https://ask.dcloud.net.cn/article/36806)**

        1. x5适配了rom的自定义主题字体，与原生字体保持一致。不会出现一个界面部分字体为原生字体、部分字体为webview字体的问题。之前系统webview在部分手机上不能适配rom自定义主题的字体。
        2. 系统的webview有浏览器兼容问题，低端Android的webview有很多新语法都不支持。使用x5可以拉齐webview内核。对于5+App和wap2app，可以全部拉齐。
        3. x5内核自带的video实现强于html的video，支持视频格式更多。
        4. 远程web页面防劫持是x5内核的一大亮点
- X5内核

## 通信

### native调用JS   [详细android code](https://blog.csdn.net/carson_ho/article/details/64904691/)

- **API注入**，原理其实就是 Native 获取 JavaScript环境上下文，并直接在上面挂载对象或者方法，使 js 可以直接调用，Android 与 IOS 分别拥有对应的挂载方式。
- **WebView 中的 prompt/console/alert 拦截**，通常使用 prompt，因为这个方法在前端中使用频率低，比较不会出现冲突；
- WebView **URL Scheme 跳转拦截**
    - 在 WebView 中发出的网络请求，客户端都能进行监听和捕获
    - 协议制定：xxcommand://xxxx?param1=1&param2=2
    - 拦截协议：客户端可以通过 API 对 WebView 发出的请求进行拦截，当解析到请求 URL 头为制定的协议时，便不发起对应的资源请求，而是解析参数，并进行相关功能或者方法的调用，完成协议功能的映射。
        - IOS上: shouldStartLoadWithRequest
        - Android: shouldOverrideUrlLoading
    - 协议回调：发送请求，这属于一个异步的过程，类似于JS的事件系统，这里我们会用到 window.addEventListener 和 window.dispatchEvent这两个基础API。
    - 参数传递方式：WebView 对 URL 会有长度的限制，可以基于 Native 可以直接调用 JS 方法并直接获取函数的返回值。我们只需要对每条协议标记一个唯一标识，并把参数存入参数池中，到时客户端再通过该唯一标识从参数池中获取对应的参数即可。

### JS调用native

- IOS: stringByEvaluatingJavaScriptFromString

    ```jsx
    // Swift
    webview.stringByEvaluatingJavaScriptFromString("alert('NativeCall')")
    ```

- Android: loadUrl (4.4-)

    ```jsx
    // 调用js中的JSBridge.trigger方法
    // 该方法的弊端是无法获取函数返回值；
    webView.loadUrl("javascript:JSBridge.trigger('NativeCall')")
    ```

- Android: evaluateJavascript (4.4+)

    ```jsx
    // 4.4+后使用该方法便可调用并获取函数返回值；该方法的执行不会使页面刷新
    mWebView.evaluateJavascript（"javascript:JSBridge.trigger('NativeCall')",      new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            //此处为 js 返回的结果
        }
    });
    ```

## JSBridge

JSBridge 简单来讲，主要是 给 JavaScript 提供调用 Native 功能的接口，核心是 构建 Native 和非Native 间消息通信的通道，而且是 双向通信的通道。

分为两个部分：

- JS部分(bridge): 在JS环境中注入 bridge 的实现代码，包含了协议的拼装/发送/参数池/回调池等一些基础功能。
- Native部分(SDK)：在客户端中 bridge 的功能映射代码，实现了URL拦截与解析/环境信息的注入/通用功能映射等功能。

这里的做法是，将这两部分一起封装成一个 Native SDK，由客户端统一引入。由于客户端的注入行为属于一个附加的异步行为，从H5方很难去捕捉准确的完成时机，因此这里需要通过客户端监听页面完成后，基于上面的回调机制通知 H5端，页面中即可通过window.addEventListener('bridgeReady', e => {})  进行初始化。

# 最后

想了一下我们的APP目前在用的方式。从H5端来看：

1. JavaScript 调用 Native 是使用 注入 API 的方式，Native会在全局对象上注入JSBridge，并且在注入完成后通知H5以及ready。JSBridge上也比较简单，常用的就是invoke和on，类似于事件机制的emit和on。二具体的参数则是需要执行的时间名、参数、callback。
2. 伪协议，通过约定好的协议字段，可以通过1中的openUrl的事件，去拉起伪协议。

# 附录

[https://github.com/debingfeng/blog/issues/6](https://github.com/debingfeng/blog/issues/6)

[https://tech.meituan.com/2017/06/09/webviewperf.html](https://tech.meituan.com/2017/06/09/webviewperf.html)

[https://zhuanlan.zhihu.com/p/73111013](https://zhuanlan.zhihu.com/p/73111013)

[https://ask.dcloud.net.cn/article/1318](https://ask.dcloud.net.cn/article/1318)