---
title: vue + axios 实现路由拦截
date: 2018-4-12 13:20:45
categories: 
    - 前端
tags: 
    - vue axios
cover: http://img0.imgtn.bdimg.com/it/u=3569112549,2295780685&fm=26&gp=0.jpg
---

 基于现在用vue+webpack搭建项目的文档已经有很多了,我就不再累述了.

  技术栈

- vue2.0
- vue-router
- axios

  拦截器

​    首先我们要明白设置拦截器的目的是什么,当我们需要统一处理http请求和响应时我们通过设置拦截器处理方便很多.

​    这个项目我引入了element ui框架,所以我是结合element中loading和message组件来处理的.我们可以单独建立一个http的js文件处理axios,再到main.js中引入.

​    

```
 1 /**
 2  * http配置
 3  */
 4 // 引入axios以及element ui中的loading和message组件
 5 import axios from 'axios'
 6 import { Loading, Message } from 'element-ui'
 7 // 超时时间
 8 axios.defaults.timeout = 5000
 9 // http请求拦截器
10 var loadinginstace
11 axios.interceptors.request.use(config => {
12   // element ui Loading方法
13   loadinginstace = Loading.service({ fullscreen: true })
14   return config
15 }, error => {
16   loadinginstace.close()
17   Message.error({
18     message: '加载超时'
19   })
20   return Promise.reject(error)
21 })
22 // http响应拦截器
23 axios.interceptors.response.use(data => {// 响应成功关闭loading
24   loadinginstace.close()
25   return data
26 }, error => {
27   loadinginstace.close()
28   Message.error({
29     message: '加载失败'
30   })
31   return Promise.reject(error)
32 })
33 
34 export default axios
```

 

   这样我们就统一处理了http请求和响应的拦截.当然我们可以根据具体的业务要求更改拦截中的处理.

 路由拦截

​    我们可以通过路由拦截做什么?我认为最主要的便是对权限的控制,比如有的页面需要登录了才能进入,有些页面不同身份渲染不同.接下来简单的讲一下登录拦截.

​    

```
 1 import Vue from 'vue'
 2 import Router from 'vue-router'
 3 
 4 Vue.use(Router)
 5 
 6 const router = new Router({
 7   routes: [
 8     {
 9       path: '/',
10       /*
11       *  按需加载 
12       */
13       component: (resolve) => {
14         require(['../components/Home'], resolve)
15       }
16     }, {
17       path: '/record',
18       name: 'record',
19       component: (resolve) => {
20         require(['../components/Record'], resolve)
21       }
22     }, {
23       path: '/Register',
24       name: 'Register',
25       component: (resolve) => {
26         require(['../components/Register'], resolve)
27       }
28     }, {
29       path: '/Luck',
30       name: 'Luck',         // 需要登录才能进入的页面可以增加一个meta属性
31       meta: { 
32         requireAuth: true
33       },
34       component: (resolve) => {
35         require(['../components/luck28/Luck'], resolve)
36       }
37     }
38   ]
39 })
40 //  判断是否需要登录权限 以及是否登录
41 router.beforeEach((to, from, next) => {
42   if (to.matched.some(res => res.meta.requireAuth)) {// 判断是否需要登录权限
43     if (localStorage.getItem('username')) {// 判断是否登录
44       next()
45     } else {// 没登录则跳转到登录界面
46       next({
47         path: '/Register',
48         query: {redirect: to.fullPath}
49       })
50     }
51   } else {
52     next()
53   }
54 })
55 
56 export default router
```

​      这样就做好了登录拦截.我们只需在main.js中引入router就可以了.

​      实现权限的控制我们还可以通过Vuex来实现,但是如果是小型项目就没必要引入Vuex了.