---
layout:     post
title:      "基于 weex 的考拉移动端动态化方案"
subtitle:   "考拉 App 的动态化方案设计及经验总结"
date:       2018-01-15
author:     "杨鑫"
header-img: "img/bg5.jpg"
tags:
    -Android
    -Weex
---

### 目录：

1.为什么要使用热发布；

2.行业现状；

3.热发布整体设计方案；

4.上线功能和数据情况；

5.使用过程中遇到的问题；

6.之后需要做的事情；

### 一、为什么使用热发布

#### 1.实时性限制
考拉作为一个跨境电商类的App，从最开始就注定会受到政策类的条款限制，从而导致经常会出现一些实时变更的需求。而目前这些实时性的需求又必须通过App的直接发版本来解决，不仅发布周期长，应用市场的审核很浪费时间，而且用户升级率也不高。

#### 2.新功能依赖升级

当产品策划提到一些新功能场景时，都必须通过一个完整的迭代流程来进行开发，最终通过发布新版本来让用户使用到新的功能，开发和测试周期长。同时，用户如果想使用新功能，必须依赖升级。无法让旧版本的用户使用到新功能。

#### 3.Bug无法热修复

移动开发过程中经常会出现因为考虑不周导致的一些线上逻辑问题或者崩溃问题，一般情况下都会通过重新打包发布应用市场来解决，目前也有更方便的通过补丁来解决问题。通过动态化方案也可以更好的通过热发的方式来修复bug。简单高效。

#### 4.多端协作成本高

目前整个移动市场是分为android和ios，当需求提出的时候，会同时通知到android和ios两位开发，进行需求评审和功能迭代，在需求沟通和实现过程中总会出现各种沟通问题导致需求实现效果不统一，同时两位开发也会很浪费人力。通过动态化方案，两端可以共用同一套代码，来解决各自的逻辑，能够节省很多人力。

### 二、行业现状

目前比较流行的两种热发布解决方案是，facebook的React Native和阿里的weex。

#### 两种方案的比较：

相同点：

1.两者的思想是一样的，都是通过将js转换成Virtual-dom再映射到Native的对应view组件；

2.遵循 W3C 标准实现了统一的 JSEngine 和 DOM API，能够以web的方式（HTML＋CSS＋JS）来开发native页面和应用；

3.都能够完成三端渲染，以及功能模块的动态下发；

不同点：

1.上层DSL语言不同，前端流行的两大框架就是React和Vue，React Native使用的自家的React，而weex使用的是vue；

2.RN支持ios和Android两个平台，web端需要自己支持和兼容；Weex号称支持到三端，但是还是有很多的兼容任务；

3.RN在打包的时候会将基础解析部分和业务逻辑一起打包，所以js文件会比较大，而weex只是将业务部分打包，具体的解析过程是在sdk中；所以bundle会比较小；

使用感受：两种方案我们都进行过调研，发现RN在编写过程中要求还是比较高，没有vue上手快，而且RN更像是一门新语言，他为不同的端提供了不同的组件，而weex更希望这些工作由native做，在weex页面中三端使用都是相同的；而且当时RN提供的list组件性能并不是很好，weex提供的是RecyclerView性能相对好很多；
因此我们移动端动态化采用了weex的方案进行。其实这两种方案都是采用virtual－dom渲染的方式来实现的，下面简单介绍一下weex方案的原理.

#### weex的原理：

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e9351f0771?w=640&h=459&f=png&s=286901) 

上面这张图就是weex的原理图，首先最开始是通过上层的DSL语言来实现业务功能，实现完成后会通过一些构建方式来进行打包，通过打包可以将整个业务逻辑打包成一个jsBundle，jsBundle里面就包含了所实现的功能，以及可以被下层jsFramework识别的代码。

有了jsBundle之后就可以将它放在后台系统中，这个就是我们需要热发布的功能。

下面一层就是sdk做的事情，sdk在拿到jsBundle之后会通过jscore进行解析，转换成一个虚拟dom树，不同的端在各自的sdk中解析虚拟的dom树再渲染成各自端的组件进行页面的展示。了解了weex的功能原理之后，下面介绍下我们整体的热发布应用方案。

### 三、动态化方案整体设计

整体的动态化方案功能涉及到前端，后端和移动端，三端的功能需求实现，具体的结构图如下。

#### 1.整体流程的思维导图

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e9355fbb1d?w=697&h=436&f=png&s=67597) 

主要包括三大部分：

weex层，weex工程负责业务页面的编写，打包和发布；

后台层，热发布的后台负责配置相对应的功能页面，提供App检测更新的能接口和推送下发更新包的能力；

app层，App端负责检测更新，缓存Bundle，以及渲染展示的策略；

下面可以看下整体的热发布流程：

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e936ae3c90?w=546&h=245&f=png&s=19330) 

最开始是编写weex的代码实现功能需求，之后通过webpack打包构建jsBundle，将打包构建好的jsBundle发布到自己的服务器上，目前使用的是ndp平台和nginx服务器，在ndp平台配置好cdn规则，这样每一个js文件就会有一个单独的静态资源地址。之后会把这个静态地址和页面路由信息配置在我们的热发布后台。用户在使用App的时候就可以拿到热发的页面渲染。当然再App端也是有两种方式，一种是用户主动打开页面触发更新接口拉取新数据，另一种是后台发布后推送给App进行后台更新。
因为我们整个策略是分为三部分，所以下面详细介绍一下各个部分所做的工作。

##### a.weex页面开发部署流程

因为我本身是客户端开发，对前端的开发流程不甚了解，所以在开发过程中走了很多弯路。

其实前端已经有很多年的历史，很多开发工具都很完善，目前我在项目中使用的也是webpack来进行打包构建。

