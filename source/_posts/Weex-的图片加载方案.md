---
title: Creator 图片的加载方案预想
date: 2017-11-24 18:45:13
author: "蓝小胖"
categories:
- 技术
---

## 前因

由于 Weex 不支持统一的本地图片加载，而且在三端中对于本地图片的加载方式都不相同。

<!-- more -->

比如：

- 在 H5 下需要将图片资源文件存放到域名下的指定静态目录
  可能是这样的：
  ```html
  <!-- 实际加载的是 http://[HostRoot]/assets/example.png -->
  <image src="/assets/example.png">
  ```

- 在安卓下引用本地图片资源
  可能是这样的：
  ```html
  <!-- 实际加载的是 file://[app_res_root]/assets/example.png -->
  <image src="/assets/example.png">
  ```
  
## 解决方案

### 通过静态服务器获取图片资源

所有的外部引用资源可以通过类似以下的形式引入

```html
<image :src="http://design.flyme.cn/example.png')">
```

**缺点**

当开发较绚丽的页面时，大量的外部图片引入会增加很多的网络请求，会降低页面的加载速度。

****

### 通过不同平台区分引入方式

对于不同平台的差异，可以通过自定义加载方法：

```javascript
let env = weex.config.env
function loadImg(url) {
    if (env.platform === 'Web') {
        return 'http://design.flyme.cn' + url
    } else if ( === 'Android') {
        return 'file://creator/res' + url
    }
}
```

html 的使用：

```html
<image :src="loadImg('/assets/example.png')">
```

**缺点**

这种引入方式是直接引入上下文中的图片。
在开发环境中进行开发时，在 H5 端的调试可以通过 `webpack` 打包到指定 件夹并启动本地服务器进行访问。
但 Android 中调试时，可能需要手动将新增的图片 copy 到 res  录中，或者通过本地服务器 ( 例如 `http://127.0.0.1/assets/example.png `) 的方式引入。这样的话，在图片加载方法中还需要针对 Android 下是否处于开发环境下作二层判断，这样就不值得了。

### 一个可行的解决方案

目前 `Creator` 已经支持 `Base64` 图片的显示，所以预想了以下方案

- 小尺寸图片（比如小于 200KB）可以通过 `webpack` 打包成 `Base64` 的格式
- 大尺寸的图片打包进资源目录。
- 大尺寸图片的使用通过区分开发和生产环境下的不同的 `publicPath` 来获取
- `Creator` 增加应用图片资源静态服务
- 执行生产环境代码上传部署的同事将图片打包上传

以下是实现方式
- webpack 配置 url-loader
  ```javascript
  // 超过指定大小的图片打包到 dist/assets 目录下 
  module: {
    rules: [{
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        loader: 'url-loader',
        options: {
            limit: 10000,
            name: utils.assetsPath('dist/assets/[name].[ext]')
        }
    }]
  }
  ```
  
- 配置 publicPath
  ```javascript
  // Creator 增加应用静态资源服务，开发环境下引用本地资源， 产环境下引用静态资源服务
  let devPath = 'http://localhost/assets'
  let proPath = 'http://[hostName]/[appName]/[version]/assets'
  output: {
      path: config.build.assetsRoot,
      filename: '[name].js',
      publicPath: process.env.NODE_ENV === 'production'
          ? proPath
          : devPath
  }
  ```
  
- 开发环境下按需将图片存放到项目中即可
- 生产环境下将所有引用到的图片打包到 `dist/assets` 下，执行 `deploy` 时同事将图片打包上传到 Creator 静态资源服务中，并对比程序版本进行区分。
  