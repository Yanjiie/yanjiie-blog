---
title: 在 Weex 中使用国际化
subtitle: "几种实现 Weex 国际化的方式"
date: 2018-01-12 10:35:30
author: "蓝小胖"
tags: 
  - Weex
---

# 在 weex 中使用国际化

Weex 是一套使用 `Vue.js` 作为上层框架的简单易用的跨平台开发方案，遵循 W3C 标准实现了统一的 `JSEngine` 和 `DOM API`，能使用一套代码打造 Web、Android、iOS 三个平台的原生应用。

对于面向多地区用户的 App 来说，应用的国际化是不可缺少的需求，在 `Android` 和 `iOS` 平台中，都拥有成熟的国际化方案。

而目前，Weex 团队并没有提供国际化相关的方案，但幸运的是，Weex 使用了 `Vue.js` 作为上层框架，我们可以在 `Vue.js` 生态中寻找相关的解决方案。经过一段时间了解，寻找到了一些国际化的解决方案，结合了 `Weex` 的一些特性，设想了一些可选择的国际化提案。

<!-- more -->

本文分下面几部分：
    
1. Vue.js 国际化方案 -- Vue-i18n
2. 基于语言文件引入的国际化方案
3. 基于 Weex 的 `JS Service` 实现国际化的方案

# Vue.js 国际化方案

众所周知，Vue.js 以其文档和生态的完整性著称。发展得如火如荼的 Vue 社区中，不乏优秀的开源作者。其中 [Vue-i18n](https://github.com/kazupon/vue-i18n) 是一款颇有人气的开源国际化插件，目前在 github 中 star 数已经 1.9k，许多国内外项目都兼容了这款插件，其中也包括 [ElementUI](http://element.eleme.io/#/zh-CN)。

Vue-i18n 是以插件的形式配合 Vue 进行工作的。通过全局的 mixin 的方式将插件提供的方法挂载到Vue的实例上。

## Vue-i18n 的基本使用

### 安装：

```bash
$ npm install vue-i18n -D
```

### 使用：

**入口文件 index.js**:

```javascript
import VueI18n from 'vue-i18n'
var App = require('./index.vue')

Vue.use(VueI18n)

// 国际化的内容
const messages = {
  en: {
    message: {
      'hello': 'hello world'
    }
  },
  ja: {
    message: {
      'hello': 'こんにちは、世界'
    }
  }
}

// 设置参数，创建 Vuei18n 的实例。
const i18n = new VueI18n({
  locale: 'ja', // set locale
  messages, // set locale messages
})

// 使用 Vuei18n 的实例 i18n，创建 Vue 的实例
new Vue({
  i18n,
  el: '#root',
  render: h => h(App)
})

```

**.Vue 单文件中**:

```html
<template>
  <div class="wrapper">
    <text class="title">{{ $t("message.hello") }}</text>
  </div>
</template>
```

**切换语言**:

```javascript
export default {
    data: () => ({}),
    methods: {
        changeLanguage() {
            this.$i18n.locale = 'cn'
        }
    }
}
```

## Vue-i18n 在 Weex 中使用

Vue-i18n 作为目前 Vue SPA 的国际化首选插件，**优点**显而易见：

1. 使用方便快捷
2. 单次注入，全局使用
3. 国际化内容可动态加载与语言切换
4. 还支持单文件组件以块的形式定义国际化内容：

    ```html
    <!-- 需要 @kazupon/vue-i18n-loader 支持 -->
    <i18n>
    {
      "en": {
        "hello": "hello world!"
      },
      "ja": {
        "hello": "こんにちは、世界！"
      }
    }
    </i18n>
    ```

**缺点**:

由于 Weex 仅使用 Vue.js 的 runtime 版本作为上层框架，在 webpack 打包的过程中，H5 和 Native 所使用的 loader 并不相同。

H5 使用的是 Vue-loader，可以通过配置 options 来自定义单文件标签库内容的编译。使用方法： [Custom Block](https://vue-loader.vuejs.org/en/configurations/custom-blocks.html)

而

> Native 使用的是 weex-loader，目前不支持自定义块的编译。

因为 loader 的不同，目前 Weex Native 不支持在单文件组件内使用块形式使用 Vue-i18n，所以国际化不能就近管理。

除了因为 loader 的不同，Vue-i18n 的部分功能受到了限制，Vue-i18n 在 Weex 中使用还存在其他的问题。

在使用 Vue 开发的 Web App 中，大部分可能都以 SPA 的形式开发，通过单一的入口结合 Router 进行多页面的构建，使得公共的 JS 可以做到一次加载，多页使用。但是在 Weex 的官方文档[《Vue 2.x 在 Weex 和 Web 中的差异》](https://weex.apache.org/cn/references/vue/difference-with-web.html)一文中：

> Weex 在原生端使用的是“多页”的实现，不同的 js bundle 将会在不同的原生页面中执行；也就是说，不同的 js bundle 之间将不同共享 js 变量。即使是 Vue 这个变量，在不同页面中也对应了不同的引用。

也就是说 Weex 官方推荐使用多页的形式开发 Weex 应用。虽然 Weex 中也支持 Vue-Router 的使用，但事实上经过项目的考证，在 Weex 中使用多页的形式开发，随着页面的不断增加，单个 ViewController 或 Activity 所使用的内存会变得越来越多，最终导致应用崩溃，而且页面跳转也不支持动画过渡，造成的用户体验也不好。

由于 Weex 使用多页开发的这个约束，使得 Vue 现有的插件均不能做到一次引入，全局使用。同样的，如果要在 Weex 中使用 Vue-i18n，需要在每一个页面的入口都在 Vue 实例中注册 VueI8N 对象，并且引入相关代码库。

经过打包测试

> 在 Weex 中引入 Vue-i18n，将使最终生成的 JS Bundle 增大 22KB 左右

22KB 无论对于本地还是网络加载 JS Bundle 来说，影响其实都微乎其微。但由于国际化资源文件不能按需加载，在打包时就需要将所有国际化资源引入，而这个缺点，才是增大 JS Bundle 的罪魁祸首。

# 基于 Weex 的 JS Service 实现国际化（待续）

