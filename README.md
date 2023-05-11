# 基于single-spa的微前端应用demo
## 示例项目
新建项目

```
mkdir micro-frontend && cd micro-frontend
```

示例代码都是通过vue来编写的，当然也可以采用其它的，比如react或者原生JS等
### 子应用app1
**新建子应用**

```
    vue create app1
```
> 除了第一步：选择Manually select features(自主搭配)模式；后续全部使用默认选项，一路回车等待应用创建完毕

**配置子应用**
> 以下所有的操作都在项目根目录/micro-frontend/app1下完成

#### vue.config.js
在项目根目录下新建vue.config.js文件
```
    const package = require('./package.json')
    module.exports = {
    // 告诉子应用在这个地址加载静态资源，否则会去基座应用的域名下加载
    publicPath: '//localhost:8081',
    // 开发服务器
    devServer: {
        port: 8081
    },
    configureWebpack: {
        // 导出umd格式的包，在全局对象上挂载属性package.name，基座应用需要通过这个全局对象获取一些信息，比如子应用导出的生命周期函数
        output: {
        // library的值在所有子应用中需要唯一
        library: package.name,
        libraryTarget: 'umd'
        }
    }
    }
```
#### 安装single-spa-vue

```
npm i single-spa-vue -S
```

single-spa-vue负责为vue应用生成通用的生命周期钩子，在子应用注册到single-spa的基座应用时需要用到

#### 改造入口文件
```
// /src/main.js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import singleSpaVue from 'single-spa-vue'

Vue.config.productionTip = false

const appOptions = {
  el: '#microApp',
  router,
  render: h => h(App)
}

// 支持应用独立运行、部署，不依赖于基座应用
if (!window.singleSpaNavigate) {
  delete appOptions.el
  new Vue(appOptions).$mount('#app')
}

// 基于基座应用，导出生命周期函数
const vueLifecycle = singleSpaVue({
  Vue,
  appOptions
})

export function bootstrap (props) {
  console.log('app1 bootstrap')
  return vueLifecycle.bootstrap(() => {})
}

export function mount (props) {
  console.log('app1 mount')
  return vueLifecycle.mount(() => {})
}

export function unmount (props) {
  console.log('app1 unmount')
  return vueLifecycle.unmount(() => {})
}
```
#### 更改视图文件
```
<!-- /views/Home.vue -->
<template>
  <div class="home">
    <h1>app1 home page</h1>
  </div>
</template>
```
```
<!-- /views/About.vue -->
<template>
  <div class="about">
    <h1>app1 about page</h1>
  </div>
</template>
```
### 环境配置文件
应用独立运行时的开发环境配置(当前子应用的根目录下新建.env文件)
```
NODE_ENV=development
VUE_APP_BASE_URL=/
```
作为子应用运行时的开发环境配置(当前子应用的根目录下新建.env.micro文件)
```
NODE_ENV=development
VUE_APP_BASE_URL=/app1
```
作为子应用构建生产环境bundle时的环境配置，但这里的NODE_ENV为development，而不是production，是为了方便，这个方便其实single-spa带来的弊端（js entry的弊端）(当前子应用的根目录下新建.env.buildMicro文件)
```
NODE_ENV=development
VUE_APP_BASE_URL=/app1
```
### 修改路由文件
```
// /src/router/index.js
// ...
const router = new VueRouter({
  mode: 'history',
  // 通过环境变量来配置路由的 base url
  base: process.env.VUE_APP_BASE_URL,
  routes
})
// ...

```
###  修改package.json中的script
```
{
  "name": "app1",
  // ...
  "scripts": {
    // 独立运行
    "serve": "vue-cli-service serve",
    // 作为子应用运行
    "serve:micro": "vue-cli-service serve --mode micro",
    // 构建子应用
    "build": "vue-cli-service build --mode buildMicro"
  },
     // ...
}

```
### 启动子应用
#### 应用独立运行
```
npm run serve
```
#### 当然下面的启动方式也可以，只不过会在pathname的开头加了/app1前缀
```
npm run serve:micro
```
### 子应用app2

