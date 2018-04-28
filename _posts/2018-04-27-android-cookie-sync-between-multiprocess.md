---

layout:     post

title:      "记一次多进程同步Cookie的解惑历程"

subtitle:   ""

date:       2018-04-27

author:     "黎星"

header-img: "img/bg9.jpg"

tags:

    -Android

    -WebView
    
    -Cookie
    
    -CookieManager
    
    -MultiProcess
    
---

---
title: 记一次多进程同步Cookie的解惑历程
categories: Android
date: 2018-04-27 14:26:55
tags: [Android, WebView, Cookie, CookieManager, MultiProcess]
---

# 前言

谈起Cookie，如果没有了解过它，可能会望文生畏。做过WebView开发的人可能会对它比较了解。Android的Cookie是由系统去管理的，其特点是会被持久化成一个db文件，保存在`/data/data/{packageName}/app_webview/Cookies`中（不同系统、不同浏览器实现可能不一样，但大体如此）。通常，网站的登录信息是使用Cookie来保存的，如果App也是使用Cookie来实现鉴权，那么在WebView和App之间就需要建立一套Cookie同步机制。

尽管考拉的鉴权机制不是使用Cookie来实现的，但我们也遇到了类似的需求，使用WebView打开一个特定的url，这个url的响应会写入指定的Cookie，然后url经过一次302重定向，经过url拦截后打开一个App页面，并把url响应中携带的Cookie带到这个App页面。

# 同进程Cookie同步

