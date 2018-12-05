---

layout:     post

title:      "如何设计一个优雅健壮的Android WebView？（下）"

subtitle:   "基于考拉电商平台的WebView实践"

date:       2018-02-27

author:     "黎星"

header-img: "img/bg9.jpg"

tags:

    Android

    WebView
    
    301/302
    
    Redirect
    
    POST
    
---

# 前言

在上文[《如何设计一个优雅健壮的Android WebView？（上）》](https://kaolamobile.github.io/2018/02/16/design-an-elegant-and-powerful-android-webview-part-one/)中，笔者分析了国内WebView的现状，以及在WebView开发过程中所遇到的一些坑。在踩坑的基础上，本文着重介绍WebView在开发过程中所需要注意的问题，这些问题大部分在网上找不到标准答案，但却是WebView开发过程中几乎都会遇到的。此外还会浅谈WebView优化，旨在给用户带来更好的WebView体验。

# WebView实战操作

WebView在使用过程中会遇到各种各样的问题，下面针对几个在生产环境中使用的WebView可能出现的问题进行探讨。

## WebView初始化

也许大部分的开发者针对要打开一个网页这一个Action，会停留在下面这段代码：

```
WebView webview = new WebView(context);
webview.loadUrl(url);
```

这应该是打开一个正常网页最简短的代码了。但大多数情况下，我们需要做一些额外的配置，例如缩放支持、Cookie管理、密码存储、DOM存储等，这些配置大部分在`WebSettings`里，具体配置的内容在上文中已有提及，本文不再具体讲解。

接下来，试想如果访问的网页返回的请求是30X，如使用http访问百度的链接（<http://www.baidu.com>），那么这时候页面就是空白一片，GG了。为什么呢？因为WebView只加载了第一个网页，接下来的事情就不管了。为了解决这个问题，我们需要一个`WebViewClient`让系统帮我们处理重定向问题。

```
webview.setWebViewClient(new WebViewClient());
```

除了处理重定向，我们还可以覆写`WebViewClient`中的方法，方法有：

```
public boolean shouldOverrideUrlLoading(WebView view, String url) 
public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request)
public void onPageStarted(WebView view, String url, Bitmap favicon) 
public void onPageFinished(WebView view, String url) 
public void onLoadResource(WebView view, String url) 
public void onPageCommitVisible(WebView view, String url) 
public WebResourceResponse shouldInterceptRequest(WebView view, String url) 
public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) 
public void onTooManyRedirects(WebView view, Message cancelMsg, Message continueMsg) 
public void onReceivedError(WebView view, int errorCode, String description, String failingUrl) 
public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) 
public void onReceivedHttpError(WebView view, WebResourceRequest request, WebResourceResponse errorResponse) 
public void onFormResubmission(WebView view, Message dontResend, Message resend) 
public void doUpdateVisitedHistory(WebView view, String url, boolean isReload) 
public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) 
public void onReceivedClientCertRequest(WebView view, ClientCertRequest request) 
public void onReceivedHttpAuthRequest(WebView view, HttpAuthHandler handler, String host, String realm) 
public boolean shouldOverrideKeyEvent(WebView view, KeyEvent event) 
public void onUnhandledKeyEvent(WebView view, KeyEvent event) 
public void onScaleChanged(WebView view, float oldScale, float newScale) 
public void onReceivedLoginRequest(WebView view, String realm, String account, String args) 
public boolean onRenderProcessGone(WebView view, RenderProcessGoneDetail detail) 
```

这些方法具体介绍可以参考文章[《WebView使用详解（二）——WebViewClient与常用事件监听》](http://blog.csdn.net/harvic880925/article/details/51523983)。有几个方法是有必要覆写来处理一些客户端逻辑的，后面遇到会详细介绍。

另外，WebView的标题不是一成不变的，加载的网页不一样，标题也不一样。在WebView中，加载的网页的标题会回调`WebChromeClient.onReceivedTitle()`方法，给开发者设置标题。因此，设置一个`WebChromeClient`也是有必要的。

```
webview.setWebChromeClient(new WebChromeClient());
```

同样，我们还可以覆写`WebChromeClient`中的方法，方法有：

```
public void onProgressChanged(WebView view, int newProgress)
public void onReceivedTitle(WebView view, String title)
public void onReceivedIcon(WebView view, Bitmap icon)
public void onReceivedTouchIconUrl(WebView view, String url, boolean precomposed)
public void onShowCustomView(View view, int requestedOrientation, CustomViewCallback callback)
public void onHideCustomView()
public boolean onCreateWindow(WebView view, boolean isDialog, boolean isUserGesture, Message resultMsg)
public void onRequestFocus(WebView view)
public void onCloseWindow(WebView window)
public boolean onJsAlert(WebView view, String url, String message, JsResult result)
public boolean onJsConfirm(WebView view, String url, String message, JsResult result)
public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result)
public void onExceededDatabaseQuota(String url, String databaseIdentifier, long quota, long estimatedDatabaseSize, long totalQuota, WebStorage.QuotaUpdater quotaUpdater)
public void onReachedMaxAppCacheSize(long requiredStorage, long quota, WebStorage.QuotaUpdater quotaUpdater)
public void onGeolocationPermissionsShowPrompt(String origin, GeolocationPermissions.Callback callback)
public void onGeolocationPermissionsHidePrompt()
public void onPermissionRequest(PermissionRequest request)
public void onPermissionRequestCanceled(PermissionRequest request)
public boolean onJsTimeout()
public void onConsoleMessage(String message, int lineNumber, String sourceID)
public boolean onConsoleMessage(ConsoleMessage consoleMessage)
public Bitmap getDefaultVideoPoster()
public void getVisitedHistory(ValueCallback<String[]> callback)
public boolean onShowFileChooser(WebView webView, ValueCallback<Uri[]> filePathCallback, FileChooserParams fileChooserParams)
public void openFileChooser(ValueCallback<Uri> uploadFile, String acceptType, String capture)
public void setupAutoFill(Message msg)
```

这些方法具体介绍可以参考文章[《WebView使用详解（三）——WebChromeClient与LoadData补充》](http://blog.csdn.net/harvic880925/article/details/51583253)。除了接收标题以外，进度条的改变，WebView请求本地文件、请求地理位置权限等，都是通过`WebChromeClient`的回调实现的。

在初始化阶段，如果启用了`Javascript`，那么需要移除相关的安全漏洞，这在上一篇文章中也有所提及。最后，在考拉`KaolaWebView.init()`方法中，执行了如下操作：

```
protected void init() {
    mContext = getContext();
    mWebJsManager = new WebJsManager();	// 初始化Js管理器
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    	// 根据本地调试开关打开Chrome调试
       WebView.setWebContentsDebuggingEnabled(WebSwitchManager.isDebugEnable());
    }
    // WebSettings配置
    WebViewSettings.setDefaultWebSettings(this);
    // 获取deviceId列表，安全相关
    WebViewHelper.requestNeedDeviceIdUrlList(null);
    // 设置下载的监听器
    setDownloadListener(this);
    // 前端控制回退栈，默认回退1。
    mBackStep = 1;
    // 重定向保护，防止空白页
    mRedirectProtected = true;
    // 截图使用
    setDrawingCacheEnabled(true);
    // 初始化具体的Jsbridge类
    enableJsApiInternal();
    // 初始化WebCache，用于加载静态资源
    initWebCache();
    // 初始化WebChromeClient，覆写其中一部分方法
    super.setWebChromeClient(mChromeClient);
    // 初始化WebViewClient，覆写其中一部分方法
    super.setWebViewClient(mWebViewClient);
}
```

## WebView加载一个网页的过程中该做些什么？

如果说加载一个网页只需要调用`WebView.loadUrl(url)`这么简单，那肯定没程序员啥事儿了。往往事情没有这么简单。加载网页是一个复杂的过程，在这个过程中，我们可能需要执行一些操作，包括：

1. 加载网页前，重置WebView状态以及与业务绑定的变量状态。WebView状态包括重定向状态(`mTouchByUser`)、前端控制的回退栈(`mBackStep`)等，业务状态包括进度条、当前页的分享内容、分享按钮的显示隐藏等。
2. 加载网页前，根据不同的域拼接本地客户端的参数，包括基本的机型信息、版本信息、登录信息以及埋点使用的Refer信息等，有时候涉及交易、财产等还需要做额外的配置。
3. 开始执行页面加载操作时，会回调`WebViewClient.onPageStarted(webview, url, favicon)`。在此方法中，可以重置重定向保护的变量(`mRedirectProtected`)，当然也可以在页面加载前重置，由于历史遗留代码问题，此处尚未省去优化。
4. 加载页面的过程中，WebView会回调几个方法。
	* `WebChromeClient.onReceivedTitle(webview, title)`，用来设置标题。需要注意的是，在部分Android系统版本中可能会回调多次这个方法，而且有时候回调的title是一个url，客户端可以针对这种情况进行特殊处理，避免在标题栏显示不必要的链接。
	* `WebChromeClient.onProgressChanged(webview, progress)`，根据这个回调，可以控制进度条的进度（包括显示与隐藏）。一般情况下，想要达到100%的进度需要的时间较长（特别是首次加载），用户长时间等待进度条不消失必定会感到焦虑，影响体验。其实当progress达到80的时候，加载出来的页面已经基本可用了。事实上，国内厂商大部分都会提前隐藏进度条，让用户以为网页加载很快。
	* `WebViewClient.shouldInterceptRequest(webview, request)`，无论是普通的页面请求(使用GET/POST)，还是页面中的异步请求，或者页面中的资源请求，都会回调这个方法，给开发一次拦截请求的机会。在这个方法中，我们可以进行静态资源的拦截并使用缓存数据代替，也可以拦截页面，使用自己的网络框架来请求数据。包括后面介绍的WebView免流方案，也和此方法有关。
	* `WebViewClient.shouldOverrideUrlLoading(webview, request)`，如果遇到了重定向，或者点击了页面中的a标签实现页面跳转，那么会回调这个方法。可以说这个是WebView里面最重要的回调之一，后面`WebView与Native页面交互`一节将会详细介绍这个方法。
	* `WebViewClient.onReceived**Error(webview, handler, error)`，加载页面的过程中发生了错误，会回调这个方法。主要是http错误以及ssl错误。在这两个回调中，我们可以进行异常上报，监控异常页面、过期页面，及时反馈给运营或前端修改。在处理ssl错误时，遇到不信任的证书可以进行特殊处理，例如对域名进行判断，针对自己公司的域名“放行”，防止进入丑陋的错误证书页面。也可以与Chrome一样，弹出ssl证书疑问弹窗，给用户选择的余地。
5. 页面加载结束后，会回调`WebViewClient.onPageFinished(webview, url)`。这时候可以根据回退栈的情况判断是否显示关闭WebView按钮。通过`mActivityWeb.canGoBackOrForward(-1)`判断是否可以回退。

## WebView与JavaScript交互——JsBridge

Android WebView与JavaScript的通信方案，目前业界已经有比较成熟的方案了。常见的有[lzyzsd/JsBridge](https://github.com/lzyzsd/JsBridge)、[pengwei1024/JsBridge](https://github.com/pengwei1024/JsBridge)等，详见[此链接](<https://github.com/search?l=Java&q=JsBridge&type=Repositories&utf8=%E2%9C%93>)。

通常，Java调用js方法有两种：

* `WebView.loadUrl("javascript:" + javascript);`
* `WebView.evaluateJavascript(javascript, callbacck);`

第一种方式已经不推荐使用了，第二种方式不仅更方便，也提供了结果的回调，但仅支持API 19以后的系统。

js调用Java的方法有四种，分别是：

* JavascriptInterface
* WebViewClient.shouldOverrideUrlLoading()
* WebChromeClient.onConsoleMessage()
* WebChromeClient.onJsPrompt()

这四种方式不再一一介绍，掘金上的[这篇文章](https://juejin.im/entry/573534f82e958a0069b27646)已经讲得很详细。

下面来介绍一下考拉使用的JsBridge方案。Java调用js方法不必多说，根据Android系统版本不同分别调用第一个方法和第二个方法。在js调用Java方法上，考拉使用的是第四种方案，即侵入`WebChromeClient.onJsPrompt(webview, url, message, defaultValue, result)`实现通信。

```
@Override
public boolean onJsPrompt(WebView view, String url, String message, String defaultValue,
        JsPromptResult result) {
    if (!ActivityUtils.activityIsAlive(mContext)) {//页面关闭后，直接返回
        try {
            result.cancel();
        } catch (Exception ignored) {
        }
        return true;
    }
    if (mJsApi != null && mJsApi.hijackJsPrompt(message)) {
        result.confirm();
        return true;
    }
    return super.onJsPrompt(view, url, message, defaultValue, result);
}
```

`onJsPrompt`方法最终是在主线程中回调，判断一下WebView所在容器的生命周期是有必要的。js与Java的方法调用主要在`mJsApi.hijackJsPrompt(message)`中。

```
public boolean hijackJsPrompt(String message) {
    if (TextUtils.isEmpty(message)) {
        return false;
    }

    boolean handle = message.startsWith(YIXIN_JSBRIDGE);

    if (handle) {
        call(message);
    }

    return handle;
}
```

首先判断该信息是否应该拦截，如果允许拦截的话，则取出js传过来的方法和参数，通过Handler把消息抛给业务层处理。

```
private void call(String message) {
    // PREFIX
    message = message.substring(KaolaJsApi.YIXIN_JSBRIDGE.length());
    // BASE64
    message = new String(Base64.decode(message));

    JSONObject json = JSONObject.parseObject(message);
    String method = json.getString("method");
    String params = json.getString("params");
    String version = json.getString("jsonrpc");

    if ("2.0".equals(version)) {
        int id = json.containsKey("id") ? json.getIntValue("id") : -1;

        call(id, method, params);
    }

    callJS("window.jsonRPC.invokeFinish()");
}

private void call(int id, String method, String params) {
	Message msg = Message.obtain();
	msg.what = MsgWhat.JSCall;
	msg.obj = new KaolaJSMessage(id, method, params);
	// 通过handler把消息发出去，待接收方处理。
	if (handler != null) {
	    handler.sendMessage(msg);
	}
}
```

jsbridge中，实现了一个存储jsbridge指令的队列CommandQueue，每次需要调用jsbridge时，只需要入队即可。

```
function CommandQueue() {
    this.backQueue = [];
    this.queue = [];
};

CommandQueue.prototype.dequeue = function() {
    if(this.queue.length <=0 && this.backQueue.length > 0) {
        this.queue = this.backQueue.reverse();
        this.backQueue = [];
    }
    return this.queue.pop();
};

CommandQueue.prototype.enqueue = function(item) {
    this.backQueue.push(item);
};

Object.defineProperty(CommandQueue.prototype, 'length',
{get: function() {return this.queue.length + this.backQueue.length; }});

var commandQueue = new CommandQueue();

function _nativeExec(){
    var command = commandQueue.dequeue();
    if(command) {
        nativeReady = false;
        var jsoncommand = JSON.stringify(command);
        var _temp = prompt(jsoncommand,'');
        return true;
    } else {
        return false;
    }
}

```

上面的代码有所删减，若需要执行完整的jsbridge功能，还需要做一些额外的配置。例如告知前端这段js代码已经注入成功的标记。

## 什么时候注入js合适？

如果做过WebView开发，并且需要和js交互的同学，大部分都会认为js在`WebViewClient.onPageFinished()`方法中注入最合适，此时dom树已经构建完成，页面已经完全展现出来[^1][^2][^3]。但如果做过页面加载速度的测试，会发现`WebViewClient.onPageFinished()`方法通常需要等待很久才会回调（首次加载通常超过3s），这是因为WebView需要加载完一个网页里主文档和所有的资源才会回调这个方法。能不能在`WebViewClient.onPageStarted()`中注入呢？答案是不确定。经过测试，有些机型可以，有些机型不行。在`WebViewClient.onPageStarted()`中注入还有一个致命的问题——这个方法可能会回调多次，会造成js代码的多次注入。

另一方面，从7.0开始，WebView加载js方式发生了一些小改变，官方建议把js注入的时机放在页面开始加载之后。援引官方的文档[^4]：

> Javascript run before page load
>
> Starting with apps targeting Android 7.0, the Javascript context will be reset when a new page is loaded. Currently, the context is carried over for the first page loaded in a new WebView instance.
>
> Developers looking to inject Javascript into the WebView should execute the script after the page has started to load.

在[这篇文章](http://blog.csdn.net/leehong2005/article/details/11808557)中也提及了js注入的时机可以在多个回调里实现，包括：

* onLoadResource
* doUpdateVisitedHistory
* onPageStarted
* onPageFinished
* onReceivedTitle
* onProgressChanged

尽管文章作者已经做了测试证明以上时机注入是可行的，但他不能完全保证没有问题。事实也是，这些回调里有多个是会回调多次的，不能保证一次注入成功。

`WebViewClient.onPageStarted()`太早，`WebViewClient.onPageFinished()`又太迟，究竟有没有比较合适的注入时机呢？试试`WebViewClient.onProgressChanged()`？这个方法在dom树渲染的过程中会回调多次，每次都会告诉我们当前加载的进度。这不正是告诉我们页面已经开始加载了吗？考拉正是使用了`WebViewClient.onProgressChanged()`方法来注入js代码。

```
@Override
public void onProgressChanged(WebView view, int newProgress) {
    super.onProgressChanged(view, newProgress);
    if (null != mIWebViewClient) {
        mIWebViewClient.onProgressChanged(view, newProgress);
    }

    if (mCallProgressCallback && newProgress >= mProgressFinishThreshold) {
        DebugLog.d("WebView", "onProgressChanged: " + newProgress);
        mCallProgressCallback = false;
        // mJsApi不为null且允许注入js的情况下，开始注入js代码。
        if (mJsApi != null && WebJsManager.enableJs(view.getUrl())) {
            mJsApi.loadLocalJsCode();
        }
        if (mIWebViewClient != null) {
            mIWebViewClient.onPageFinished(view, newProgress);
        }
    }
}

```

可以看到，我们使用了`mProgressFinishThreshold`这个变量控制注入时机，这与前面提及的`当progress达到80的时候，加载出来的页面已经基本可用了`是相呼应的。

> 达到80%很容易，达到100%却很难。

正是因为这个原因，页面的进度加载到80%的时候，实际上dom树已经渲染得差不多了，表明WebView已经解析了\<html\>标签，这时候注入一定是成功的。在`WebViewClient.onProgressChanged()`实现js注入有几个需要注意的地方：

1. 上文提到的多次注入控制，我们使用了mCallProgressCallback变量控制
2. 重新加载一个URL之前，需要重置mCallProgressCallback，让重新加载后的页面再次注入js
3. 注入的进度阈值可以自由定制，理论上10%-100%都是合理的，我们使用了80%。

## H5页面、Weex页面与Native页面交互——KaolaRouter

H5页面、Weex页面与Native页面的交互是通过URL拦截实现的。在WebView中，`WebViewClient.shouldOverrideUrlLoading()`方法能够获取到当前加载的URL，然后把URL传递给考拉路由框架，便可以判断URL是否能够跳转到其他非H5页面，考拉路由框架在[《考拉Android客户端路由总线设计》](http://iluhcm.com/2017/07/12/design-of-router-using-in-android/)一文中有详细介绍，但当时未引入Weex页面，关于如何整合三者的通信，后续文章会有详细介绍。

在`WebViewClient.shouldOverrideUrlLoading()`中，根据URL类型做了判断：

```
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    if (StringUtils.isNotBlank(url) && url.equals("about:blank")) {   //js调用reload刷新页面时候，个别机型跳到空页面问题修复
        url = getUrl();
    }
    url = WebViewUtils.removeBlank(url);
    mCallProgressCallback = true;
    //允许启动第三方应用客户端
    if (WebViewUtils.canHandleUrl(url)) {
        boolean handleByCaller = false;
        // 如果不是用户触发的操作，就没有必要交给上层处理了，直接走url拦截规则。
        if (null != mIWebViewClient && isTouchByUser()) {
        	// 先交给业务层拦截处理
            handleByCaller = mIWebViewClient.shouldOverrideUrlLoading(view, url);
        }
        if (!handleByCaller) {
        	// 业务层不拦截，走通用路由总线规则
            handleByCaller = handleOverrideUrl(url);
        }
        mRedirectProtected = true;
        return handleByCaller || super.shouldOverrideUrlLoading(view, url);
    } else {
        try {
            notifyBeforeLoadUrl(url);
            // https://sumile.cn/archives/1223.html
            Intent intent = Intent.parseUri(url, Intent.URI_INTENT_SCHEME);
            intent.addCategory(Intent.CATEGORY_BROWSABLE);
            intent.setComponent(null);
            intent.setSelector(null);
            mContext.startActivity(intent);
            if (!mIsBlankPageRedirect) {
                back();
            }
        } catch (Exception e) {
            ExceptionUtils.printExceptionTrace(e);
        }
        return true;
    }
}

private boolean handleOverrideUrl(final String url) {
   RouterResult result =  WebActivityRouter.startFromWeb(
            new IntentBuilder(mContext, url).setRouterActivityResult(new RouterActivityResult() {
                @Override
                public void onActivityFound() {
                    if (!mIsBlankPageRedirect) {
                    	// 路由拦截成功以后，为防止首次进入WebView产生白屏，因此加了保护机制
                        back();
                    }
                }

                @Override
                public void onActivityNotFound() {
                    
                }
            }));
    return result.isSuccess();
}
```

代码里写了注释，就不一一解释了。

## WebView下拉刷新实现

由于考拉使用的下拉刷新跟Material Design所使用的下拉刷新样式不一致，因此不能直接套用`SwipeRefreshLayout`。考拉使用的是一套改造过的[Android-PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh)，WebView的下拉刷新，正是继承自`PullToRefreshBase`来实现的。

```
/**
 * 创建者：Square Xu
 * 日期：2017/2/23
 * 功能模块：webview下拉刷新组件
 */
public class PullToRefreshWebView extends PullToRefreshBase<KaolaWebview> {
    public PullToRefreshWebView(Context context) {
        super(context);
    }

    public PullToRefreshWebView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public PullToRefreshWebView(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs);
    }

    public PullToRefreshWebView(Context context, Mode mode) {
        super(context, mode);
    }

    public PullToRefreshWebView(Context context, Mode mode, AnimationStyle animStyle) {
        super(context, mode, animStyle);
    }

    @Override
    public Orientation getPullToRefreshScrollDirection() {
        return Orientation.VERTICAL;
    }

    @Override
    protected KaolaWebview createRefreshableView(Context context, AttributeSet attrs) {
        KaolaWebview kaolaWebview = new KaolaWebview(context, attrs);
        //解决键盘弹起时候闪动的问题
        setGravity(AXIS_PULL_BEFORE);
        return kaolaWebview;
    }

    @Override
    protected boolean isReadyForPullEnd() {
        return false;
    }

    @Override
    protected boolean isReadyForPullStart() {
        return getRefreshableView().getScrollY() == 0;
    }
}
```

考拉使用了全屏模式实现沉浸式状态栏及[滑动返回](https://juejin.im/post/5a7919ce5188257a8929915a?from=timeline)，全屏模式和WebView下拉刷新相结合对键盘的弹起产生了闪动效果，经过组内大神的研究与多次调试（感谢@俊俊），发现`setGravity(AXIS_PULL_BEFORE)`能够解决闪动的问题。

## 如何处理加载错误(Http、SSL、Resource)？

对于WebView加载一个网页过程中所产生的错误回调，大致有三种：

* `WebViewClient.onReceivedHttpError(webView, webResourceRequest, webResourceResponse)`

任何HTTP请求产生的错误都会回调这个方法，包括主页面的html文档请求，iframe、图片等资源请求。在这个回调中，由于混杂了很多请求，不适合用来展示加载错误的页面，而适合做监控报警。当某个URL，或者某个资源收到大量报警时，说明页面或资源可能存在问题，这时候可以让相关运营及时响应修改。

* `WebViewClient.onReceivedSslError(webview, sslErrorHandler, sslError)`

任何HTTPS请求，遇到SSL错误时都会回调这个方法。比较正确的做法是让用户选择是否信任这个网站，这时候可以弹出信任选择框供用户选择（大部分正规浏览器是这么做的）。但人都是有私心的，何况是遇到自家的网站时。我们可以让一些特定的网站，不管其证书是否存在问题，都让用户信任它。在这一点上，分享一个小坑。考拉的SSL证书使用的是GeoTrust的`GeoTrust SSL CA - G3`，但是在某些机型上，打开考拉的页面都会提示证书错误。这时候就不得不使用“绝招”——让考拉的所有二级域都是可信任的。

```
@Override
public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) {
    if (UrlUtils.isKaolaHost(getUrl())) {
        handler.proceed();
    } else {
        super.onReceivedSslError(view, handler, error);
    }
}
```

* `WebViewClient.onReceivedError(webView, webResourceRequest, webResourceError)`

只有在主页面加载出现错误时，才会回调这个方法。这正是展示加载错误页面最合适的方法。然鹅，如果不管三七二十一直接展示错误页面的话，那很有可能会误判，给用户造成经常加载页面失败的错觉。由于不同的WebView实现可能不一样，所以我们首先需要排除几种误判的例子：

1. 加载失败的url跟WebView里的url不是同一个url，排除；
2. errorCode=-1，表明是ERROR_UNKNOWN的错误，为了保证不误判，排除
3. failingUrl=null&errorCode=-12，由于错误的url是空而不是ERROR_BAD_URL，排除

```
@Override
public void onReceivedError(WebView view, int errorCode, String description, String failingUrl) {
    super.onReceivedError(view, errorCode, description, failingUrl);

    // -12 == EventHandle.ERROR_BAD_URL, a hide return code inside android.net.http package
    if ((failingUrl != null && !failingUrl.equals(view.getUrl()) && !failingUrl.equals(view.getOriginalUrl())) /* not subresource error*/
            || (failingUrl == null && errorCode != -12) /*not bad url*/
            || errorCode == -1) { //当 errorCode = -1 且错误信息为 net::ERR_CACHE_MISS
        return;
    }

    if (!TextUtils.isEmpty(failingUrl)) {
        if (failingUrl.equals(view.getUrl())) {
            if (null != mIWebViewClient) {
                mIWebViewClient.onReceivedError(view);
            }
        }
    }
}
```

## 如何操作cookie？

Cookie默认情况下是不需要做处理的，如果有特殊需求，如针对某个页面设置额外的Cookie字段，可以通过代码来控制。下面列出几个有用的接口：

* 获取某个url下的所有Cookie：`CookieManager.getInstance().getCookie(url)`
* 判断WebView是否接受Cookie：`CookieManager.getInstance().acceptCookie()`
* 清除Session Cookie：`CookieManager.getInstance().removeSessionCookies(ValueCallback<Boolean> callback)`
* 清除所有Cookie：`CookieManager.getInstance().removeAllCookies(ValueCallback<Boolean> callback)`
* Cookie持久化：`CookieManager.getInstance().flush()`
* 针对某个主机设置Cookie：`CookieManager.getInstance().setCookie(String url, String value)`

## 如何调试WebView加载的页面？

在Android 4.4版本以后，可以使用Chrome开发者工具调试WebView内容[^5]。调试需要在代码里设置打开调试开关。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    WebView.setWebContentsDebuggingEnabled(true);
}
```

开启后，使用USB连接电脑，加载URL时，打开Chrome开发者工具，在浏览器输入

```
chrome://inspect
```

可以看到当前正在浏览的页面，点击`inspect`即可看到WebView加载的内容。

# WebView优化

除了上面提到的基本操作用来实现一个完整的浏览器功能外，WebView的加载速度、稳定性和安全性是可以进一步加强和提高的。下面从几个方面介绍一下WebView的优化方案，这些方案可能并不是都适用于所有场景，但思路是可以借鉴的。

## CandyWebCache

我们知道，在加载页面的过程中，js、css和图片资源占用了大量的流量，如果这些资源一开始就放在本地，或者只需要下载一次，后面重复利用，岂不美哉。尽管WebView也有几套缓存方案[^6]，但是总体而言效果不理想。基于自建缓存系统的思路，由网易杭研研发的CandyWebCache项目应运而生。CandyWebCache是一套支持离线缓存WebView资源并实时更新远程资源的解决方案，支持打母包时下载当前最新的资源文件集成到apk中，也支持在线实时更新资源。在WebView中，我们需要拦截`WebViewClient.shouldInterceptRequest()`方法，检测缓存是否存在，存在则直接取本地缓存数据，减少网络请求产生的流量。

```
@Override
public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
    if (WebSwitchManager.isWebCacheEnabled()) {
        try {
            WebResourceResponse resourceResponse = CandyWebCache.getsInstance().getResponse(view, request);
            return WebViewUtils.handleResponseHeader(resourceResponse);
        } catch (Throwable e) {
            ExceptionUtils.uploadCatchedException(e);
        }
    }
    return super.shouldInterceptRequest(view, request);
}

@Override
public WebResourceResponse shouldInterceptRequest(WebView view, String url) {
    if (WebSwitchManager.isWebCacheEnabled()) {
        try {
            WebResourceResponse resourceResponse = CandyWebCache.getsInstance().getResponse(view, url);
            return WebViewUtils.handleResponseHeader(resourceResponse);
        } catch (Throwable e) {
            ExceptionUtils.uploadCatchedException(e);
        }
    }
    return super.shouldInterceptRequest(view, url);
}
```

除了上述缓存方案外，腾讯的QQ会员团队也推出了开源的解决方案[VasSonic](https://github.com/Tencent/vassonic)，旨在提升H5的页面访问体验，但最好由前后端一起配合改造。这套整体的解决方案有很多借鉴意义，考拉也在学习中。

## Https、HttpDns、CDN

将http请求切换为https请求，可以降低运营商网络劫持（js劫持、图片劫持等）的概率，特别是使用了http2后，能够大幅提升web性能，减少网络延迟，减少请求的流量。

HttpDns，使用http协议向特定的DNS服务器进行域名解析请求，代替基于DNS协议向运营商的Local DNS发起解析请求，可以降低运营商DNS劫持带来的访问失败。目前在WebView上使用HttpDns尚存在一定问题，网上也没有较好的解决方案（[阿里云Android WebView+HttpDns最佳实践](https://help.aliyun.com/document_detail/60181.html)、[腾讯云HttpDns SDK接入](https://github.com/tencentyun/httpdns-android-sdk)、[webview接入HttpDNS实践](http://blog.csdn.net/u012455213/article/details/78281026)），因此还在调研中。

另一方面，可以把静态资源部署到多路CDN，直接通过CDN地址访问，减少网络延迟，多路CDN保障单个CDN大面积节点访问失败时可切换到备用的CDN上。

## WebView独立进程

WebView实例在Android7.0系统以后，已经可以选择运行在一个独立进程上[^7]；8.0以后默认就是运行在独立的沙盒进程中[^8]，未来Google也在朝这个方向发展，具体的WebView历史可以参考上一篇文章[《如何设计一个优雅健壮的Android WebView？（上）》](https://kaolamobile.github.io/2018/02/16/design-an-elegant-and-powerful-android-webview-part-one/)第一小节。

Android7.0系统以后，WebView相对来说是比较稳定的，无论承载WebView的容器是否在主进程，都不需要担心WebView崩溃导致应用也跟着崩溃。然后7.0以下的系统就没有这么幸运了，特别是低版本的WebView。考虑应用的稳定性，我们可以把7.0以下系统的WebView使用一个独立进程的Activity来包装，这样即使WebView崩溃了，也只是WebView所在的进程发生了崩溃，主进程还是不受影响的。

```
public static Intent getWebViewIntent(Context context) {
    Intent intent;
    if (isWebInMainProcess()) {
        intent = new Intent(context, MainWebviewActivity.class);
    } else {
        intent = new Intent(context, WebviewActivity.class);
    }
    return intent;
}

public static boolean isWebInMainProcess() {
    return android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.N;
}
```

## WebView免流

从去年开始，市场上出现了一批互联网套餐卡，如腾讯王卡、蚂蚁宝卡、京东强卡、阿里鱼卡、网易白金卡等，这些互联网套餐相比传统的运营商套餐来说，资费便宜，流量多，甚至某些卡还拥有特殊权限——对某些应用免流。如[网易白金卡](https://bjk.163.com/)，对于网易系与百度系的部分应用实现免流。

### 免流原理

市面上常见的免流应用，原理无非就是走“特殊通道”，让这一部分的流量不计入运营商的流量统计平台中。Android中要实现这种“特殊通道”，有几种方案。

1. 微屁恩。目前运营商貌似没有采用这种方案，但确实是可行的。由于国情，不多介绍，懂的自然懂。
2. 全局代理。把所有的流量中转到代理服务器中，代理服务器再根据流量判断是否属于免流流量。
3. IP直连。走这个IP的所有流量，服务器判断是否免流。

### WebView免流方案

对于上面提到的几种方案，native页面所有的请求都是应用层发起的，实际上都比较好实现，但WebView的页面和资源请求是通过JNI发起的，想要拦截请求的话，需要一些功夫。网罗网上的所有方案，目前觉得可行的有两种，分别是全局代理和拦截`WebViewClient.shouldInterceptRequest()`。

#### 全局代理

由于WebView并没有提供接口针对具体的WebView实例设置代理，所以我们只能进行全局代理。设置全局代理时，需要通知系统代理环境发生了改变，不幸地是，Android并没有提供公开的接口，这就导致了我们只能hook系统接口，根据不同的系统版本来实现通知的目的[^9]、[^10]。6.0以后的系统，尚未尝试是否可行，根据公司同事的反馈，和5.0系统的方案是一致的。

```
/**
 * Set Proxy for Android 4.1 - 4.3.
 */
@SuppressWarnings("all")
private static boolean setProxyJB(WebView webview, String host, int port) {
    Log.d(LOG_TAG, "Setting proxy with 4.1 - 4.3 API.");

    try {
        Class wvcClass = Class.forName("android.webkit.WebViewClassic");
        Class wvParams[] = new Class[1];
        wvParams[0] = Class.forName("android.webkit.WebView");
        Method fromWebView = wvcClass.getDeclaredMethod("fromWebView", wvParams);
        Object webViewClassic = fromWebView.invoke(null, webview);

        Class wv = Class.forName("android.webkit.WebViewClassic");
        Field mWebViewCoreField = wv.getDeclaredField("mWebViewCore");
        Object mWebViewCoreFieldInstance = getFieldValueSafely(mWebViewCoreField, webViewClassic);

        Class wvc = Class.forName("android.webkit.WebViewCore");
        Field mBrowserFrameField = wvc.getDeclaredField("mBrowserFrame");
        Object mBrowserFrame = getFieldValueSafely(mBrowserFrameField, mWebViewCoreFieldInstance);

        Class bf = Class.forName("android.webkit.BrowserFrame");
        Field sJavaBridgeField = bf.getDeclaredField("sJavaBridge");
        Object sJavaBridge = getFieldValueSafely(sJavaBridgeField, mBrowserFrame);

        Class ppclass = Class.forName("android.net.ProxyProperties");
        Class pparams[] = new Class[3];
        pparams[0] = String.class;
        pparams[1] = int.class;
        pparams[2] = String.class;
        Constructor ppcont = ppclass.getConstructor(pparams);

        Class jwcjb = Class.forName("android.webkit.JWebCoreJavaBridge");
        Class params[] = new Class[1];
        params[0] = Class.forName("android.net.ProxyProperties");
        Method updateProxyInstance = jwcjb.getDeclaredMethod("updateProxy", params);

        updateProxyInstance.invoke(sJavaBridge, ppcont.newInstance(host, port, null));
    } catch (Exception ex) {
        Log.e(LOG_TAG, "Setting proxy with >= 4.1 API failed with error: " + ex.getMessage());
        return false;
    }

    Log.d(LOG_TAG, "Setting proxy with 4.1 - 4.3 API successful!");
    return true;
}

/**
 * Set Proxy for Android 5.0.
 */
public static void setWebViewProxyL(Context context, String host, int port) {
    System.setProperty("http.proxyHost", host);
    System.setProperty("http.proxyPort", port + "");
    try {
        Context appContext = context.getApplicationContext();
        Class applictionClass = Class.forName("android.app.Application");
        Field mLoadedApkField = applictionClass.getDeclaredField("mLoadedApk");
        mLoadedApkField.setAccessible(true);
        Object mloadedApk = mLoadedApkField.get(appContext);
        Class loadedApkClass = Class.forName("android.app.LoadedApk");
        Field mReceiversField = loadedApkClass.getDeclaredField("mReceivers");
        mReceiversField.setAccessible(true);
        ArrayMap receivers = (ArrayMap) mReceiversField.get(mloadedApk);
        for (Object receiverMap : receivers.values()) {
            for (Object receiver : ((ArrayMap) receiverMap).keySet()) {
                Class clazz = receiver.getClass();
                if (clazz.getName().contains("ProxyChangeListener")) {
                    Method onReceiveMethod = clazz.getDeclaredMethod("onReceive", Context.class, Intent.class);
                    Intent intent = new Intent(Proxy.PROXY_CHANGE_ACTION);
                    onReceiveMethod.invoke(receiver, appContext, intent);
                }
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

需要注意的是，在WebView退出时，需要重置代理。

#### 拦截`WebViewClient.shouldInterceptRequest()`

拦截`WebViewClient.shouldInterceptRequest()`的目的是使用免流的第三种方案——IP替换。直接看代码。

```
@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
@Override
public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
    WebResourceResponse resourceResponse = CandyWebCache.getsInstance().getResponse(view, request);
    if (request.getUrl() != null && request.getMethod().equalsIgnoreCase("get")) {
        Uri uri = request.getUrl();
        String url = uri.toString();
        String scheme = uri.getScheme().trim();
        String host = uri.getHost();
        String path = uri.getPath();
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(host)) {
            return null;
        }
        // HttpDns解析css文件的网络请求及图片请求
        if ((scheme.equalsIgnoreCase("http") || scheme.equalsIgnoreCase("https")) && (path.endsWith(".css")
                || path.endsWith(".png")
                || path.endsWith(".jpg")
                || path.endsWith(".gif")
                || path.endsWith(".js"))) {
            try {
                URL oldUrl = new URL(uri.toString());
                URLConnection connection;
                // 获取HttpDns域名解析结果
                List<String> ips = HttpDnsManager.getInstance().getIPListByHostAsync(host);
                if (!ListUtils.isEmpty(ips)) {
                    String ip = ips.get(0);
                    String newUrl = url.replaceFirst(oldUrl.getHost(), ip);
                    connection = new URL(newUrl).openConnection(); // 设置HTTP请求头Host域
                    connection.setRequestProperty("Host", oldUrl.getHost());
                } else {
                    connection = new URL(url).openConnection(); // 设置HTTP请求头Host域
                }
                String fileExtension = MimeTypeMap.getFileExtensionFromUrl(url);
                String mimeType = MimeTypeMap.getSingleton().getMimeTypeFromExtension(fileExtension);
                return new WebResourceResponse(mimeType, "UTF-8", connection.getInputStream());
            } catch (MalformedURLException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return super.shouldInterceptRequest(view, request);
}

@Override
public WebResourceResponse shouldInterceptRequest(WebView view, String url) {
    if (!TextUtils.isEmpty(url) && Uri.parse(url).getScheme() != null) {
        Uri uri = Uri.parse(url);
        String scheme = uri.getScheme().trim();
        String host = uri.getHost();
        String path = uri.getPath();
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(host)) {
            return null;
        }
        // HttpDns解析css文件的网络请求及图片请求
        if ((scheme.equalsIgnoreCase("http") || scheme.equalsIgnoreCase("https")) && (path.endsWith(".css")
                || path.endsWith(".png")
                || path.endsWith(".jpg")
                || path.endsWith(".gif")
                || path.endsWith(".js"))) {
            try {
                URL oldUrl = new URL(uri.toString());
                URLConnection connection;
                // 获取HttpDns域名解析结果
                List<String> ips = HttpDnsManager.getInstance().getIPListByHostAsync(host);
                if (!ListUtils.isEmpty(ips)) {
                    String ip = ips.get(0);
                    String newUrl = url.replaceFirst(oldUrl.getHost(), ip);
                    connection = new URL(newUrl).openConnection(); // 设置HTTP请求头Host域
                    connection.setRequestProperty("Host", oldUrl.getHost());
                } else {
                    connection = new URL(url).openConnection(); // 设置HTTP请求头Host域
                }
                String fileExtension = MimeTypeMap.getFileExtensionFromUrl(url);
                String mimeType = MimeTypeMap.getSingleton().getMimeTypeFromExtension(fileExtension);
                return new WebResourceResponse(mimeType, "UTF-8", connection.getInputStream());
            } catch (MalformedURLException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return super.shouldInterceptRequest(view, url);
}
```

使用此种方案，还可以把WebView网络请求与native网络请求使用的框架统一起来，方便管理。

# 总结

本文介绍了WebView在开发中的一些实践经验和优化流程。为了满足业务需求，WebView着实提供了非常丰富的接口供应用层处理业务逻辑。针对WebView的二次开发，本文介绍了一些常用的回调处理逻辑以及开发过程中总结下的经验。由于是经验，不一定是准确的，若有错误的地方，敬请指出纠正，不胜感激！


-----

# 参考链接

[^1]: <https://medium.com/@filipe.batista/inject-javascript-into-webview-2b702a2a029f>
[^2]: <https://stackoverflow.com/questions/21552912/android-web-view-inject-local-javascript-file-to-remote-webpage>
[^3]: <https://stackoverflow.com/questions/23172247/android-webview-javascript-injection>
[^4]: <https://developer.android.com/about/versions/nougat/android-7.0.html#webview>
[^5]: <https://developers.google.com/web/tools/chrome-devtools/remote-debugging/webviews>
[^6]: <https://www.jianshu.com/p/5e7075f4875f>
[^7]: <https://developer.android.com/about/versions/nougat/android-7.0.html#webview>
[^8]: <https://developer.android.com/about/versions/oreo/android-8.0-changes.html#security-all>
[^9]: <https://stackoverflow.com/questions/25272393/android-webview-set-proxy-programmatically-on-android-l/25485747#25485747>
[^10]: <https://stackoverflow.com/questions/4488338/webview-android-proxy>