title:  Stetho工具介绍
tag: 工具
---


[stetho](http://facebook.github.io/stetho/)是Facebook推出的安卓APP网络诊断和数据监控的工具，接入方便，功能强大，是开发者必备的好工具。

主要功能包括：

- 查看App的布局
- 网络请求抓包
- 数据库、sp文件查看
- 自定义dumpapp插件
- 对于JavaScript的支持

无需root，只要能通过adb连接设备，操作方便。

## 接入方法

### gradle配置
因为目前我们的项目中已经集成了okhttp，只需要在build.gradle添加如下两行配置

```groovy
	dependencies {
    //...

    compile 'com.facebook.stetho:stetho-js-rhino:1.3.1'
    compile 'com.facebook.stetho:stetho-okhttp3:1.4.2'
	}

```
### 初始化
在Application类中完成初始化
```java
private void MyApplicationCreate() {
		// ...
     Stetho.initializeWithDefaults(mContext);
  }
```

## 使用功能

1. adb方式连接到设备
2. 运行debug模式的app
3. 在Chrome浏览器地址栏中输入chrome://inspect
4. 选择需要inspect的应用进程
![](http://ww1.sinaimg.cn/large/8b331ee1gw1fbg4wwn5ftj21aw0qo7az.jpg)

### 查看App的布局
![](http://ww1.sinaimg.cn/large/8b331ee1gw1fbfhku1ltsj21kw0s5tt8.jpg)

### 网络诊断
给OkHttpClient添加拦截器。
```java
new OkHttpClient.Builder()
    .addNetworkInterceptor(new StethoInterceptor())
    .build();
```

主要功能有包括下载图片的预览，JSON数据查看，网络请求内容和返回内容
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbfhpn53jjj21kw0rjk3j.jpg)

### 数据库、sp文件查看

![](http://ww4.sinaimg.cn/large/8b331ee1gw1fbfhrmo7blj21kw0rvgx0.jpg)


![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbfhsqtl35j21kw0rq7fi.jpg)

### 自定义dumpapp插件

```java
Stetho.initialize(Stetho.newInitializerBuilder(context)
        .enableDumpapp(new DumperPluginsProvider() {
          @Override
          public Iterable<DumperPlugin> get() {
            return new Stetho.DefaultDumperPluginsBuilder(context)
                .provide(new HelloWorldDumperPlugin())
                .provide(new APODDumperPlugin(context.getContentResolver()))
                .finish();
          }
        })
        .enableWebKitInspector(new ExtInspectorModulesProvider(context))
        .build());
```

其中HelloWorldDumperPlugin和APODDumperPlugin是自定义的插件，具体内容可以参考[Stetho提供的sample程序](https://github.com/facebook/stetho/tree/master/stetho-sample)
运行dumpapp脚本后以达到与app交互通信的效果。
```bash
$ ./dumpapp -l
apod
crash
files
hello
hprof
prefs

```

#### 原理简介
其中dumpapp是一个python脚本，通信的方式使用的是android提供的[smartsocket接口](https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt)
```javascript
--- smartsockets -------------------------------------------------------
Port 5037 is used for smart sockets which allow a client on the host
side to request access to a service in the host adb daemon or in the
remote (device) daemon.  The service is requested by ascii name,
preceeded by a 4 digit hex length.  Upon successful connection an
"OKAY" response is sent, otherwise a "FAIL" message is returned.  Once
connected the client is talking to that (remote or local) service.
client: <hex4> <service-name>
server: "OKAY"
client: <hex4> <service-name>
server: "FAIL" <hex4> <reason>

```
使用PyCharm[对Python进行断点调试](http://stackoverflow.com/questions/27952331/debugging-with-pycharm-terminal-arguments)：
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbfian208yj21kw0arwia.jpg)
这段脚本的功能就是通过读取`/proc/net/unix`文件去找app设置的socket
![](http://ww2.sinaimg.cn/large/8b331ee1gw1fbfid70oipj21gs044t9m.jpg)
1. 扫描android所有提供socket功能的设备，找到steho的devtools_remote
2. 建立与第一步找到的进程socket，然后通过smartsocket进行通信。
3. 设备上的app相当于一个服务器，脚本是客户端对它进行访问

后缀为_devtools_remote的socket是android留给chrome的后门。
```javascript
// Note that _devtools_remote is a magic suffix understood by Chrome which //causes the discovery process to begin.
```
详细内容可以看这篇[官方文档](https://developer.chrome.com/devtools/docs/remote-debugging-legacy)
这篇文档提供的例子是在命令行中输入下面的命令，就能在电脑上看到手机chrome中的内容了：
`adb forward tcp:9222 localabstract:chrome_devtools_remote`

![](http://ww2.sinaimg.cn/large/8b331ee1gw1fbfijp8jqfj21320lo0tp.jpg)

打开的chrome-devtool其实是一个websocket连接。
![](http://ww4.sinaimg.cn/large/8b331ee1gw1fbfipebrd2j21kw069766.jpg)
```java
private void handlePageList(LightHttpResponse response)
    throws JSONException {
  if (mPageListResponse == null) {
    JSONArray reply = new JSONArray();
    JSONObject page = new JSONObject();
    page.put("type", "app");
    page.put("title", makeTitle());
    page.put("id", PAGE_ID);
    page.put("description", "");

    page.put("webSocketDebuggerUrl", "ws://" + mInspectorPath);
    Uri chromeFrontendUrl = new Uri.Builder()
        .scheme("http")
        .authority("chrome-devtools-frontend.appspot.com")
        .appendEncodedPath("serve_rev")
        .appendEncodedPath(WEBKIT_REV)
        .appendEncodedPath("devtools.html")
        .appendQueryParameter("ws", mInspectorPath)
        .build();
    page.put("devtoolsFrontendUrl", chromeFrontendUrl.toString());

    reply.put(page);
    mPageListResponse = LightHttpBody.create(reply.toString(), "application/json");
  }
  setSuccessfulResponse(response, mPageListResponse);
}
```

在android上的服务端socket写法,
详见LocalSocketServer类的listenOnAddress方法
```java
  private void listenOnAddress(String address) throws IOException {
    mServerSocket = bindToSocket(address);
    LogUtil.i("Listening on @" + address);

    while (!Thread.interrupted()) {
      try {
        // Use previously accepted socket the first time around, otherwise wait to accept another.
        LocalSocket socket = mServerSocket.accept();

        // Start worker thread
        Thread t = new WorkerThread(socket, mSocketHandler);
        t.setName(
            WORKER_THREAD_NAME_PREFIX +
            "-" + mFriendlyName +
            "-" + mThreadId.incrementAndGet());
        t.setDaemon(true);
        t.start();
      } catch (SocketException se) {
        // ignore exception if interrupting the thread
        if (Thread.interrupted()) {
          break;
        }
        LogUtil.w(se, "I/O error");
      } catch (InterruptedIOException ex) {
        break;
      } catch (IOException e) {
        LogUtil.w(e, "I/O error initialising connection thread");
        break;
      }
    }

    LogUtil.i("Server shutdown on @" + address);
  }
```

### 对于JavaScript的支持
Chrome开发者工具原生支持JavaScript，所以Stetho也提供了JavaScript的支持。
通过在console中输入如下代码可以让设备app弹出一个toast
```javascript
importPackage(android.widget);
importPackage(android.os);
var handler = new Handler(Looper.getMainLooper());
handler.post(function() { Toast.makeText(context, "Hello from JavaScript", Toast.LENGTH_LONG).show() });
```
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbfixpdt6hj21kw0rmgv8.jpg)
更多玩法见[Rhino on Stetho
](https://github.com/facebook/stetho/tree/master/stetho-js-rhino)


----------
## 相关链接
http://facebook.github.io/stetho/
https://github.com/facebook/stetho/tree/master/stetho-js-rhino
https://www.figotan.org/2016/04/18/using-stetho-to-diagnose-data-on-android/
https://developer.chrome.com/devtools/docs/remote-debugging-legacy
https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt