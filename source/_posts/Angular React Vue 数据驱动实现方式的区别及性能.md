---
layout: post
title: "Angular React Vue 数据驱动实现方式的区别及性能"
subtitle: "记一次组内分享"
date: 2017-12-29 15:05:44
author: "蓝小胖"
tags:
	- JavaScript
---

# Angular.js 1 的数据驱动实现方式：
## Angular 使用的是脏检查( Dirty Check )机制。

Angular 视图模板里,读取数据有一个叫 `$scope` 的上下文,可以就认为它是一个存储数据的JS对象, 里面放了各种视图模板需要访问的数据/函数。`$scope` 里面的值可以自由更改 (比如说在 Angular 的控制器(Controller)里面)。

比如说有这样一常简单的 Angular 应用 (模板中引用了 `$scope` 里的 `title` 数据, 同时绑定了 `onclick` 事件修改 `title` 数据):

```html
<script>
    var testApp = angular.module('testApp', []);
    testApp.controller('TestController', function ($scope) {
        $scope.title = 'test title';
        $scope.onDIVClick = function () {
            $scope.title = 'another title';
        };
    });
</script>
<div ng-app="testApp" ng-controller="TestController" ng-click="onDIVClick()">
    {{title}}  <!-- 显示为: test title -->
</div>
```

<!-- more -->

然后点击 div 触发事件, 就会调用 `$scope.onDIVClick`, 修改了 `title` 数据, 然后视图就更新了 (内容变为了 `another title`)。

==这一步DOM更新是怎么做到的?==

答案是, Angular 不监听数据更新, 数据发生任何改变时 Angular 都不理睬, 它只是找了一个恰当的时机, 遍历所有的DOM更新方法, 从被修改过任意次的 `$scope` 数据中尝试更新DO (这里这个"恰当的时机"就是click事件处理结束时)。

**Angular 脏检查实现**

- 首先, `$watch`: Angular 在解析视图模板时, 会找出其中的数据绑定, 以及对应的更新DOM的方式, (比如说这里的 `{{title}}`, 解析出值表达式为 `$scope.title`, 更新DOM方式为添加表达式的结果到文字内容区), 然后通过 `$scope.$watch` 将这一绑定注册到当前 `$scope` 上下文的更新响应操作里
- 然后, `$apply`: 可能更新数据时(比如事件响应函数里), Angular 调用 `$scope.$apply(expression)` 处理操作函数, `$apply(...)` 会在处理完成后调用 `$scope.$digest()`
- `$digest`: 在这一函数里, Angular 正式执行数据到视图的查找以及更新操作:
    - `$digest` 每一个循环里会从根作用域( `$rootScope` )开始(以深度优先方式)遍历所有的 `$scope` 注册的 `$watch` 响应操作。对每一个 `$watch` 响应, 取出数据绑定的值表达式, 求出值, 与上一次的求值作比较, 求值不一样 则取出DOM更新函数, 更新视图
    - 上一步里, 循环开始时置 `dirty = false`。只要有任何一个 `$watch` 响应的值发生了更新, 则当前 `$digest` 循环置 `dirty = true`
    - 每次循环结束后, 只要 `dirty === true` 依然成立, 重新开始新的一轮 `$digest` 检查循环, 直到 `dirty === false` (这就是为什么这个实现机制叫做脏检查( Dirty Check ))

**脏检查会执行几次?**

即便 $watch 响应里没有更新任何数据, 通常来说脏检查循环也会执行两次, 比如在这个例子里面就是的:

- 首先, 在执行 `$digest` 开始脏检查循环前, click事件触发调用的 `onDIVClick` 中已经更新了 `titl`e 数据
- 进入第一遍脏检查循环后, `{{title}}` 对应的 `$watch` 响应中, 值表达式之前的结果是 `'test title'`, 现在由于数据已经发生了改变, 新的结果变成了 `'another title'`, 第一遍循环, `dirty` 置为 `true`, 进入下一遍循环
- 进入第二便脏检查循环后, 值表达式两次结果均为 `'another title'`, 没有变化, `dirty` 为 `false`。结束脏检查

所以什么时候脏检查循环只会执行一遍呢? 就是 `$apply(...)` 处理的操作中没有数据更新操作 (这里描述并不准确, 实际上是否有数据更新跟脏循环执行次数不一定相关, 等会再说)。也就是说, 移除 `onDIVClick` 里的 `title` 赋值操作, 脏检查循环就只会执行一次。