如果App和WebView处于同一个进程，那么实现起来是比较简单的，可以参考[这篇文章](https://medium.com/@elye.project/a-tale-on-android-cookies-store-management-b04832ca18c6)，代码不做过多解释，以`okhttp`为例：

```
import android.webkit.CookieManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import okhttp3.Cookie;
import okhttp3.CookieJar;
import okhttp3.HttpUrl;

/**
 * Provides a synchronization point between the webview cookie store and okhttp3.OkHttpClient cookie store
 */
public final class WebviewCookieHandler implements CookieJar {
    private CookieManager webviewCookieManager = CookieManager.getInstance();

    @Override
    public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
        String urlString = url.toString();

        for (Cookie cookie : cookies) {
            webviewCookieManager.setCookie(urlString, cookie.toString());
        }
    }

    @Override
    public List<Cookie> loadForRequest(HttpUrl url) {
        String urlString = url.toString();
        String cookiesString = webviewCookieManager.getCookie(urlString);

        if (cookiesString != null && !cookiesString.isEmpty()) {
            //We can split on the ';' char as the cookie manager only returns cookies
            //that match the url and haven't expired, so the cookie attributes aren't included
            String[] cookieHeaders = cookiesString.split(";");
            List<Cookie> cookies = new ArrayList<>(cookieHeaders.length);

            for (String header : cookieHeaders) {
                cookies.add(Cookie.parse(url, header));
            }

            return cookies;
        }

        return Collections.emptyList();
    }
}
```

> 代码来自 <https://gist.github.com/justinthomas-syncbak/cd29feebd6837d5b45f6576c73faedac>。

# 多进程Cookie同步

但是如果App和WebView处于不同的进程，事情就没那么简单了。由于不同进程之间数据是不共享的，进程之间的Cookie同步就成了一个问题。随后的测试发现，进程之间的Cookie数据不一定同步，但App的多进程是共享一个Cookies文件的。我们遇到的问题是，WebView进程访问携带了特定Cookie的url后，这些Cookie并没有同步到主进程。于是，带着层层疑问，我们开始了进程间同步Cookie的猜想实验。考虑一下两个进程间可能导致Cookie数据不一致的地方（以下假设App在A进程，WebView在B进程）：

1. WebView访问一个url，B进程的WebView写入Cookie以后，没有立即写入Cookies.db持久化，导致B进程读取不到最新的Cookie；
2. 由于Cookie是和WebView挂钩的，可能需要在A进程创建一个WebView来让Cookie在进程间同步；
3. A进程需要调用`CookieManager.getInstance().setAcceptCookie(true)`保证A进程能够读取到Cookie；
4. B进程的Cookie可能失效了，导致A进程读取不到Cookie（后面解释为什么会出现这种情况）；
5. A进程和B进程的Cookie文件根本不是同一个，导致数据无法同步；
6. A进程创建了WebView并且访问了同域的url，然后冲掉了B进程之前已经持久化的Cookie；
7. Cookie是通过CookieManager管理的，CookieManager是个单例，可能只会读取一次Cookies.db，然后缓存在内存中；

下面我们一一分析上述7种情况，并加以条件进行测试。需要说明的是，为了不影响每次实验的结果，都需要在加载url之前，清空`/data/data`目录下的Cookie文件。

## 第一个猜想——Cookie持久化时间

> WebView访问一个url，B进程的WebView写入Cookie以后，没有立即写入Cookies.db持久化，导致B进程读取不到最新的Cookie。

WebView在加载url时，服务端返回需要写入的Cookie可以使用Chrome Inspect来查看。针对WebView的Cookie持久化时机，我们可以做一个简单的实验。

> 实验步骤：  
> 1、使用WebView加载url；  
> 2、加载完成后（调用`WebViewClient.onPageFinished()`），拿到Cookie文件，查看是否有写入Cookie。

以`https://m.baidu.com`为例，未加载WebView组件之前，我们可以找一台root过的手机，查看`/data/data/{packageName}`目录下是没有`app_webview`目录的。

![](https://haitao.nos.netease.com/6d9bbe26-bb50-4b1e-969d-d51a4147bdf8.png)

加载url以后，可以使用chrome inspect查看Cookie信息，`m.baidu.com`会生成以下Cookie：

![baidu_cookie](https://haitao.nos.netease.com/a09a28e3-781a-4b28-a3f4-8d61eafd8068.png)

此时再次访问上述目录，可以发现app_webview目录已经存在了，并且生成了Cookie文件。说明在第一次打开WebView加载完`https://m.baidu.com`的时候就已经生成了Cookie并且持久化，

![](https://haitao.nos.netease.com/25bb33c7-4011-48c8-b886-dfb087a67eb7.png)

为了进一步证实，我们导出`/data/data/{packageName}/app_webview/Cookies`文件，并查看是否包含上面的Cookie，来证实Cookie是否有被持久化。

![](https://haitao.nos.netease.com/7572807a-231f-4f4c-a87e-e11bdcbc7138.png)

结果显而易见——**Cookie在WebView加载完成url以后几乎是立即持久化的**，我们的第一个猜想不成立。

## 第二个猜想——Cookie同步条件

> 由于Cookie是和WebView挂钩的，可能需要在A进程创建一个WebView来让Cookie在进程间同步。

我们知道，WebView的Cookie是交由系统去管理的[^1]，WebView在实例化过程中可能对Cookie进行一定的操作。如果没有实例化WebView，是不是Cookie就同步不过来呢？基于这个猜想，我们进行第二次实验。

> 实验步骤：  
> 1、B进程加载`https://m.baidu.com`后，在B进程使用`CookieManager`查看`m.baidu.com`的Cookie；  
> 2、A进程实例化WebView，不加载，然后在A进程使用`CookieManager`查看`m.baidu.com`的Cookie；  
> 3、B进程再次使用WebView加载`https://m.taobao.com`，在B进程查看`m.taobao.com`的Cookie；  
> 4、A进程再次实例化WebView，不加载，在A进程查看`m.taobao.com`的Cookie。

我们看到一个有趣的现象：

![](https://haitao.nos.netease.com/425d3035-e780-446f-801e-eeb891355db8.png)

首次实例化A进程的WebView时，可以拿到B进程之前写入的Cookie。但当B进程再次写入其他Cookie时，此时再实例化A进程的WebView却取不到了。这个过程可能说明了**只有在第一次实例化WebView的时候才会去同步持久化的Cookie**，当Cookie再次更新时，别的进程读取不到更新后的Cookie数据。第二个猜想不成立。

## 第三个猜想——`setAcceptCookie(true)`

> A进程需要调用`CookieManager.getInstance().setAcceptCookie(true)`保证A进程能够读取到Cookie。

既然需要使用到Cookie，而进程是否默认允许记录Cookie是个未知的行为，索性我们可以测试一下，强制让进程允许记录Cookie。可以使用如下代码：

```
CookieManage.getInstance().setAcceptCookie(true);
```

> 实验步骤：  
> 1、在Application启动的时候调用`CookieManage.getInstance().setAcceptCookie(true);`
> 2、重复猜想二的实验步骤，观察A进程和B进程的Cookie同步情况。
> 3、在Application启动的时候调用`CookieManage.getInstance().setAcceptCookie(false);`
> 4、再次重复猜想二的步骤。

无论是否设置允许记录Cookie，测试结果和猜想二的结果一样，图就不贴了，说明**Cookie在进程间的同步和是否允许记录Cookie无关**。第三个猜想不成立。

## 第四个猜想——Cookie失效问题

> B进程的Cookie可能失效了，导致A进程读取不到Cookie。

B进程的Cookie可能失效了，导致A进程读取不到Cookie。出现这个猜想的原因是我们使用chrome inspect查看Cookie时，它显示的时间的确是过期了的，比如刚才访问的`https://m.baidu.com`，

![baidu_cookie_expired](https://haitao.nos.netease.com/76f8ddab-c225-4be6-9c3e-7bf8d1526c67.png)

有一条Cookie的时间表示为`2019-04-28T05:38:12.000Z`，但是注意到时间最后的字母`Z`，它表示的是GMT/UTC时间里的GMT+0时区[^2]。转换成北京时间(GMT+8)后，就是下午1点38分。

![gmt_time_converter](https://haitao.nos.netease.com/27849e64-0a7a-4e07-9c36-8469e4b791bf.png)

说明这条Cookie还是有效的，排除了由于Cookie失效导致A进程访问不到的可能。另外，在Android中，即使Cookie已经失效，也能够通过`CookieManager.getInstance().getCookie(url)`取得，并且该方法返回一个字符串，不包含Cookie的`Expires`字段。第四个猜想不成立。

## 第五个猜想——Cookie文件进程读取

> A进程和B进程的Cookie文件根本不是同一个，导致数据无法同步。

A进程和B进程的Cookie文件根本不是同一个，导致数据无法同步。经过上面的猜想和实验，其实可以说明这个猜想是不成立的，如果进程读取的Cookie文件不是同一个的话，那么在B进程访问`https://m.baidu.com`后，A进程不可能拿到B进程的WebView写入的Cookie，测试二的结论说明了这一点。为了让事实更具有说服力，还是以实验说明这一点。

> 实验步骤：  
> 1、B进程访问`https://m.baidu.com`；  
> 2、保存Cookie文件的最后修改时间；  
> 3、A进程再次访问`https://m.baidu.com`(或者别的url也可以)；  
> 4、查看Cookie文件的最后修改时间并与步骤二的进行比对。

我们分别在14:06的B进程和14:08的A进程访问了`https://m.baidu.com`，结果如下：

![](https://haitao.nos.netease.com/d924081c-28d5-4db1-a8ed-088ebaa5f3cd.png)

说明**App里的不同进程使用的是同一个Cookie文件进行读取和写入**。第五个猜想不成立。

## 第六个猜想——Cookie同进程同域访问

> A进程创建了WebView并且访问了同域的url，然后覆盖了B进程之前已经持久化的Cookie

由第五个猜想的实验结果可知，不同进程间是使用同一个Cookie文件进行持久化。如果A进程和B进程都允许写Cookie，那么进程间就可能产生Cookie覆盖的现象。我们可以测试一下。

> 实验步骤：  
> 1、使用B进程WebView打开`https://m.baidu.com`，记录当前的Cookie文件；  
> 2、使用A进程WebView打开`https://m.baidu.com`，记录当前的Cookie文件；  
> 3、对第一步和第二步的Cookie文件进行对比。

![baidu_cookie_b](https://haitao.nos.netease.com/eb5b327a-1e0f-4de8-b354-a15521d696b8.png)
（B进程访问`https://m.baidu.com`）

![baidu_cookie_a](https://haitao.nos.netease.com/641a8a8b-89f5-445a-8935-a933aec9bd07.png)
（A进程访问`https://m.baidu.com`）

从图中可以看到，B进程访问url后的Cookie和A进程访问url后的Cookie数据几乎是一致的，只有一列不一样——`last_access_utc`。我们猜测这个字段表示上一次成功读取/写入该Cookie的时间（没有找到相关的文档介绍），但至少说明Cookies这个文件发生了覆盖，也就是说，**App里的不同进程对同一个域访问，可能会造成Cookie覆盖**。

即便如此，到目前为止，还没有能够解释B进程的部分Cookie在A进程获取不到的现象。

## 第七个猜想——CookieManager的锅

> Cookie是通过CookieManager管理的，CookieManager是个单例，单个进程可能只会读取一次Cookies.db，然后缓存在内存中。

Android中所有与Cookie的操作都与CookieManager有关，上面的几种猜想都没有考虑到CookieManager的问题，CookieManager是一个单例，一旦创建，除非进程被清除，否则便不会销毁。如果说CookieManager只有在创建时才读取一次Cookies.db文件，后面对Cookie的读取优先使用内存中的缓存，那么上面的现象便可以解释得通了。还是通过实验来验证。

> 实验步骤：  
> 1、A进程未初始化CookieManager的情况下，使用进程B访问`https://m.baidu.com`，Cookie持久化后，然后分别在初始化A进程的CookieManager前后，查看A进程的Cookie情况；接着再使用进程B访问`https://m.taobao.com`，Cookie持久化后，再次查看A进程的Cookie情况。  
> 2、A进程未初始化CookieManager的情况下，使用进程B访问`https://m.baidu.com`和`https://m.taobao.com`，Cookie持久化后，初始化A进程的CookieManager，并查看A进程的Cookie情况。

![cookie_manager_problem](https://haitao.nos.netease.com/6481f12e-a5f1-41e7-ae75-fc94c63265cf.png)

结果证实了猜想！CookieManager在未初始化时取不到`m.baidu.com`的Cookie，一旦初始化了CookieManager，则能够取到`m.baidu.com`的Cookie。但步骤二再一次说明，**只要初始化了CookieManager，那么该进程的Cookie再也取不到其他进程更新后的Cookie信息**。

# 多进程Cookie同步结论

至此，多进程下Cookie同步问题的猜想全部验证完毕了，可以得出的结论是——Cookie在多进程间的获取只和第一次初始化CookieManager有关系，一旦CookieManager实例创建，则需要重启进程才能同步进程间的Cookie。

回到本文遇到的问题，既然问题的原因已经找到了，那么肯定有解决办法。一种不完美的方案是先启动B进程并加载url，等到加载完成即将跳转到App页面的时候通知主进程初始化CookieManager，这样便可以取到url中指定的Cookie信息。这种方案的缺点是再次访问这个url写入新的指定Cookie时不会立即同步到主进程，需要等到App重启主进程以后才会同步；另外一种解决方案是把WebView和App都放在主进程即可。本文最终由于没有能够完美解决多进程Cookie同步方案，因此采用了第二种方案。

# 参考链接

1. <http://www.cnblogs.com/zhangyoushugz/archive/2012/12/01/2797647.html>
2. <https://stackoverflow.com/questions/19112357/java-simpledateformatyyyy-mm-ddthhmmssz-gives-timezone-as-ist>

[^1]: http://www.cnblogs.com/zhangyoushugz/archive/2012/12/01/2797647.html
[^2]: https://stackoverflow.com/questions/19112357/java-simpledateformatyyyy-mm-ddthhmmssz-gives-timezone-as-ist