在/micro-frontend目录下新建子应用app2，步骤同app1，只需把过程中出现的'app1'字样改成'app2'即可，vue.config.js中的8081改成8082`

### 子应用app3
示例项目是基于react脚手架create-react-app创建的，整个集成的过程中难点有两个：

- webpack的配置，这部分内容官网有提供
- 子应用入口的配置，单纯看官方文档的示例项目根本跑不起来，或者即使跑起来也有问题，react和vue的集成还不一样，react需要在主项目的配置中也加一点东西，这部分官网配置没说，是通过single-spa-react源码看出来的

#### 安装app3

```
npx create-react-app app3
```
以下所有操作都在/micro-frontend/app3目录下进行

#### 安装react-router-dom、single-spa-react
```
安装react-router-dom、single-spa-react
```

#### 打散配置
```
npm run eject
```

####  更改 webpack 配置文件
/config/webpack.config.js

- 删掉optimization部分，这部分配置和chunk有关，有动态生成的异步chunk存在，会导致主应用无法配置，因为chunk的名字会变，其实这也是single-spa的缺陷，或者说采用JS entry的缺陷，JS entry建议将所有内容都打成一个bundle - app.js
- 更改entry和output部分
```
{
  ...
  entry: [
      paths.appIndexJs,
    ].filter(Boolean),
  output: {
    path: isEnvProduction ? paths.appBuild : undefined,
    filename: 'js/app.js',
    publicPath: '//localhost:3000',
    jsonpFunction: `webpackJsonp${appPackageJson.name}`,
    library: 'app3',
    libraryTarget: 'umd'
  },
  ...
}
```

#### 项目入口文件改造
我这里将无关紧要的内容都删了，只留了/src/index.js和/src/index.css
+ /src/index.js
> 参考/app3/src/index.js
+ /src/index.css
> 参考/app3/src/index.css

#### 启动子应用
```
npm run start
```

### 基座应用 layout
在/micro-frontend目录下新建基座应用，为了简洁明了，新建项目时选择的配置项和子应用一样；在本示例中基座应用采用了vue来实现，用别的方式或者框架实现也可以，比如自己用webpack构建一个项目。
> 以下操作都在/micro-frontend/layout目录下进行

#### 安装single-spa
```
npm i single-spa -S
```

#### 改造基座项目
**入口文件**
```
// src/main.js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import { registerApplication, start } from 'single-spa'

Vue.config.productionTip = false

// 远程加载子应用
function createScript(url) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src = url
    script.onload = resolve
    script.onerror = reject
    const firstScript = document.getElementsByTagName('script')[0]
    firstScript.parentNode.insertBefore(script, firstScript)
  })
}

// 记载函数，返回一个 promise
function loadApp(url, globalVar) {
  // 支持远程加载子应用
  return async () => {
    await createScript(url + '/js/chunk-vendors.js')
    await createScript(url + '/js/app.js')
    // 这里的return很重要，需要从这个全局对象中拿到子应用暴露出来的生命周期函数
    return window[globalVar]
  }
}

// 子应用列表
const apps = [
  {
    // 子应用名称
    name: 'app1',
    // 子应用加载函数，是一个promise
    app: loadApp('http://localhost:8081', 'app1'),
    // 当路由满足条件时（返回true），激活（挂载）子应用
    activeWhen: location => location.pathname.startsWith('/app1'),
    // 传递给子应用的对象
    customProps: {}
  },
  {
    name: 'app2',
    app: loadApp('http://localhost:8082', 'app2'),
    activeWhen: location => location.pathname.startsWith('/app2'),
    customProps: {}
  },
  {
    // 子应用名称
    name: 'app3',
    // 子应用加载函数，是一个promise
    app: loadApp('http://localhost:3000', 'app3'),
    // 当路由满足条件时（返回true），激活（挂载）子应用
    activeWhen: location => location.pathname.startsWith('/app3'),
    // 传递给子应用的对象，这个很重要，该配置告诉react子应用自己的容器元素是什么，这块儿和vue子应用的集成不一样，官网并没有说这部分，或者我没找到，是通过看single-spa-react源码知道的
    customProps: {
      domElement: document.getElementById('microApp'),
      // 添加 name 属性是为了兼容自己写的lyn-single-spa，原生的不需要，当然加了也不影响
      name: 'app3'
    }
  }
]

// 注册子应用
for (let i = apps.length - 1; i >= 0; i--) {
  registerApplication(apps[i])
}

new Vue({
  router,
  mounted() {
    // 启动
    start()
  },
  render: h => h(App)
}).$mount('#app')

```

#### App.vue
```
<template>
  <div id="app">
    <div id="nav">
      <router-link to="/app1">app1</router-link> |
      <router-link to="/app2">app2</router-link>
    </div>
    <!-- 子应用容器 -->
    <div id = "microApp">
      <router-view/>
    </div>
  </div>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

#nav {
  padding: 30px;
}

#nav a {
  font-weight: bold;
  color: #2c3e50;
}

#nav a.router-link-exact-active {
  color: #42b983;
}
</style>

```

#### 路由
```
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)

const routes = []

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes
})

export default router

```

#### 启动基座应用

```
npm run serve
```
#### 上述配置基本上照搬了下面这篇文章
https://segmentfault.com/a/1190000041336746