那么, 什么时候脏检查会执行无数次呢? 很简单, 在 `$watch` 响应的值表达式中中每次都返回新的数据。

==推荐阅读：==
1. [Angular 脏检查机制研究](http://blog.yunfei.me/blog/angular-dirty-check.html)
2. [Angular沉思录（一）数据绑定](https://github.com/xufei/blog/issues/10)

# Vue.js 的数据驱动实现方式：
## Vue 使用的是依赖收集。（基础 —— getter/setter）

同样是实现了双向绑定，但 Vue 使用的方法与 Angular 却完全不同。Vue 的文档中是这样描述的：

> 把一个普通对象传给 Vue 实例作为它的 data 选项，Vue.js 将遍历它的属性，用 Object.defineProperty 将它们转为 getter/setter。这是 ES5 特性，不能打补丁实现，这便是为什么 Vue.js 不支持 IE8 及更低版本。

`getter/setter` 使得开发者有机会在对象属性取值和赋值的时候进行自定义操作，响应系统便是基于这个特性实现的。

### Watcher

`Watcher` 是 Vue 核心。每个 `Watcher` 都拥有一个自己的表达式，`Watcher` 的作用就是维护这个表达式依赖的数据项，并在数据项更新的时候更新表达式的值。其实从 Vue 的实现来讲，`Watcher` 应该被叫做 `Listener `或 `Subscriber` 更加合适。

> 模板中每个指令/数据绑定都有一个对应的 watcher 对象，在计算过程中它把属性记录为依赖。之后当依赖的 setter 被调用时，会触发 watcher 重新计算 ，也就会导致它的关联指令更新 DOM。
![](http://cn.vuejs.org/images/data.png)

### 依赖收集

依赖收集是通过 `property` 的 `getter` 完成的，依赖收集的过程涉及到 Vue 的三类对象：`Watcher`、`Dep` 和 `Observer`。其中 `Observer` 负责将数据项转化为响应式对象，而 `Dep` 则用来描述 `Watcher` 和 `Observer` 的依赖关系。

Vue 中，每个 `Observer` 对应一个 `Dep` 对象，而每个 `Dep` 对象可以对应多个 `Watcher`，每个 `Watcher` 也可以对应多个 `Dep` 对象，从而实现了多对多的依赖关系。

搞清楚了三类对象，下面就来看看依赖关系到底是怎样建立起来的：

1. Vue 实例初始化的过程中，首先，每个数据项都会生成一个 `Observer`，每个 `Observer` 又会初始化一个 `Dep` 实例；
2. 接下来，模板中的每个指令和数据绑定都会生成一个 `Watcher` 实例，实例化的过程中，会计算这个 `Watcher` 对应表达式的值；
3. 计算开始之前，`Watcher` 会设置 `Dep` 的静态属性 `Dep.target` 指向其自身，开始依赖收集；
4. 计算表达式的过程中，该 `Watcher` 依赖的数据项会被访问，从而触发其 `getter` 中的代码；
5. 数据项 `getter` 中会判断 `Dep.target` 是否存在，若存在则将自身的 `Dep` 实例保存到 `Watcher` 的列表中，并在此 `Dep` 实例中注册 `Watcher` 为订阅者；
6. 重复上述过程直至 `Watcher` 计算结束，`Dep.target` 被清除，依赖收集完成；

在依赖关系建立后，每当数据项发生变化（`setter` 被访问），`Observer` 会调用其 `Dep` 实例的 `notify` 方法，在这个 `Dep` 实例中注册的 `Watcher` 将会被通知，并重新进行计算及依赖收集的过程，然后执行相应的回调函数。以上就是完成响应的整个过程。


==推荐阅读：==
1. [Vue官方文档--深入响应式原理](https://cn.vuejs.org/v2/guide/reactivity.html)
2. [Vue 响应式原理探析](https://zjy.name/archives/vue-reactive-study.html)
3. [深入浅出基于“依赖收集”的响应式原理](https://segmentfault.com/a/1190000011153487)


## React.js 的数据驱动实现方式
React 本质没有双向绑定概念，算法本质是 diff。通过 dom 的抽象化，在 render 时通过比较 vdom 的差异，再使用原生 api patch 到真实 dom 中。

