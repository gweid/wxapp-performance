# 微信小程序性能优化

![分包](/imgs/img5.png)



## 微信小程序官方推荐的优化策略

-   从启动性能方面，可参考微信开发者文档：[启动性能](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips/start.html)
-   从渲染性能方面，可参考微信开发者文档：[运行时新能（即渲染性能）](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips.html)



## 1、启动性能优化

小程序在启动过程会分以下几步

-   1、准备运行环境；这一步是微信内部处理，优化无法从这里下手
-   2、下载，注入并执行对应小程序代码包
-   3、渲染小程序首页

**总结：由于第一步为微信内部处理，所以优化启动性能一般在第二和第三步**



### 1-1、控制代码包大小



#### 1-1-1、图片使用 cdn

一般项目中图片等静态文件是影响体积的主要因素, 而小程序支持的体积不能超过 2M, 因此，对于静态资源，需要用 cdn 链接的形式。



#### 1-1-2、清除项目中没用的文件

在开发过程，遇到项目改版，肯定会有一些没用的废弃资源，这些要清理掉，以免影响包的体积。



#### 1-1-3、分包

分包的意义: 将小程序不常用或者首次加载不需要用到的页面放在多个分包里，主包值保留核心页面。这样子，启动时只加载主包，分包会在用到的时候按需下载，从而优化启动时下包的时间。

![分包](/imgs/img1.png)

小程序目录结构

```
├── app.js
├── app.json
├── app.wxss
├── packageA
│ └── pages
│ ├── cat
│ └── dog
├── pages
│ ├── index
│ └── logs
└── utils
```

使用分包: 在 app.json 中用 subpackages 字段声明项目分包结构

```
{
  "pages":[
    "pages/index",
    "pages/logs"
  ],
  "subpackages": [
    {
      "root": "packageA",
      "pages": [
        "pages/cat",
        "pages/dog"
      ]
    }, {
      "root": "packageB",
      "name": "pack2",
      "pages": [
        "pages/apple",
        "pages/banana"
      ]
    }
  ]
}


root: String               分包根目录
name: String               分包别名，在分包预下载时可以使用
pages: StringArray         分包页面路径，相对于分包目录
independent: Boolean       分包是否是独立分包
```

一些注意点:

-   tabBar 必须在主包
-   分包 A 无法引用分包 B 的 js、template、静态文件，但可以引用主包的



#### 1-1-3、独立分包

-   从独立分包中页面进入小程序时，不需要下载主包。因此独立分包不能依赖于依赖主包和其他分包中的内容, 包括 js 文件、template、wxss、自定义组件、插件等。并且，独立分包的 getApp 也不一定可以获得 App 对象
-   一般使用场景: 广告页、活动页、支付页等
-   在 app.json 的 subpackages 字段中对应的分包配置项中定义 independent 字段声明对应分包为独立分包

```
{
  "pages": [
    "pages/index",
    "pages/logs"
  ],
  "subpackages": [
    {
      "root": "moduleA",
      "pages": [
        "pages/rabbit",
        "pages/squirrel"
      ]
    }, {
      "root": "moduleB",
      "pages": [
        "pages/pear",
        "pages/pineapple"
      ],
      "independent": true  // 用于独立分包
    }
  ]
}
```



#### 1-1-4、分包预下载

这个就是进入小程序某个页面时，让框架自动下载可能需要的分包，提高进入后续分包页面时的启动速度，而不是等到需要进去分包页面的时候再下载

![分包预下载](/imgs/img2.png)

-   packages: StringArray 进入页面后预下载分包的 root 或 name。\_\_APP\_\_ 表示主包
-   network: String 在指定网络下预下载，可选值为 all: 不限网络 wifi: 仅 wifi 下预下载

```
├── app.js
├── app.json
├── app.wxss
├── packageA
│ └── pages
│ ├── cat
│ └── dog
├── pages
│ ├── index
│ └── logs
└── utils

{
  "pages": [
    "pages/index",
    "pages/logs"
  ],
  "subpackages": [
    {
      "root": "packageA",
      "pages": [
        "pages/cat",
        "pages/dog"
      ]
    }, {
      "root": "packageB",
      "name": "pack2",
      "pages": [
        "pages/apple",
        "pages/banana"
      ]
    }
  ],
  // 分包预下载
  "preloadRule": {
    "pages/index": {
      "network": "all",
      "packages": ["pack2"]
    },
    "pages/logs": {
      "packages": ["packageB"]
    }
  }
}

```



#### 1-1-5、**分包精细化**

这个策略主要是为了控制和减少主包的体积，使主包体积最优，防止一些主包未使用的资源被放入主包，而占用主包体积。

- 把只被分包使用的资源收集到分包里，比如图片，js模块，组件，**npm包** 等
- 为了确保主包体积最优，对于组件和静态资源，如果他们被多个分包所引用且未在主包中引用，可以将这些资源生成多份副本分别输出到对应分包中，而不会占用主包体积。



#### 1-1-6、小程序瘦身工具