因为weex上层的DSL选择的是vue，所以具体需求页面的代码都建议采用vue来编写（遇到的问题可以参考vue的官方文档[https://vuejs.org/index.html](https://vuejs.org/index.html)，不过weex由于是在客户端渲染，所以有些vue中俄功能无法使用到。

代码编写不是重点，重点说下webpack打包时候的配置，我目前是在package.json中配置了两种打包方案。区分了开发和生产环境。在生产环境下可以对js文件进行压缩和混淆。

通过webpack在build文件的入口处可以设置一些公共的组件，减少vue代码中的引入：


    const App = require("${relativePath}")
    //全局注册组件
    Vue.component('wrapperLayout', require("components/wrapper-layout.vue"))
	
    App.el = '#root'
    new Vue(App)

我们这里配置了一个通用的页面框架，包括导航条和网络请求loading效果。

在vue文件build的过程中要设置"weex-loader":

    let weexConfig = getBaseConfig()
    weexConfig.output.filename = 'weex/[name].js'
    weexConfig.module.loaders[1].loaders.push('weex-loader')

    module.exports = [weexConfig]

build完成之后会生成对应页面的js文件。与java不同，这些js文件会将引入的组件和方法全部包含进来，所以一个页面只有一个js文件，这样也就导致了这个js文件可能会比较大。目前来看，我们开发的复杂页面最大在2-300k上下。

打包完成后会将所有的js文件和资源文件通过构建平台部署在negix服务器上，并且关联到cdn，这样在页面设置的时候可以直接使用cdn的链接。

![image](https://user-gold-cdn.xitu.io/2018/1/8/160d55e937ce2395?w=709&h=648&f=png&s=32106)

##### b.资源图片处理过程

资源图片目前了解到的可以采用五种方式：local,base64,CDN,iconfont,SVG。目前我们采用的主要是base64和CDN的方式，而前端活动页采用的方式是iconfont。

#### 2.后台接口和数据库设计

##### a.后台数据库表结构设计

后台数据库中存放bundle信息的表结构参数为：

bundleId，minSupportVersion，bundleVersion，bundleModule，fileDownloadUrl，loadType，desc，person，time

bundleId：当前页面的唯一标识；

minSupportVersion：当前页面最低可以支持到的App版本号；

bundleVersion：当前bundle的发布版本；

bundleModule：当前bundle所属模块；

fileDownLoadUrl：bundle的下载地址；

loadType：当前bundle页面可以使用的加载方式，1：网络检查，先使用本地，二次加载最新；2：网络检查更新，当此即加载，使用最新；

desc：当前bundle更新情况的的描述信息；

person：更新者；

time：更新和上传时间；

数据库中的表结构为：

bundleId | minSupportVersion | bundleVersion |bundleModule| fileDownloadUrl | loadType| desc|person| time
---|---|---|---|---|---|---|---|---
recommendDetail | 3.9 | 1 |seeding| http://gala/3.9/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-16

** 其中bundleId和minSupportVersion一起作为唯一标识 **；一旦两个有一个有变化，就做为一条新的数据

** 数据库的操作有两种情况： **

** 1.native无改动 **

某次更新的内容，并不依赖于WeexSdk和Native的更改，所以之前的所有版本都支持，则后台接口直接数据库中根据bundleId和appVersion查询到当前这条数据，更新bundleVersion(加一)，替换fileDownloadUrl为新的地址即可。

例如：原来数据库中存的是：

bundleId | minSupportVersion | bundleVersion |bundleModule| fileDownloadUrl | loadType| desc|person| time
---|---|---|---|---|---|---|---|---
recommendDetail | 3.9 | 1 |seeding| http://gala/3.9/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-16

则更新成：

bundleId | minSupportVersion | bundleVersion |bundleModule| fileDownloadUrl | loadType| desc|person| time
---|---|---|---|---|---|---|---|---
recommendDetail | 3.9 | 2 |seeding| http://gala/3.9/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-16

只是文件和版本变了，其他都不变；这样3.9和4.0版本的App都可以使用此次开发的新功能。

** 2.native有改动 **

之前的某个版本在数据库中有一条当前页面的数据。新版本这次更新的内容，最新版本才可以用，之前所有的旧版本都无法使用，则在数据库中插入一条新数据。

例如：原来数据库中存的是：

bundleId | minSupportVersion | bundleVersion |bundleModule| fileDownloadUrl | loadType| desc|person| time
---|---|---|---|---|---|---|---|---
recommendDetail | 3.9 | 1 | seeding|http://gala/3.9/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-15

则更新成：

bundleId | minSupportVersion | bundleVersion |bundleModule| fileDownloadUrl | loadType| desc|person| time
---|---|---|---|---|---|---|---|---
recommendDetail | 3.9 | 1 | seeding| http://gala/3.9/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-15
recommendDetail | 4.0 | 1 | seeding| http://gala/4.0/recommend/recommenddetail.js | 1| 更新了xxx,changeLog... | 张三 | 2017-10-16

此次更新之后，数据库中针对recommendDetail的bundleId会存在两条信息，第一条给3.9到4.0版本之间的App使用，第二条给4.0版本之后的App使用。

##### b.后台接口设计

app中检测更新的接口：

requestParams：bundleId，appVersion；

后台接口需根据bundleId，appVersion在数据库中查询得到不大于appVersion的minSupportVersion最大的一条数据返回；

response：

	{
    	"bundleId":"recommendDetail",
    	"minSupportVersion":3.9,
    	"bundleVersion":1,
    	"fileDownloadUrl":"http://gala/3.9/recommend/recommendDetail.js",
    	"loadType":1
	}

例如：	
1.如果当前App的版本是3.9，

针对上面的第一种情况，后台要返回给我最新的bundleVersoin为2的recommendDetail.js文件

针对第二种情况，后台要返回给我3.9版本可用的recommendDetail.js文件，

2.如果当前App的版本是4.0，

针对上面的第一种情况，后台要返回给我minSupportVersion为3.9，

针对第二种情况，后台要返回给我4.0版本可用的recommendDetail.js文件。

##### c.后台配置管理系统

![image](https://user-gold-cdn.xitu.io/2018/1/8/160d55e938ce39fd?w=791&h=440&f=png&s=17430)

#### 3.App端更新展示策略

##### a.更新逻辑整体架构

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e9358884e0?w=616&h=342&f=png&s=28748) 

通过热发后台我们可以根据对应的bundleId拿到需要显示的Bundle信息，那么我们的native页面又是如何知道要用那个bundleId去请求，以及显示的呢？下面看下我们App端的策略，这个图是我们整个App端的架构，其中又分为两大部分，一部分是路由策略，一部分是检测更新策略。

###### 路由策略

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e96d60af34?w=804&h=415&f=png&s=39987) 

整个weex页面都是基于路由功能来进行展示和跳转的。也就是说每个weex页面都会有一个唯一的uri。举一个例子，比如在App中的A页面，这时候点了一个按钮跳转到B页面，而B页面是一个weex页面，那么A就会将B的路由信息传递给Router，Router会解析这个路由信息将他对应的bundleId传给weexVersionManager，即weex版本管理器，在管理器里拿到bundle进行显示，如果B页面再出现了跳转逻辑，就会通过weex和Native的桥接器再将路由信息传递给Router进行下一次跳转。

###### 显示更新策略

 ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e96e6381b9?w=685&h=401&f=png&s=52287) 

