# 微信小程序性能优化

## 微信小程序官方推荐的优化策略

-   从启动性能方面
-   从渲染性能方面

## 1、启动性能优化

小程序在启动过程会分以下几步

-   1、准备运行的环境；这一步是微信内部处理，优化无法从这里下手
-   2、下载、注入、执行小程序代码包
-   3、渲染小程序首页

总结：由于第一步为微信内部处理，所以优化启动性能一般在第二和第三步

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
![分包](/imgs/img2.png)

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