使用 [小程序瘦身工具](https://github.com/wechat-miniprogram/miniprogram-slim)，通过剔除无用文件、压缩图片、复用代码等方式减少小程序代码包体积



### 1-2、首屏渲染性能优化



#### 1-2-1、提前首屏数据请求

大部分小程序在渲染首页时，需要依赖服务端的接口数据，小程序为开发者提供了提前发起数据请求的能力

-   [数据预拉取](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/pre-fetch.html)：能够在小程序冷启动的时候通过微信后台提前向第三方服务器拉取业务数据，当代码包加载完时可以更快地渲染页面，减少用户等待时间，从而提升小程序的打开速度。
-   [周期性更新](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/background-fetch.html)：在用户未打开小程序的情况下，也能从服务器提前拉取数据，当用户打开小程序时可以更快地渲染页面，减少用户等待时间



#### 1-2-2、缓存请求数据

通过 wx.setStorageSync 等异步读写本地缓存的能力，数据存储在本地，返回的会比网络请求快。优先从缓存中获取数据进行渲染，等网络请求回来再更新数据。



#### 1-2-3、精简首屏数数据

-   延迟请求非关键渲染数据，缩短网络请求时延
-   与视图层渲染无关的数据尽量不要放在 data 中，加快首屏渲染完成时间
-   接口返回的数据要做处理，不要直接塞给 data，要去除不需要的数据

```
data() {
    // 与视图层渲染无关的数据不放在 data 中
    this['otherData'] =  {
        height: 100
    }

    return {
        list: []
    }
}
```



#### 1-2-4、避免阻塞渲染

在小程序启动流程中，会顺序执行 app.onLaunch, app.onShow, page.onLoad, page.onShow, page.onReady，所以，尽量避免在这些生命周期中使用 Sync 结尾的同步 API



#### 1-2-5、避免首页白屏

使用 [骨架屏](https://github.com/jayZOU/skeleton) 或者加载交互功能，避免用户觉得小程序无响应



## 2、渲染性能优化

小程序的视图层目前使用 WebView 作为渲染载体，而逻辑层是由独立的 JavascriptCore 作为运行环境。在架构上，WebView 和 JavascriptCore 都是独立的模块，并不具备数据直接共享的通道。当前，视图层和逻辑层的数据传输，实际上通过两边提供的 evaluateJavascript  来实现。即用户传输的数据，需要将其转换为字符串形式传递，同时把转换后的数据内容拼接成一份 JS 脚本，再通过执行 JS 脚本的形式传递到两边独立环境

![小程序渲染机制](/imgs/img3.png)



### 2-1、避免频繁的去 setData

原因: 视图层目前使用 WebView 作为渲染载体, 逻辑层是由独立的 JavascriptCore 作为运行环境, 每次使用 setData 都需要两个进程之间通讯

频繁 setData 会造成

-   Android 下用户在滑动时会感觉到卡顿，操作反馈延迟严重，因为 JS 线程一直在编译执行渲染，未能及时将用户操作事件传递到逻辑层，逻辑层亦无法及时将操作处理结果及时传递到视图层；
-   渲染有出现延时，由于 WebView 的 JS 线程一直处于忙碌状态，逻辑层到页面层的通信耗时上升，视图层收到的数据消息时距离发出时间已经过去了几百毫秒，渲染的结果并不实时；

**总结：把 setData 操作合并**



### 2-2、避免每次 setData 都传递大量新数据

这个主要存在于长列表的渲染

-   每次 setData 都是进程间的通讯，通讯开销与 setData 的数据量相关

解决：

```
// 1.通过一个二维数组来存储数据
let feedList = [[array]];

// 2.维护一个页面变量值，加载完一次数据page++
let page = 1

// 3.页面每次滚动到底部，通过数据路径更新数据
onReachBottom:()=>{
    fetchNewData().then((newVal)=>{
        this.setData({
            ['feedList[' + (page - 1) + ']']: newVal,
        })
    }
}
// 4.最终我们的数据是[[array1],[array2]]这样的格式，然后通过wx:for遍历渲染数据

// wxml文件里做一个双重循环，达到增量渲染的效果，实测数万级列表也不会卡顿。
<block wx:for="{{List}}" wx:for-item="list-item">
    <block wx:for="{{list-item}}" wx:for-item="list">
      <view class="tab-content">
       {{list.content}}
      </view>
    </block>
</block>
```



### 2-3、避免后台状态进行 setData

当页面进入后台态（用户不可见），不应该继续去进行 setData，后台态页面的渲染用户是无法感受的，另外后台态页面去 setData 也会抢占前台页面的执行

![避免后台setData](/imgs/img4.png)



### 2-4、事件使用不当

-   避免不当使用 onPageScroll, 使用节流防抖限制频繁监听 onPageScroll
-   避免在 onPageScroll 频繁调用 setData、执行复杂逻辑



### 2-5、使用自定义组件

有些场景需要频繁更新，自定义组建的更新只在组件内部进行，不受页面其他部分内容影响。例如：9 块 9 活动倒计时
