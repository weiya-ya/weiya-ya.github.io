---
title: vue
date: 2018-2-12 10:20:45
categories: 
    - 前端
tags: 
    - vue axios
cover: http://img4.imgtn.bdimg.com/it/u=408489464,4062564120&fm=26&gp=0.jpg
---

### 组件data为什么必须是函数？

因为组件可能被多处使用，但它们的data是私有的，所以每个组件都要return一个新的data对象，如果共享data，修改其中一个会影响其他组件

### 组件通信

- 父子组件通信：`$on`、`$emit`
- 非父子组件的通信: event bus
- 复杂情况： vuex

### 怎么动态添加组件

场景：在vue中，点击button，随机生成a、b、c组件中的一个

- `is`
- `render`

思路：设定一个components数组，button点击一次，push一个组件名，`v-for`遍历components，并用`is`或`render`动态生成

### vue-loader是什么？

vue-loader 是一个 webpack 的 loader，可以将单文件组件转换为 JavaScript 模块

引用文档的说法：

- 默认支持 `ES2015`；
- 允许对 Vue 组件的组成部分使用其它 `webpack loader`，比如对 `<style>` 使用 `Sass` 和对 `<template>` 使用 `Jade`；
- `.vue` 文件中允许自定义节点，然后使用自定义的 loader 进行处理；
- 把 `<style>` 和 `<template>` 中的静态资源当作模块来对待，并使用 `webpack loader` 进行处理；
- 对每个组件模拟出 CSS 作用域；
- 支持开发期组件的热重载。

### 实现 Vue SSR基本原理

主要通过`vue-server-renderer`将Vue组件输出成HTML，过程：

1. 客户端 entry-client 主要作用挂载到 DOM 上，服务端 entry-server 除了创建和返回实例，还进行路由匹配与数据预获取
2. webpack打包客户端为client-bundle，打包服务端为server-bundle
3. 服务器接收请求，根据 url 来加载相应组件，然后生成 html 发送给客户端
4. 客户端激活， Vue 在浏览器端接管由服务端发送的静态 HTML，使其变为由 Vue 管理的动态 DOM，为确保混合成功，客户端与服务器端需要共享同一套数据。在服务端，可以在渲染之前获取数据，填充到 stroe 里，这样，在客户端挂载到 DOM 之前，可以直接从 store 里取数据。首屏的动态数据通过 window.**INITIAL_STATE** 发送到客户端

### 数据双向绑定原理

实现数据绑定的常见做法：

- `Object.defineProperty`：劫持各个属性的`setter`，`getter`
- 脏值检测：通过特定事件进行轮循
- 发布/订阅模式：通过消息发布并将消息进行订阅

vue采用的是数据劫持结合发布者-订阅者模式的方式，通过`Object.defineProperty()`来实现对属性的劫持，并在数据变动时发布消息给订阅者，使其触发相应的监听回调。

具体步骤：

1、 实现Observer

将需要observe的数据对象进行递归遍历，包括子属性对象的属性，都加上`setter`和`getter`。实现一个消息订阅器，维护一个数组，用来收集订阅者，数据变动触发notify，再调用订阅者的update方法

2、 实现Compile

compile解析模板指令，将模板中的变量替换成数据，然后初始化渲染页面视图，并将每个指令对应的节点绑定更新函数，添加监听数据的订阅者，一旦数据有变动，收到通知，更新视图

3、 实现Watcher

Watcher订阅者是Observer和Compile之间通信的桥梁

主要做的事情是:

- 在自身实例化时往属性订阅器(dep)里面添加自己
- 自身必须有一个update()方法
- 待属性变动dep.notice()通知时，能调用自身的update()方法，并触发Compile中绑定的回调，则功成身退。

4、 实现MVVM

MVVM作为数据绑定的入口，整合Observer、Compile和Watcher三者，通过Observer来监听自己的model数据变化，通过Compile来解析编译模板指令，最终利用Watcher搭起Observer和Compile之间的通信桥梁，达到数据变化 -> 视图更新；视图交互变化(input) -> 数据model变更的双向绑定效果

参考：[剖析Vue原理&实现双向绑定MVVM](https://link.juejin.im?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000006599500%23articleHeader4)

### 对Vue.js的template编译的理解

template会被编译成AST语法树，AST会经过generate得到render函数，render的返回值是VNode，VNode是Vue的虚拟DOM节点

- parse 过程，将 template 利用正则转化成 AST 抽象语法树。
- optimize 过程，标记静态节点，后 diff 过程跳过静态节点，提升性能。
- generate 过程，生成 render 字符串

司徒大佬有一篇很好的文章：[前端模板的原理与实现](https://link.juejin.im?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000006990480)

### vue 为什么采用`Virtual DOM`？

一方面是出于性能方面的考量：

- 创建真实DOM的代价高：真实的 DOM 节点 node 实现的属性很多，而 vnode 仅仅实现一些必要的属性，相比起来，创建一个 vnode 的成本比较低。
- 触发多次浏览器重绘及回流：使用 vnode ，相当于加了一个缓冲，让一次数据变动所带来的所有 node 变化，先在 vnode 中进行修改，然后 diff 之后对所有产生差异的节点集中一次对 DOM tree 进行修改，以减少浏览器的重绘及回流

但是性能受场景的影响是非常大的，不同的场景可能造成不同实现方案之间成倍的性能差距，所以依赖细粒度绑定及 `Virtual DOM`哪个的性能更好不是一个容易下定论的问题。更重要的原因是为了解耦`HTML`依赖，这带来两个非常重要的好处是：

- 不再依赖 HTML 解析器进行模版解析，可以进行更多的 AOT 工作提高运行时效率：通过模版 AOT 编译，Vue 的运行时体积可以进一步压缩，运行时效率可以进一步提升；
- 可以渲染到 DOM 以外的平台，实现 SSR、同构渲染这些高级特性，Weex 等框架应用的就是这一特性。

综上，`Virtual DOM` 在性能上的收益并不是最主要的，更重要的是它使得 Vue 具备了现代框架应有的高级特性。

### diff算法

这部分比较复杂，不好懂，推荐一篇不错的文章： [解析vue2.0的diff算法](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Faooy%2Fblog%2Fissues%2F2)

### vue 和 react 区别

相同点:

- 都支持`SSR`
- 都有`Virtual DOM`
- 组件化开发
- 数据驱动
- ...

不同点:

- vue推荐的是使用 webpack + vue-loader 的单文件组件格式，React 推荐的做法是 JSX + inline style
- vue 的`Virtual DOM`是追踪每个组件的依赖关系，不会渲染整个组件树，react 每当应该状态被改变时，全部子组件都会 re-render
- ...


作者：Alvin_Liu链接：https://juejin.im/post/5ab2ff496fb9a028c06ab78f来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。