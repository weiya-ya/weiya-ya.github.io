---
title: vue全家桶开发的小技巧
date: 2018-8-12 8:38:45
categories: 
    - 前端
tags: 
    - vue
cover: https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=1000306574,2054737931&fm=26&gp=0.jpg
---

### css的scoped属性

vue 为了防止 css 污染，当组件的 `<style>` 标签有 `scoped` 属性时，它的 css 只作用于当前组件中的元素。实现原理很简单，给当前组件中的每个标签都加上唯一的自定义属性：`data-v-唯一的属性`，然后 css 选择器都加上属性选择器`.article-title[data-v-唯一的属性]`，这样这个 css 只会匹配到当前页面的这个元素。

**注意**：每个组件的最外层的标签会带上父组件的`data-v-`属性，也就是这个标签会被父组件的样式匹配到，所以父组件尽量不要使用标签选择器，这个标签不要使用父组件中的 id 或者 class。



![css-scoped](https://user-gold-cdn.xitu.io/2019/9/26/16d6cdc562c60ecd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



在父组件想修改子组件的css（修改elementUI组件的样式），我们可以借助深度作用选择器 `>>>`

```
div >>> .el-input{
    width: 100px;
}
/* sass/less的话可能无法识别，这时候需要使用 /deep/ 选择器。 */
div /deep/ .el-input{
    width: 100px;
}
复制代码
```

深度作用选择器会去掉后面元素的属性选择器`[data-v-]`，即上面代码会编译成：`div[data-v-12345667] .el-input{}`。就可以匹配到子组件的元素，从而覆盖样式。

### 父子组件的生命周期钩子函数执行先后顺序

组件的生命周期钩子函数是到了某个生命周期点就会触发，而不是在这个钩子函数中进行生命周期，比如说DOM加载好了，就会触发`mounted` 钩子函数，所以在`created` 里面写一个延迟定时器，`mounted` 钩子不会等定时器执行。

各个周期钩子函数触发的时间点参考（图来源于网络）



![life](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1142" height="1280"></svg>)



关于父子组件的生命周期：不同的钩子函数有不同的表现。父组件的虚拟 DOM 先初始化好了（`beforeMount`），才会去初始化子组件的虚拟 DOM （`beforeMount`），而 `mounted` 事件，等价于 `window.onload`，子组件 DOM 没加载好，父组件 DOM 永远不可能加载好。所以基本生命周期钩子函数执行顺序是：父beforeCreate -> 父created -> 父beforeMount -> 子beforeCreate -> 子created -> 子beforeMount -> 子mounted -> 父mounted

父子组件的update和beforeUpdate执行先后顺序：数据修改+虚拟DOM准备好会触发beforeUpdate，换句话说beforeUpdate等价于beforeMount，而update等价于mounted。所以先后顺序是： 父beforeUpdate -> 子beforeUpdate -> 子update -> 父update。

同理`beforeDestory`和`destoryed`的先后顺序是：父beforeDestory -> 子beforeDestory -> 子destoryed -> 父destoryed。

生命周期钩子函数其实也可以写成数组的形式：`mounted: [mounted1, mounted2]`,同一个生命周期可以触发多个函数，这也是`mixin`（混入）的原理，`mixin`里面也可以写生命周期钩子，最终会和组件里面的生命周期钩子函数一起变成数组形式，`mixin`里面的钩子函数会先执行。

### 异步请求数据在哪个钩子函数中执行比较好

~~很多人觉得在 `created` 事件里面把数据请求到，然后一起生成虚拟 DOM，再渲染会更好。实际上呢，请求是需要时间的，而且这个时间具有不稳定性，很可能 `vue` 的虚拟 DOM 准备好了，你的数据才请求到，然后又得更新一遍虚拟 DOM，再渲染，极大地延长了白屏时间，用户体验很不好。而在 `mounted` 事件请求数据呢，静态页面会先渲染好，等数据好了，再更新部分 DOM 即可。~~

> 补充：经掘友指出，这里理解有误，抱歉。 

生命周期钩子函数的中异步会放入事件队列，而不会在这个钩子函数中执行。也就是说你在 `created` 和 `mounted` 中请求数据是一样的，都不会立即更新数据，所以不会导致虚拟DOM重新加载，也不影响页面中静态的部分加载。生命周期钩子函数中的异步赋值，vue会在一遍流程走完之后执行`update`。另外，给数据赋值然后更新 DOM 也是异步的，侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更，去掉重复赋值然后更新。

生命周期钩子函数中的异步行为测试：

```
export default {
    data(){
        return {
            list:[],
        }
    },
    methods:{
        getData(){
            //生成指定范围的随机整数
            const randomNum = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min; 
            //生成固定长度的非空数组
            const randomArr = length => Array.from({ length }, (item, index) => index * 2); 
            const time = randomNum(100,3000);//模拟请求时间
            console.log('getData start');
            return new Promise(resolve => {
                setTimeout(() => {
                    const arr = randomArr(10);
                    resolve(arr);
                },time)
            })
        }
    },
    async created(){
        console.log('created');
        this.list = await this.getData();
        console.log('getData end');
    },
    beforeMount() {
        console.log('beforeMount');
    },
    mounted(){
        console.log('mounted');
    },
    updated(){
        console.log('updated');
    }
}
复制代码
```

结果如下图，所以在 `created` 中和 `mounted` 中请求数据，数据的更新时间是一样的，在 `created` 中发起请求，可以更早的请求到数据。并且使用服务端渲染SSR的时候， `mounted` 钩子不会加载。



![result](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="162" height="162"></svg>)



### 父组件监听子组件的生命周期

可以写自定义事件，然后在子组件的生命周期函数中触发这个自定义事件，但是不优雅，我们可以使用 hook：

```
<child @hook:created="childCreated"></child>
复制代码
```

从 A 页面切换到 B 页面，A 页面中有一个定时器，到了 B 页面用不上，需要在离开 A 页面的时候清除掉，办法很简单，在 A 页面的生命周期钩子函数`beforeDestory`或者路由钩子函数`beforeRouteLeave`里面清除掉就行，但是问题来了，怎么拿到定时器呢？把定时器写到 `data` 里面，可行但是不优雅，我们有如下写法：

```
//在初始化定时器之后
this.$once('hook:beforeDestory',()=>{
    clearInterval(timer);
})
复制代码
```

### is属性的妙用

由于 HTML 标签的限制，tr 标签里面只能有 th, td 标签，而写自定义标签则会被解析到 tr 标签外层，所以这时候我们可以用 is 属性

```
<tr>
    <td is="child">
</tr>
复制代码
```

最近有个页面有大量的 SVG 图标，我将每一个 SVG 都写成了一个组件。由于 SVG 组件名称又各不相同，所以需要动态标签来表示：

```
<!-- 假设我们的数据如下 -->
arr: [ { id: 1, name: 'first' }, { id: 2, name: 'second' }, { id: 3, name: 'third' }, ]

<!-- 本来需要这样写 -->
<div v-for="item in arr" :key="item.id">
    <p>item.name</p>
    <svg-first v-if="item.id===1"></svg-first>
    <svg-second v-if="item.id===2"></svg-second>
    <svg-third  v-if="item.id===3"></svg-third>
</div>

<!-- 其实这样写更优雅 -->
<div v-for="item in arr" :key="item.id">
    <p>item.name</p>
    <component :is="'svg'+item.name"></component>
</div>
复制代码
```

### 给事件传额外参数

原生 DOM 事件绑定的函数的第一个参数都会是事件对象`event`，但是有时候我们想给这个函数传其他的参数，直接传会覆盖掉`event`，我们可以这么写`<div @click="clickDiv(params,$event)"></div>`，变量`$event`就代表事件对象。

如果要传的变量不是事件对象呢？在使用 `elementUI` 的时候碰到这么一个情况，在表格中使用了下拉菜单组件，代码如下：

```
<el-table-column label="日期" width="180">
    <template v-slot="{row}">
        <el-dropdown-item @command="handleCommand">
            <span>
                下拉菜单<i class="el-icon-arrow-down el-icon--right"></i>
            </span>
            <template #dropdown>
                <el-dropdown-menu>
                    <el-dropdown-item command="a">黄金糕</el-dropdown-item>
                    <el-dropdown-item command="b">狮子头</el-dropdown-item>
                </el-dropdown-menu>
            </template>
        </el-dropdown-item>
    </template>
</el-table-column>
复制代码
```

下拉菜单事件 `command` 函数自带一个参数，为下拉选中的值，这个时候我们想把表格数据传过去，如果`@command="handleCommand(row)"`这样写，就会覆盖掉自带的参数，该怎么办呢？这时候我们可以借助箭头函数：`@command="command => handleCommand(row,command)"`，完美解决传参问题。

顺便说一下，`elementUI` 的表格可以用变量`$index`代表当前的列数，和`$event`一样的使用：

```
<el-table-column label="操作">
    <template v-slot="{ row, $index }">
        <el-button @click="handleEdit($index, row)">编辑</el-button>
    </template>
</el-table-column>
复制代码
```

经掘友指点，默认参数有多个的时候，可以这样写：`@current-change="(...defaultArgs) => treeclick(ortherArgs, ...defaultArgs)"`

### v-slot 语法

`v-slot` 的用法（slot 语法已经废弃）：相当于在组件中留一个空位，使用该组件的时候可以传一些标签过去，插入到对应的空位。可以有多个空位，取不同的名字即可，默认是 `default`。同时还可以将一些数据传过去，简写是`#`。

```
<!-- 子组件 -->
<div class="container">
    <header>
        <slot name="header"></slot>
    </header>
    <main>
        <slot></slot>
    </main>
    <footer>
        <slot name="footer"></slot>
    </footer>
</div>
<!-- 父组件 -->
<base-layout>
    <!-- 插槽可以简写为# -->
    <template #header="data">
        <h1>Here might be a page title</h1>
    </template>
    <!-- v-slot:default可省略 -->
    <div v-slot:default>
        <p>A paragraph for the main content.</p>
        <p>And another one.</p>
    </div>
    <!-- 可以使用解构 -->
    <template #footer="{ user }">
        <p>Here's some contact info</p>
    </template>
</base-layout>
复制代码
```

总结：

- 其他名称的 slot（非 default）仅能用于 `template` 标签。
- 插槽里面的标签拿不到传给子标签的数据（插槽相当于孙子组件）：

```
<child :data="data">
    <div>在这里访问不到data数据</div>
</child>
复制代码
```

- 插槽可以使用解构语法`v-slot="{ user }"`。

### 子组件修改父组件传过来的值

`v-model`在使用的时候很像双向绑定的，但是 Vue 是单项数据流，`v-model` 只是语法糖而已：父组件用`v-bind`将值传给子组件，子组件通过 change/input 事件触发修改父组件的值。

```
<input v-model="inputValue" />
<!-- 等价于 -->
<input :value="inputValue" @change="inputValue = $event.target.value" />
复制代码
```

`v-model` 不仅仅能在 `input` 上用，在组件上也能使用。

vue 组件间传递数据是单向的，即数据总是由父组件传递到子组件，子组件在其内部可以有自己维护的数据，但它无权修改父组件传递给它的数据，我们也可以参照`v-model`语法糖进行修改父组件的值，但是每次都这样写太麻烦了，vue 提供了一个修饰符`.sync`,用法如下：

```
<child :value.sync="inputValue"></child>
<!-- 子组件 -->
<script>
export default {
    props: {
        //props可以设置值得类型，默认值，是否必传以及校验函数
        value: {
            type: [String, Number],
            required: true,
        },
    },
    //用一个变量中转，子组件中就用_value就不会直接修改父组件的值
    computed: {
        _value: {
            get() {
                return this.value;
            },
            set(val) {
                this.$emit('update:value', val);
            },
        },
    },
};
</script>
复制代码
```

### 父组件通过 ref 访问到子组件

虽然 `vue` 提供 `$parent` 和 `$children`来访问父/子组件，但是组件的父组件/子组件存在很多不确定性，例如组件被复用，他的父组件有多种情况。我们可以通过 ref 访问到子组件的数据和方法。

```
<child ref="myChild"></child>
<script>
export default {
    async mounted() {
        await this.$nextTick();
        console.dir(this.$refs.myChild);
    },
};
</script>
复制代码
```

注意：

- `ref` 必须等 DOM 加载好了才可以访问
- 虽然 `mounted` 生命周期 DOM 已经加载好了，但是为了以防万一，我们可以使用 `$nextTick` 函数

### 背景图、css 的@import 使用路径别名

在用 `Webpack` 处理打包时，可将某一目录配置一个别名，代码中就能使用与别名的相对路径引用资源

```
import tool from '@/utils/test'; // Webpack 能正确识别并打包。
复制代码
```

但是在 `css` 文件，如 less, sass, stylus 中，使用 `@import "@/style/theme"` 的语法引用相对 `@` 的目录确会报错。 解决办法是是在引用路径的字符串最前面添加上 ~ 符号。

- css module 中： `@import "~@/style/theme.less"`
- css 属性中： `background: url("~@/assets/xxx.jpg")`
- html 标签中： `<img src="~@/assets/xxx.jpg">`

### vue-router 的 hash 模式和 history 模式

我们先来看一个完整的 URL：`https://www.baidu.com/blog/guide/vuePlugin.html#vue-router`。其中 `https://www.baidu.com` 是网站根目录，`/blog/guide/`是子目录，`vuePlugin.html`是子目录下的文件（如果只有目录，没有指定文件，会默认请求`index.html`文件），而`#vue-router`就是哈希值。

vue 是单页应用，打包之后只有一个 `index.html`，将他部署到服务器上之后，访问对应文件的目录就是访问这个文件。

hash 模式：网址后面跟着 hash 值，hash 值对应每一个 `router` 的名称，`hash` 值改变意味着`router` 改变，监听 `onhashchange` 事件，来替换页面内容。

history 模式：网址后面跟着‘假的目录名’，其值就是 `router` 的名称，而浏览器会去请求这个目录的文件（并不存在，会 404），所以 `history` 模式需要服务器配合，配置 404 页面重定向到到我们的 `index.html`，然后 `vue-router` 会根据目录的名称来替换页面内容。

优缺点：

1. hash 模式的 # 号很丑，使用的是 `onhashchange` 事件切换路由，兼容性会好一点，不需要服务器配合
2. history 模式好看点，但是本地开发、网站上线，都需要服务器额外配置，并且还需要自己写 404 页面，使用的是 `HTML5` 的 `history API`，兼容性差一点。

两者的配置区别在于：

```
const router = new VueRouter({
    mode: 'history', //"hash"模式是默认的，无需配置
    base: '/',//默认配置
    routes: [...]
})
复制代码
```

vue-cli3 的 vue.config.js 配置：

```
module.exports = {
    publicPath: "./", // hash模式打包用
    // publicPath: "/", // history模式打包用
    devServer: {
        open: true,
        port: 88,
        // historyApiFallback: true, //history模式本地开发用
    }
}
复制代码
```

如果是网站部署在根目录，`router` 的 `base` 就不用填。如果整个单页应用服务在 `/app/` 下，然后 `base` 就应该设为 `"/app/"`，同时打包配置(`vue.config.js`)的 `publicPath` 也应该设置成`/app/`。

`vue-cli3`生成新项目的时候会有选择路由的模式，选择`history`模式就会帮你都配置好。

### vue-router的钩子函数

钩子函数分三种：组件内钩子，全局钩子，路由独享钩子。

`APP.vue`没有组件内钩子函数，因为`APP.vue`是页面的入口，这个组件是必定会加载的，而使用组件内钩子函数可以阻止组件加载。

全局钩子主要用于路由鉴权，但是消耗很大。组件内的钩子`beforeRouteLeave`主要用于用户离开前的提示（比如说有未保存的文章），这个钩子有一些坑：hash模式下，浏览器的后退按钮无法触发这个钩子函数。同时我们还可以监听用户的关闭当前窗口/浏览器事件：

```
window.onbeforeunload = e => "确定离开当前页面，你的修改将不会被保存!";
复制代码
```

为了防止恶意网站，用户关闭窗口/浏览器事件是不可阻止的，只能提示，而且不同的浏览器兼容性也不同。

### Vuex持久化存储

Vuex 中的数据，刷新页面之后就会丢失。要实现持久化存储需要借助本地存储（cookie 和 storage 等），一般是登录之后返回的数据（角色，权限，token 等）需要存储到 Vuex，所以我们可以在登录页将数据存储到本地，而在主页面（除了登录页，其他所有页面的入口）进入之前（`beforeCreate` 或者路由钩子 `beforeRouteEnter`）读取出来，并提交到 `Vuex` 就好了。这样即使刷新，也会触发主页面的进入钩子函数，会被提交到 `Vuex`。

```
beforeRouteEnter (to, from, next) {
    const token = localStorage.getItem('token');
    let right = localStorage.getItem('right');
    try{
        right = JSON.parse(right);
    }catch{
        next(vm => {
            //弹窗采用elementUI
            vm.$alert('获取权限失败').finally(() => {
                vm.$router.repalce({name:'login'})
            })
        })
    }
    if(!right || !token){
        next({name:'login',replace:true})
    }else{
        next(vm => {
            //这里面的事件会在mounted之后触发
            vm.$store.commit('setToken',token);
    	    vm.$store.commit('setRight',right);
        })
    }
}
复制代码
```

`beforeRouteEnter`的回调会在`mounted`钩子之后触发，这就比较蛋疼了。而主页面的`mounted`会在所有子组件的`mounted`之后触发，所以我们可以这样写。

```
import store from '^/store';//将实例化的store引入进来

beforeRouteEnter (to, from, next) {
    const token = localStorage.getItem('token');
    if(!token){
        next({name:'login',replace:true})
    }else{
        store.commit('setToken',token);
        next();
    }
}
复制代码
```

要想实现数据修改之后仍能持久化存储，我们可以先把数据存到`localstorage`，然后监听`window.onstorage`事件，数据有修改提交到`Vuex`。

### mutations 里面触发 action

`mutations` 是同步修改 `state` 的值，假如另一个值是异步获取（`action`）的，依赖于这个同步的值的修改，需要在 `mutations` 里面赋值之前触发 `action` 里面的事件，我们可以给实例化的 `Vuex` 命名，在 `mutations` 里面拿到 `store` 对象。

```
const store = new Vuex.Store({
    state: {
        age: 18,
        name: 'zhangsan',
    },
    mutations: {
        setAge(state, val) {
            // 假如age变化了之后，name也要跟着变化
            // 需要在每次给age赋值的时候，同步触发action里面的getName
            state.age = val;
            store.dispatch('getName');
        },
        setName(state, val) {
            state.name = val;
        },
    },
    actions: {
        getName({ commit }) {
            const name = fetch('name'); //从接口异步获取
            commit('setName', name);
        },
    },
});
复制代码
```

### Vue.observable进行组件通信

如果项目很小，不需要用到 `vuex` ，可以用`Vue.observable`来模拟一个：

```
//store.js
import Vue from 'vue';

const store = Vue.observable({ name: '张三', age: 20 });
const mutations = {
    setAge(age) {
        store.age = age;
    },
    setName(name) {
        store.name = name;
    },
};
export { store, mutations };
复制代码
```

### axios 的 qs 插件

get 请求的数据放在 url 里面，类似于`http://www.baidu.com?a=1&b=2`，其中`a=1&b=2`就是 get 的参数，而对于 post 请求，参数放到 body 里面，常用的数据格式有表单数据和 json 数据，两者的差异就是数据格式不同，表单数据编码格式和 get 一样，只不过是放在 body 里面，而 json 数据则是 json 字符串

qs 基本使用:

```
import qs from 'qs'; //qs是axios里面自带的，所以直接引入就可以了
const data = qs.stringify({
    username: this.formData.username,
    oldPassword: this.formData.oldPassword,
    newPassword: this.formData.newPassword1,
});
this.$http.post('/changePassword.php', data);
复制代码
```

`qs.parse()`是将 URL 解析成对象的形式，`qs.stringify()`是将对象 序列化成 URL 的形式，以&进行拼接。而对于不同的数据格式，axios 会自动设置对应的`content-type`，不需要手动设置。

- 表单数据（不带文件）的 content-type 是`application/x-www-form-urlencoded`
- 表单数据（带文件）的 content-type 是`multipart/form-data`
- json 数据的 content-type 是`application/json`

碰到过一次接口需要我用表单传一个数组。假设数据是`arr = [1,2,3]`如果直接使用 qs.stringify()，则数据会变成`arr[]=1&arr[]=2&arr[]=3`，很容易看出来，多了一个`[]`，让接口把参数名改成`arr[]`就能用，但是这样不好。不过可以发现，表单传数组的本质就是**同名参数传多次**，这时候我们也可以这样：

```
const data = new FormData();
arr.forEach(item => {
	data.append('arr', item);
});
复制代码
```

测试一下，完美解决，但是事情到这里还没完，翻一下[qs 官方文档](https://link.juejin.im?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fqs)，qs 转换支持第二个参数，完美解决我们的问题。

```
const data = qs.stringify(arr, { arrayFormat: 'repeat' }); //  arr=1&arr=2&arr=3
复制代码
```

### elementUI 的一些总结

1. 表单验证同步写法，避免多层嵌套函数

```
const valid = await new Promise(resolve => this.$refs.form.validate(resolve));
if (!valid) return
复制代码
```

1. 按需引入之后级联菜单高度撑满屏幕。解决办法：加一句全局样式

```
.el-cascader-menu > .el-scrollbar__wrap{
    height: 250px;
}
复制代码
```

1. 级联菜单的数据是按需获取的没法回显。解决办法：根据已有的路径数据去请求树数据，然后给级联菜单加一个v-if，等数据都请求好了再显示出来。比如说省市县三级联动数据，已知用户选择的是广东省-深圳市-南山区，那么分别去请求所有省、广东省、深圳市的数据，然后将数据拼成一个 tree ，绑定到级联菜单，然后设置`v-if="true"`。
2. 表格高度自适应，可以给表格外层加一个 div ，然后给这个 div 计算高度（或者弹性盒子自适应高度），表格属性`height="100%"`

```
<div class="table-wrap">
    <el-table :height="100%"></el-table>
</div>
复制代码
 /* less写法 */
.table-wrap{
    height: calc(~"100vh - 200px"); 
    /* 部分版本这样写会失效，需要加上下面一句 */
    /deep/ .el-table{
        height: 100% !important;
    }
}
复制代码
```

1. 使用多个 `upload` 组件，需要将这些文件一起上传到服务器。可以通过`this.$refs.poster.uploadFiles`拿到文件对象。然后自己手动组装成表单数据。

```
<el-form-item label="模板文件：" required>
    <el-upload ref="template" action="" :auto-upload="false" accept="application/zip" :limit="1">
        <span v-if="temForm.id">
            <el-button slot="trigger" type="text"><i class="el-icon-refresh"></i>更新文件</el-button>
        </span>
        <el-button slot="trigger" size="mini" type="success" v-else>上传文件</el-button>
    </el-upload>
</el-form-item>
<el-form-item label="模板海报：" required>
    <el-upload action="" :auto-upload="false" ref="poster" accept="image/gif,image/jpeg,image/png,image/jpg" :show-file-list="false" :on-change="changePhoto">
        <img :src="previewUrl" @load="revokeUrl" title="点击上传海报" alt="资源海报" width="250" height="140">
        <template #tip>
            <div>tips: 建议上传尺寸250*140</div>
        </template>
    </el-upload>
 </el-form-item>
复制代码
methods:{
    //选择图片之后替换旧图片和显示略缩图
    changePhoto(file, fileList) {
    //创建的Blob URL可直接预览图片
        this.previewUrl = window.URL.createObjectURL(file.raw);
        if (fileList.length > 1) {
            fileList.shift();
        }
    },
    revokeUrl(e) {
        //图片加载完成之后销毁Blob URL
        if (e.target.src.startsWith("blob:")) window.URL.revokeObjectURL(e.target.src);
   },
    //提交表单数据
    async submitData() {
        const template = this.$refs.template.uploadFiles[0], //模板文件
            poster = this.$refs.poster.uploadFiles[0], //海报文件
            formData = new FormData();
        if (!template) return this.$message.warning("必须选择模板文件");
        if (!poster) return this.$message.warning("必须选择海报文件");
        formData.append("zip", template.raw);
        formData.append("poster", poster.raw);
        const res = await this.$http.post('url', formData);
    },
}
复制代码
```

1. 使用`VueI18n`国际化，需要将`elementUI`的语言包和项目中的语言包合并成一个。

```
import VueI18n from "vue-i18n";
import zhLocale from './locales/zh.js';/* 引入本地简体中文语言包 */
import zhTWLocale from './locales/zh-TW.js';/* 引入本地繁体中文语言包 */
import enLocale from './locales/en.js';/* 引入本地英语语言包 */
import zhElemment from 'element-ui/lib/locale/lang/zh-CN'//引入elementUI简体中文语言包
import zhTWElemment from 'element-ui/lib/locale/lang/zh-TW'//引入elementUI繁体中文语言包
import enElemment from 'element-ui/lib/locale/lang/en'//引入elementUI英语语言包

Vue.use(VueI18n);
const messages = {//语言包
    zh: Object.assign(zhLocale, zhElemment),//本地语言包加入elementUI的语言包
    'zh-TW': Object.assign(zhTWLocale, zhTWElemment),//本地语言包加入elementUI的语言包
    en: Object.assign(enLocale, enElemment)//本地语言包加入elementUI的语言包
};
const i18n = new VueI18n({
    locale: "zh", //zh默认是简体中文
    messages
});

Vue.use(ElementUI, {
    i18n: (key, value) => i18n.t(key, value)
})
```

