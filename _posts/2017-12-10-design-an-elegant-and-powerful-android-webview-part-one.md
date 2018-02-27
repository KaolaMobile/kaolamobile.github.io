---

layout:     post

title:      "如何设计一个优雅健壮的Android WebView？（上）"

subtitle:   "基于考拉电商平台的WebView实践"

date:       2017-12-10

author:     "黎星"

header-img: "img/bg9.jpg"

tags:

    -Android

    -WebView
    
    -301/302
    
    -Redirect
    
    -POST
    
---

# 前言

Android应用层的开发有几大模块，其中WebView是最重要的模块之一。网上能够搜索到的WebView资料可谓寥寥，Github上的[开源项目](https://android-arsenal.com/tag/230)也不是很多，更别提有一个现成封装好的WebView容器直接用于生产环境了。本文仅当记录在使用WebView实现业务需求时所踩下的一些坑，并提供一些解决思路，避免遇到相同问题的朋友再次踩坑。

# WebView现状

Android系统的WebView发展历史可谓一波三折，系统WebView开发者肯定费劲心思才换取了今天的局面——应用里的WebView和Chrome表现一致。对于Android初学者，或者刚要开始接触WebView的开发来说，WebView是有点难以适应，甚至是有一些惧怕的。开源社区对于WebView的改造和包装非常少，需要开发者查找大量资料去理解WebView。

## WebView Changelog

在Android4.4（API level 19）系统以前，Android使用了原生自带的Android Webkit内核，这个内核对HTML5的支持不是很好，现在使用4.4以下机子的也不多了，就不对这个内核做过多介绍了，有兴趣可以看下[这篇文章](http://blog.csdn.net/tanqiantot/article/details/7966263)。

从Android4.4系统开始，Chromium内核取代了Webkit内核，正式地接管了WebView的渲染工作。Chromium是一个开源的浏览器内核项目，基于Chromium开源项目修改实现的浏览器非常多，包括最著名的Chrome浏览器，以及一众国内浏览器（360浏览器、QQ浏览器等）。其中Chromium在Android上面的实现是`Android System WebView`[^1]。

从Android5.0系统开始，WebView移植成了一个独立的apk，可以不依赖系统而独立存在和更新，我们可以在`系统->设置->Android System WebView`看到WebView的当前版本。

从Android7.0系统开始，如果系统安装了Chrome (version>51)，那么Chrome将会直接为应用的WebView提供渲染，WebView版本会随着Chrome的更新而更新，用户也可以选择WebView的服务提供方（在开发者选项->WebView Implementation里），WebView可以脱离应用，在一个独立的沙盒进程中渲染页面（需要在开发者选项里打开）[^2]。

从Android8.0系统开始，默认开启WebView多进程模式，即WebView运行在独立的沙盒进程中[^3]。

## 为什么WebView那么难搞？

尽管应用开发者使用WebView和使用普通的View一样简单，只需要在xml里定义或者直接实例化出来即可使用，但WebView是相当难搞的。为什么呢？以下有几个可能的因素。

* 繁杂的WebView配置

WebView在初始化的时候就提供了默认配置`WebSettings`，但是很多默认配置是不能够满足业务需求的，还需要进行二次配置，例如考拉App在默认配置基础做了如下修改：

```
public static void setDefaultWebSettings(WebView webView) {
    WebSettings webSettings = webView.getSettings();
    //5.0以上开启混合模式加载
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        webSettings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
    }
    webSettings.setLoadWithOverviewMode(true);
    webSettings.setUseWideViewPort(true);
    //允许js代码
    webSettings.setJavaScriptEnabled(true);
    //允许SessionStorage/LocalStorage存储
    webSettings.setDomStorageEnabled(true);
    //禁用放缩
    webSettings.setDisplayZoomControls(false);
    webSettings.setBuiltInZoomControls(false);
    //禁用文字缩放
    webSettings.setTextZoom(100);
    //10M缓存，api 18后，系统自动管理。
    webSettings.setAppCacheMaxSize(10 * 1024 * 1024);
    //允许缓存，设置缓存位置
    webSettings.setAppCacheEnabled(true);
    webSettings.setAppCachePath(context.getDir("appcache", 0).getPath());
    //允许WebView使用File协议
    webSettings.setAllowFileAccess(true);
    //不保存密码
    webSettings.setSavePassword(false);
	//设置UA
    webSettings.setUserAgentString(webSettings.getUserAgentString() + " kaolaApp/" + AppUtils.getVersionName());
    //移除部分系统JavaScript接口
    KaolaWebViewSecurity.removeJavascriptInterfaces(webView);
    //自动加载图片
    webSettings.setLoadsImagesAutomatically(true);
}
```

除此之外，使用方还需要根据业务需求实现`WebViewClient`和`WebChromeClient`，这两个类所需要覆写的方法更多，用来实现标题定制、加载进度条控制、jsbridge交互、url拦截、错误处理（包括http、资源、网络）等很多与业务相关的功能。

* 复杂的前端环境

如今，万维网的核心语言，超文本标记语言已经发展到了HTML5，随之而来的是html、css、js相应的升级与更新。高版本的语法无法在低版本的内核上识别和渲染，业务上需要使用到新的特性时，开发不得不面对后向兼容的问题。互联网的链接千千万万，使用哪些语言特性不是WebView能决定的，要求WebView适配所有页面几乎是不可能的事情。

* 版本间差异

WebView不同的版本方法的实现是有可能不一样的，而前端一般情况下只会调用系统的api来实现功能，这就会导致Android不同的系统、不同的WebView版本表现不一致的情况。一个典型的例子是下面即将描述的WebView中的文件上传功能，当我们在Web页面上点击选择文件的控件(`<input type="file">`)时，会产生不同的回调方法。除了文件上传功能，版本间的差异还有很多很多，比如缓存机制的版本差异，js安全漏洞的屏蔽，cookie管理等。Google也在想办法解决这些差异给开发者带来的适配压力，例如Webkit内核到Chromium内核的切换对开发者是透明的，底层的API完全没有改变，这也是好的设计模式带来的益处。

* 国内ROM、浏览器对WebView内核的魔改

国产手机的厂商基本在出厂时都自带了浏览器，查看系统应用时，发现并没有内置`com.android.webview`或者`com.google.android.webview`包，这些浏览器并不是简单地套了一层WebView的壳，而是直接使用了Chromium内核，至于有没有魔改过内核源码，不得而知。国产出品的浏览器，如360浏览器、QQ浏览器、UC浏览器，几乎都魔改了内核。值得一提的是，腾讯出品的X5内核，号称页面渲染流畅度高于原生内核，客户端减少了WebView带来坑的同时，增加了前端适配的难度，功能实现上需要有更多地考虑。

* 需要一定的Web知识

如果仅仅会使用`WebView.loadUrl()`来加载一个网页而不了解底层到底发生了什么，那么url发生错误、url中的某些内容加载不出来、url里的内容点击无效、支付宝支付浮层弹不起来、与前端无法沟通等等问题就会接踵而至。要开发好一个功能完整的WebView，需要对Web知识（html、js、css）有一定了解，知道loadUrl，WebView在后台请求这个url以后，服务器做了哪些响应，又下发了哪些资源，这些资源的作用是怎么样的。

## 为什么Github上的WebView项目不适用？

从[上面的链接](https://android-arsenal.com/tag/230)可以看到，Github上面star过千的WebView项目主要是[FinestWebView-Android](https://github.com/TheFinestArtist/FinestWebView-Android)和[Android-AdvancedWebView](https://github.com/delight-im/Android-AdvancedWebView)。看过源码的话应该知道，第一个项目偏向于实现一个浏览器，第二个项目提供的接口太少，并且一些坑并未填完。陆续看过几个别的开源实现，发现并不理想。后来想想，很难不依赖于业务而单独实现一个WebView，特别是与前端约定了jsbridge接口，需要处理页面关闭、全屏、url拦截、登录、分享等一系列功能，即便是接入了开源平台的WebView，也需要做大量的扩展才有可能完全满足需求。与其如此，每个电商平台都有自己一套规则，基于电商的业务需求来自己扩展WebView是比较合理的。

# WebView踩坑历程

可以说，如果是初次接触WebView，不踩坑几乎是不可能的。笔者在接触到前人留下来的WebView代码时，有些地方写的很trickey，如果不仔细阅读，或者翻阅资料，很有可能就会掉进坑里。下面介绍几个曾经遇到过的坑。

## WebSettings.setJavaScriptEnabled

我相信99%的应用都会调用下面这句

```
WebSettings.setJavaScriptEnabled(true);
```

在Android 4.3版本调用`WebSettings.setJavaScriptEnabled()`方法时会调用一下reload方法，同时会回调多次`WebChromeClient.onJsPrompt()`。如果有业务逻辑依赖于这两个方法，就需要注意判断回调多次是否会带来影响了。

同时，如果启用了JavaScript，务必做好安全措施，防止远程执行漏洞[^5]。

```
@TargetApi(11)
private static final void removeJavascriptInterfaces(WebView webView) {
    try {
        if (Build.VERSION.SDK_INT >= 11 && Build.VERSION.SDK_INT < 17) {
	        webView.removeJavascriptInterface("searchBoxJavaBridge_");
	        webView.removeJavascriptInterface("accessibility");
	        webView.removeJavascriptInterface("accessibilityTraversal");
        }
    } catch (Throwable tr) {
        tr.printStackTrace();
    }
}
```

## 301/302重定向问题

WebView的301/302重定向问题，绝对在踩坑排行榜里名列前茅。。。随便搜了几个解决方案，要么不能满足业务需求，要么清一色没有彻底解决问题。

<https://stackoverflow.com/questions/4066438/android-webview-how-to-handle-redirects-in-app-instead-of-opening-a-browser>
<http://blog.csdn.net/jdsjlzx/article/details/51698250>
<http://www.cnblogs.com/pedro-neer/p/5318354.html>
<http://www.jianshu.com/p/c01769ababfa>

### 301/302业务场景及白屏问题

先来分析一下业务场景。对于需要对url进行拦截以及在url中需要拼接特定参数的WebView来说，301和302发生的情景主要有以下几种：

* 首次进入，有重定向，然后直接加载H5页面，如http跳转https
* 首次进入，有重定向，然后跳转到native页面，如扫一扫短链，然后跳转到native
* 二次加载，有重定向，跳转到native页面
* 对于考拉业务来说，还有类似登录后跳转到某个页面的需求。如我的拼团，未登录状态下点击我的拼团跳转到登录页面，登录完成后再加载我的拼团页。

第一种情况属于正常情况，暂时没遇到什么坑。

第二种情况，会遇到**WebView空白页问题**，属于原始url不能拦截到native页面，但301/302后的url拦截到native页面的情况，当遇到这种情况时，需要把WebView对应的Activity结束，否则当用户从拦截后的页面返回上一个页面时，是一个WebView空白页。

第三种情况，也会遇到**WebView空白页问题**，原因在于加载的第一个页面发生了重定向到了第二个页面，第二个页面被客户端拦截跳转到native页面，那么WebView就停留在第一个页面的状态了，第一个页面显然是空白页。

第四种情况，会遇到**无限加载登录页面的问题**。考拉的登录链接是类似下面这种格式：

```
https://m.kaola.com/login.html?target=登录后跳转的url
```

如果登录成功后还重新加载这个url，那么就会循环跳转到登录页面。第四点解决起来比较简单，登录成功以后拿到target后的跳转url再重新加载即可。

### 301/302回退栈问题

无论是哪种重定向场景，都不可避免地会遇到回退栈的处理问题，如果处理不当，用户按返回键的时候不一定能回到重定向之前的那个页面。很多开发者在覆写`WebViewClient.shouldOverrideUrlLoading()`方法时，会简单地使用以下方式粗暴处理：

```
WebView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
    	view.loadUrl(url);
    	return true;
    }
    ...
)
```

这种方法最致命的弱点就是如果不经过特殊处理，那么按返回键是没有效果的，还会停留在302之前的页面。现有的解决方案无非就几种：

1. 手动管理回退栈，遇到重定向时回退两次[^6]。
2. 通过`HitTestResult`判断是否是重定向，从而决定是否自己加载url[^7] [^8]。
3. 通过设置标记位，在`onPageStarted`和`onPageFinished`分别标记变量避免重定向[^9]。

可以说，这几种解决方案都不是完美的，都有缺陷。

### 301/302较优解决方案

#### 解决301/302回退栈问题

能否结合上面的几种方案，来更加准确地判断301/302的情况呢？下面说一下本文的解决思路。在提供解决方案之前，我们需要了解一下`shouldOverrideUrlLoading`方法的返回值代表什么意思。

> Give the host application a chance to take over the control when a new url is about to be loaded in the current WebView. If WebViewClient is not provided, by default WebView will ask Activity Manager to choose the proper handler for the url. If WebViewClient is provided, **return true means the host application handles the url, while return false means the current WebView handles the url.**

简单地说，就是返回true，那么url就已经由客户端处理了，WebView就不管了，如果返回false，那么当前的WebView实现就会去处理这个url。

WebView能否知道某个url是不是301/302呢？当然知道，WebView能够拿到url的请求信息和响应信息，根据header里的code很轻松就可以实现，事实正是如此，交给WebView来处理重定向(`return false`)，这时候按返回键，是可以正常地回到重定向之前的那个页面的。（PS：从上面的章节可知，WebView在5.0以后是一个独立的apk，可以单独升级，新版本的WebView实现肯定处理了重定向问题）

但是，业务对url拦截有需求，肯定不能把所有的情况都交给系统WebView处理。为了解决url拦截问题，本文引入了另一种思想——通过用户的touch事件来判断重定向。下面通过代码来说明。

```
/**
 * WebView基础类，处理一些基础的公有操作
 *
 * @author xingli
 * @time 2017-12-06
 */
public class BaseWebView extends WebView {

    private boolean mTouchByUser;

    public BaseWebView(Context context) {
        super(context);
    }

    public BaseWebView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public BaseWebView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public final void loadUrl(String url, Map<String, String> additionalHttpHeaders) {
        super.loadUrl(url, additionalHttpHeaders);
        resetAllStateInternal(url);
    }

    @Override
    public void loadUrl(String url) {
        super.loadUrl(url);
        resetAllStateInternal(url);
    }

    @Override
    public final void postUrl(String url, byte[] postData) {
        super.postUrl(url, postData);
        resetAllStateInternal(url);
    }

    @Override
    public final void loadData(String data, String mimeType, String encoding) {
        super.loadData(data, mimeType, encoding);
        resetAllStateInternal(getUrl());
    }

    @Override
    public final void loadDataWithBaseURL(String baseUrl, String data, String mimeType, String encoding,
            String historyUrl) {
        super.loadDataWithBaseURL(baseUrl, data, mimeType, encoding, historyUrl);
        resetAllStateInternal(getUrl());
    }

    @Override
    public void reload() {
        super.reload();
        resetAllStateInternal(getUrl());
    }

    public boolean isTouchByUser() {
        return mTouchByUser;
    }

    private void resetAllStateInternal(String url) {
        if (!TextUtils.isEmpty(url) && url.startsWith("javascript:")) {
            return;
        }
        resetAllState();
    }

	// 加载url时重置touch状态
    protected void resetAllState() {
        mTouchByUser = false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
            	//用户按下到下一个链接加载之前，置为true
                mTouchByUser = true;
                break;
        }
        return super.onTouchEvent(event);
    }

    @Override
    public void setWebViewClient(final WebViewClient client) {
        super.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                boolean handleByChild = null != client && client.shouldOverrideUrlLoading(view, url);
            	   if (handleByChild) {
             		// 开放client接口给上层业务调用，如果返回true，表示业务已处理。
                    return true;
            	   } else if (!isTouchByUser()) {
             		// 如果业务没有处理，并且在加载过程中用户没有再次触摸屏幕，认为是301/302事件，直接交由系统处理。
                    return super.shouldOverrideUrlLoading(view, url);
                } else {
                	//否则，属于二次加载某个链接的情况，为了解决拼接参数丢失问题，重新调用loadUrl方法添加固有参数。
                    loadUrl(url);
                    return true;
                }
            }

            @RequiresApi(api = Build.VERSION_CODES.N)
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
                boolean handleByChild = null != client && client.shouldOverrideUrlLoading(view, request);

                if (handleByChild) {
                    return true;
                } else if (!isTouchByUser()) {
                    return super.shouldOverrideUrlLoading(view, request);
                } else {
                    loadUrl(request.getUrl().toString());
                    return true;
                }
            }
        });
    }
}
```

上述代码解决了正常情况下的回退栈问题。

#### 解决业务白屏问题

为了解决白屏问题，考拉目前的解决思路和上面的回退栈问题思路有些类似，通过监听`touch`事件分发以及`onPageFinished`事件来判断是否产生白屏，代码如下：

```
public class KaolaWebview extends BaseWebView implements DownloadListener, Lifeful, OnActivityResultListener {

    private boolean mIsBlankPageRedirect;  //是否因重定向导致的空白页面。

    public KaolaWebview(Context context) {
        super(context);
        init();
    }

    public KaolaWebview(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public KaolaWebview(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    protected void back() {
        if (mBackStep < 1) {
            mJsApi.trigger2("kaolaGoback");
        } else {
            realBack();
        }
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_UP) {
            mIsBlankPageRedirect = true;
        }
        return super.dispatchTouchEvent(ev);
    }

    private WebViewClient mWebViewClient = new WebViewClient() {
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            url = WebViewUtils.removeBlank(url);
            //允许启动第三方应用客户端
            if (WebViewUtils.canHandleUrl(url)) {
                boolean handleByCaller = false;
                // 如果不是用户触发的操作，就没有必要交给上层处理了，直接走url拦截规则。
                if (null != mIWebViewClient && isTouchByUser()) {
                    handleByCaller = mIWebViewClient.shouldOverrideUrlLoading(view, url);
                }
                if (!handleByCaller) {
                    handleByCaller = handleOverrideUrl(url);
                }
                return handleByCaller || super.shouldOverrideUrlLoading(view, url);
            } else {
                try {
                    notifyBeforeLoadUrl(url);
                    Intent intent = Intent.parseUri(url, Intent.URI_INTENT_SCHEME);
                    intent.addCategory(Intent.CATEGORY_BROWSABLE);
                    mContext.startActivity(intent);
                    if (!mIsBlankPageRedirect) {
                    	// 如果遇到白屏问题，手动后退
                        back();
                    }
                } catch (Exception e) {
                    ExceptionUtils.printExceptionTrace(e);
                }
                return true;
            }
        }

        @RequiresApi(Build.VERSION_CODES.LOLLIPOP)
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
            return shouldOverrideUrlLoading(view, request.getUrl().toString());
        }
        
        private boolean handleOverrideUrl(final String url) {
           RouterResult result =  WebActivityRouter.startFromWeb(
                    new IntentBuilder(mContext, url).setRouterActivityResult(new RouterActivityResult() {
                        @Override
                        public void onActivityFound() {
                            if (!mIsBlankPageRedirect) {
                    			// 路由已经拦截到跳转到native页面，但此时可能发生了
                    			// 301/302跳转，那么执行后退动作，防止白屏。
                                back();
                            }
                        }

                        @Override
                        public void onActivityNotFound() {
                            if (mIWebViewClient != null) {
                                mIWebViewClient.onActivityNotFound();
                            }
                        }
                    }));
            return result.isSuccess();
        }
    };

    @Override
    public void onPageFinished(WebView view, String url) {
        mIsBlankPageRedirect = true;
        if (null != mIWebViewClient) {
            mIWebViewClient.onPageReallyFinish(view, url);
        }
        super.onPageFinished(view, url);
    }
}
```

本来上面的两个问题可以用同一个变量控制解决的，但由于历史代码遗留问题，目前还没有时间优化测试，这也是代码暂不公布的原因之一（代码太丑陋`:(`）。

## url参数拼接问题

一般情况下，WebView会拼接一些本地参数作为识别码传给前端，如app版本号，网络状态等，例如需要加载的url是

```
http://m.kaola.com?platform=android
```

假设我们拼接appVersion和network，则拼接后url变成：

```
http://m.kaola.com?platform=android&appVersion=3.10.0&network=4g
```

使用`WebView.loadUrl()`加载上面拼接好的url，随意点击这个页面上的某个链接跳转到别的页面，本地拼接的参数是不会自动带过去的。如果需要前端处理参数问题，那么如果是同域，可以通过cookie传递。非同域的话，还是需要客户端拼接参数带过去。

## 部分机型没有WebView，应用直接崩溃

在Crash平台上面发现有部分机型会存在下面这个崩溃，这些机型都是7.0系统及以上的。

```
android.util.AndroidRuntimeException: android.webkit.WebViewFactory$MissingWebViewPackageException: Failed to load WebView provider: No WebView installed
at android.webkit.WebViewFactory.getProviderClass(WebViewFactory.java:371)
at android.webkit.WebViewFactory.getProvider(WebViewFactory.java:194)
at android.webkit.WebView.getFactory(WebView.java:2325)
at android.webkit.WebView.ensureProviderCreated(WebView.java:2320)
at android.webkit.WebView.setOverScrollMode(WebView.java:2379)
at android.view.View.(View.java:4015)
at android.view.View.(View.java:4132)
at android.view.ViewGroup.(ViewGroup.java:578)
at android.widget.AbsoluteLayout.(AbsoluteLayout.java:55)
at android.webkit.WebView.(WebView.java:627)
at android.webkit.WebView.(WebView.java:572)
at android.webkit.WebView.(WebView.java:555)
at android.webkit.WebView.(WebView.java:542)
at com.kaola.modules.webview.BaseWebView.void (android.content.Context)(Unknown Source)
```

经过测试发现，普通用户是没有办法卸载WebView的（即使能卸载，也只是把更新卸载了，原始版本的WebView还是存在的），所以理论上不会存在异常……但既然发生并且上传上来了，那么就需要细细分析一下原因了。跟着代码`WebViewFactory.getProvider()`走，

```
static WebViewFactoryProvider getProvider() {
    synchronized (sProviderLock) {
        // For now the main purpose of this function (and the factory abstraction) is to keep
        // us honest and minimize usage of WebView internals when binding the proxy.
        if (sProviderInstance != null) return sProviderInstance;

        final int uid = android.os.Process.myUid();
        if (uid == android.os.Process.ROOT_UID || uid == android.os.Process.SYSTEM_UID
                || uid == android.os.Process.PHONE_UID || uid == android.os.Process.NFC_UID
                || uid == android.os.Process.BLUETOOTH_UID) {
            throw new UnsupportedOperationException(
                    "For security reasons, WebView is not allowed in privileged processes");
        }

        StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskReads();
        Trace.traceBegin(Trace.TRACE_TAG_WEBVIEW, "WebViewFactory.getProvider()");
        try {
            Class<WebViewFactoryProvider> providerClass = getProviderClass();
            Method staticFactory = null;
            try {
                staticFactory = providerClass.getMethod(
                    CHROMIUM_WEBVIEW_FACTORY_METHOD, WebViewDelegate.class);
            } catch (Exception e) {
                if (DEBUG) {
                    Log.w(LOGTAG, "error instantiating provider with static factory method", e);
                }
            }

            Trace.traceBegin(Trace.TRACE_TAG_WEBVIEW, "WebViewFactoryProvider invocation");
            try {
                sProviderInstance = (WebViewFactoryProvider)
                        staticFactory.invoke(null, new WebViewDelegate());
                if (DEBUG) Log.v(LOGTAG, "Loaded provider: " + sProviderInstance);
                return sProviderInstance;
            } catch (Exception e) {
                Log.e(LOGTAG, "error instantiating provider", e);
                throw new AndroidRuntimeException(e);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_WEBVIEW);
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WEBVIEW);
            StrictMode.setThreadPolicy(oldPolicy);
        }
    }
}
```

可以看到，获取WebView的实例，就是先拿到`WebViewFactoryProvider`这个工厂类，通过`WebViewFactoryProvider`工厂类里的静态方法`CHROMIUM_WEBVIEW_FACTORY_METHOD`创建一个`WebViewFactoryProvider`，接着，调用`WebViewFactoryProvider.createWebView()`创建一个`WebViewProvider`（相当于WebView的代理类），后面WebView的方法都是通过代理类来实现的。

在第一步获取`WebVIewFactoryProvider`类的过程中，

```
private static Class<WebViewFactoryProvider> getProviderClass() {
    Context webViewContext = null;
    Application initialApplication = AppGlobals.getInitialApplication();

    try {
    	//获取WebView上下文并设置provider
        webViewContext = getWebViewContextAndSetProvider();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_WEBVIEW);
    }
	 代码省略...
    }
}

private static Context getWebViewContextAndSetProvider() {
    Application initialApplication = AppGlobals.getInitialApplication();
    WebViewProviderResponse response = null;
    Trace.traceBegin(Trace.TRACE_TAG_WEBVIEW,
            "WebViewUpdateService.waitForAndGetProvider()");
    try {
        response = getUpdateService().waitForAndGetProvider();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_WEBVIEW);
    }
    if (response.status != LIBLOAD_SUCCESS
            && response.status != LIBLOAD_FAILED_WAITING_FOR_RELRO) {
        // 崩溃就发生在这里。
        throw new MissingWebViewPackageException("Failed to load WebView provider: "
                + getWebViewPreparationErrorReason(response.status));
    }
}

```

可以发现，在与WebView包通信的过程中，so库并没有加载成功，最后代码到了native层，没有继续跟下去了。

对于这种问题，解决方案有两种，一种是判断包名，如果检测到系统包名里不包含`com.google.android.webview`或者`com.android.webview`，则认为用户手机里的WebView不可用；另外一种是通过try/catch判断WebView实例化是否成功，如果抛出了`WebViewFactory$MissingWebViewPackageException`异常，则认为用户的WebView不可用。

需要说明的是，第一种解决方案是不可靠的，因为国内的厂商基于Chromium的WebView实现有很多种，很有可能包名就被换了，比如MiWebView，包名是`com.mi.webkit.core`。

## WebView中的POST请求

在WebView中，如果前端使用POST方式向后端发起一个请求，那么这个请求是不会走到`WebViewClient.shouldOverrideUrlLoading()`方法里的[^10]。网上有一些解决方案，例如[android-post-webview](https://github.com/KeejOow/android-post-webview)，通过js判断是否是post请求，如果是的话，在`WebViewClient.shouldInterceptRequest()`方法里自己建立连接，并拿到对应的页面信息，返回给`WebResourceResponse`。总之，尽量避免Web页面使用POST请求，否则会带来很大不必要的麻烦。

## WebView文件上传功能

WebView中的文件上传功能，当我们在Web页面上点击选择文件的控件(`<input type="file">`)时，会产生不同的回调方法:[^4]

> void openFileChooser(ValueCallback<Uri> uploadMsg) works on Android 2.2 (API level 8) up to Android 2.3 (API level 10)
> 
> openFileChooser(ValueCallback<Uri> uploadMsg, String acceptType) works on Android 3.0 (API level 11) up to Android 4.0 (API level 15)
> 
> openFileChooser(ValueCallback<Uri> uploadMsg, String acceptType, String capture) works on Android 4.1 (API level 16) up to Android 4.3 (API level 18)
> 
> onShowFileChooser(WebView webView, ValueCallback<Uri[]> filePathCallback, WebChromeClient.FileChooserParams fileChooserParams) works on Android 5.0 (API level 21) and above

最坑的点是在Android4.4系统上没有回调，这将导致功能的不完整，需要前端去做兼容。解决方案就是和前端另外约定一个jsbridge来解决此类问题。

# 总结

限于篇幅，《如何设计一个优雅健壮的Android WebView？（上）》先介绍到这里。本文介绍了目前Android里的WebView现状，以及由于现状的不可改变导致遗留下的一些坑。所幸，世界上没有什么代码问题是一个程序员不能解决的，如果有，那就用两个程序员解决。既然我们已经把前人留下的一些坑填了，那么是时候构造一个可以用于生产环境的WebView了！《如何设计一个优雅健壮的Android WebView？（下）》将会介绍如何打造WebView的实战操作，以及为了用户更好的体验，提出的一些WebView优化策略，敬请期待。

# 参考链接

1. <https://developer.chrome.com/multidevice/webview/overview>
2. <https://developer.android.com/about/versions/nougat/android-7.0.html#webview>
3. <https://developer.android.com/about/versions/oreo/android-8.0-changes.html#o-sec>
4. <https://stackoverflow.com/questions/30078217/why-openfilechooser-in-webchromeclient-is-hidden-from-the-docs-is-it-safe-to-us>
5. <http://blog.csdn.net/self_study/article/details/55046348>
6. <http://qbeenslee.com/article/android-webview-302-redirect/>
7. <https://juejin.im/entry/5977598d51882548c0045bde>
8. <http://www.cnblogs.com/zimengfang/p/6183869.html>
9. <http://blog.csdn.net/dg_summer/article/details/78105582>
10. <https://issuetracker.google.com/issues/36918490>



[^1]: <https://developer.chrome.com/multidevice/webview/overview>
[^2]: <https://developer.android.com/about/versions/nougat/android-7.0.html#webview>
[^3]: <https://developer.android.com/about/versions/oreo/android-8.0-changes.html#o-sec>
[^4]: <https://stackoverflow.com/questions/30078217/why-openfilechooser-in-webchromeclient-is-hidden-from-the-docs-is-it-safe-to-us>
[^5]: <http://blog.csdn.net/self_study/article/details/55046348>
[^6]: <http://qbeenslee.com/article/android-webview-302-redirect/>
[^7]: <https://juejin.im/entry/5977598d51882548c0045bde>
[^8]: <http://www.cnblogs.com/zimengfang/p/6183869.html>
[^9]: <http://blog.csdn.net/dg_summer/article/details/78105582>
[^10]: <https://issuetracker.google.com/issues/36918490>