上面这部分就是Bundle加载和检测更新的部分，具体流程可以看下下面的流程图，这里主要分为两部分，一部分是显示策略，一部分是检测更新策略。
先看一下显示的流程，根据之前的流程，路由中将页面对应的bundleId传给VersionManager，versionManager在拿到bundleId之后会先在本地检查是否有可以使用的缓存，如果有缓存会先取缓存来加载，如果本地没有缓存，则会去热发布后台请求最新的bundle，请求回来之后再判断是否需 要刷新当前页面来进行显示。如果网络失败了也会给用户一个网络错误的提示。
右边这部分就是检测更新的流程，检测更新的过程是在后台执行的，不会影响用户前台的交互。在开始检测的时候会去后台查询是否有新的bundle可用，拿到返回的数据后会判断当前这个bundle是否已经下载过了，如果已经下载过了说明本地就是最新的，则不会再做其他事情，也不会影响用户的使用。如果 本地没有下载过，说明需要更新和覆盖之前的缓存，那么就会去下载这个jsBundle，下载成功后将jsBUndle缓存起来。再根据之前协议规则中设置的loadType进行区分。之前提到了loadType是分为当次加载和二次加载，如果是当次加载会直接刷新页面，如果是二次加载则等到用户第二次进入 的时候才会使用这个最新的。

### 四、线上页面及数据情况

在考拉AndroidApp中，目前weex页面在考拉App中的占比达到了10%。目前上线的功能是种草社区和小考拉问答绝大部分页面。如下图：

  ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e96be92f40?w=330&h=587&f=png&s=111908)   ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e96e9a9387?w=330&h=587&f=png&s=115825)   ![Alt pic](https://user-gold-cdn.xitu.io/2018/1/8/160d55e9725eb43f?w=330&h=587&f=png&s=76447) 


性能情况：整个页面的渲染时长可以维持在1秒内。其中下载js时长是:100-200ms。正常页面渲染时长在150-300ms左右

### 五、一些经验总结

1.关于.9图片的使用：
在img标签里可以使用.9图片，但是要将resize属性设置为stretch，如下：
```
<img class="comment-reply-bg" src="local: //res/bg_comment_reply" resize="stretch"/>
```

2.关于class样式切换：
```
<div class="focus-common" :class="[focusStatus?'focused-container':'focus-container']" >
```

3.style样式切换：
```
:style="{color: focusStatus?'#999999':'#8CAA5E'}"`
```
4.父组件调用子组件的方法和data：

在子组件中设置ref即id，在父组件中即可通过ref来获取到对应的子view调用子view的方法：
```
父vue：
<recommendDetailPage ref="detailPage"></recommendDetailPage>
this.$refs.detailPage.setData(this.$refs.detailPage[0].recommendDetail)
```
5.子组件给父组件回调：
```
//子组件通过emit通知父组件，event中传参数

showGoBack(event) {
    this.$emit('showgoback',event);
},

//父组件使用子组件时，html中声明这个方法，js中调用方法

<bridgeWebview :loadUrl="url" class="webview_style" @showgoback="showGoBack"></bridgeWebview>

showGoBack(event) {
    console.log("event.canGoBack2:"+event.canGoBack);
},
```
6.text显示问题

text标签中的内容不能换行，否则在手机上显示会出现换行；下面代码中第二个会显示两行。例如：
```
1. <text >hello world</text> 

2. <text >hello world
    </text>
```
### 六、之后还需要做的事情

1.三端共建，提高效率
我们考拉移动端目前也在三端通用性上进行了进一步的调研和实现。因为ios和android两端之前的底层支持情况不一样，导致在共建底层接口和共建组件，以及沟通上会花费一些时间。之后的目的主要是可以完善底层的共建框架，这样上层的业务才可以在三端通用，提高开发效率。

2.组件weex的热发布
目前我的开发的weex都是页面级别的，后续会在组件级别上进行一些尝试。并且抽离这些公共组件，打包成npm包，可以在多个项目中共用。
