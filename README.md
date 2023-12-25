# 微前端`qiankun`接入`Vite`构建的子应用


# 存在问题

qiankun官方暂未有文档表明已经支持Vite，所以直接用普通方式接入vite构建的子应用时会出现如下问题：

![](https://cdn.jsdelivr.net/gh/HelloAllenW/BlogAssets/images/202312251700488.png)

<br />

# 原因分析

## 1. 开发模式：

在开发环境下，如果我们使用 vite 来构建 vue3 子应用，基于vite的构建机制，会在子应用的 html 的入口文件的 script 标签上携带 `type=module`。而我们知道qiankun父应用引入子应用，本质上是将html做为入口文件，并通过`import-html-entry`这个库去加载子应用所需要的资源列表JS、CSS，然后通过eval直接执行，而基于vite构建的JS中import、export并没有被转码（vite是基于浏览器支持的 `ESM import`特性实现的 bundless，通过利用浏览器进行模块间依赖加载，而不需要在编译时进行。），会导致直接报错（不允许在非 `type=module` 的 script 里面使用 import）

## 2. 生产模式：

生产模式下，因为没有诸如webpack中支持运行时publicPath,也就是`__webpack_public_path__`，换句话说就是vite不支持运行时publicPath，其主要作用是用来解决微应用动态载入的脚本、样式、图片等地址不正确的问题。

<br />

# 解决方案

有一款开源插件：`vite-plugin-qiankun`，通过这个插件可以在qiankun下解决上述两种模式的问题，同时保留了vite构建模块的优势。

##  1. 创建主应用

（1）安装qiankun

```
npm install qiankun
```

（2）新建`src/qiankun/index.js`文件，进行单独的抽离

```
import { registerMicroApps, start } from 'qiankun'
registerMicroApps([
    {
        // 必须与子应用注册名字相同
        name: 'vue-app',
        // 入口路径，开发时为子应用所启本地服务，上线时为子应用线上路径
        entry: 'http://127.0.0.1:5174', 
        // 子应用挂载的节点
        container: '#vue-app-container',
        // 当访问路由为 /micro-vue 时加载子应用
        activeRule: '/micro-vue', 
        // 主应用向子应用传递参数
        props: {
            msg: "我是来自主应用的值-vue"
        }
    },
    {
        name: 'react-app',
        entry: 'http://127.0.0.1:5175',
        container: '#react-app-container',
        activeRule: '/micro-react',
        props: {
            msg: "我是来自主应用的值-react"
        }
    }
])
start()
```

（3）在`main.js`中导入

```
import "./qiankun"
```

（4）在`App.vue`挂载子应用

```
  <div id="vue-app-container" />
  <div id="react-app-container" />
```



## 2.创建子应用 micro-vue-app （vue3 + vite）

qiankun目前是不支持vite的，需要借助插件完成

（1） 安装插件

```
npm install vite-plugin-qiankun
```

（2）修改`vite.config.js`

```
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import qiankun from 'vite-plugin-qiankun'

export default defineConfig({
  plugins: [
    vue(),
    qiankun('vue-app', { // 子应用名字，与主应用注册的子应用名字保持一致
      useDevMode: true
    })
  ],
  server: {
    origin: 'http://localhost:5174', // 解决静态资源加载404问题
    host: 'localhost',
    port: 5174
  }
})
```

（3）修改`main.js`

```
import { createApp } from 'vue'
import App from './App.vue'
import { store } from './store'

import { renderWithQiankun, qiankunWindow } from 'vite-plugin-qiankun/dist/helper'

let instance = null;
const initQianKun = () => {
    renderWithQiankun({
        mount(props) {
            render(props.container)
            //  可以通过props读取主应用的参数：msg
            // 监听主应用传值
            props.onGlobalStateChange((res) => {
                store.count = res.count
                console.log(res.count)
            })
        },
        bootstrap() { },
        unmount() {
            instance.unmount()
            instance._instance = null
            instance = null
        },
        update() { }
    })
}

const render = (container) => {
    if (instance) return;
    // 如果是在主应用的环境下就挂载主应用的节点，否则挂载到本地
    // 注意：这边需要避免 id（app） 重复导致子应用挂载失败
    const appDom = container ? container.querySelector("#app") : "#app"
    instance = createApp(App)
    instance.mount(appDom)
}

// 判断当前应用是否在主应用中
qiankunWindow.__POWERED_BY_QIANKUN__ ? initQianKun() : render()
```

（4）修改route文件，采用hash模式

qiankun官方是以`window.__POWERED_BY_QIANKUN__`来判断当前是否为qiankun环境下，而该插件引用之后是通过`qiankunWindow.__POWERED_BY_QIANKUN__`来判断。

```
import { createWebHashHistory } from 'vue-router'
import { qiankunWindow } from 'vite-plugin-qiankun/dist/helper'
const router = createRouter({
    // 值和主应用中的activeRule保持一致    
    history: createWebHashHistory(qiankunWindow.__POWERED_BY_QIANKUN__ ? '/vueApp' : '/'),     
    routes: routes 
});
```



## 3.创建子应用 micor-react-app （react18 + vite）

和vue配置一样，但是会报错：

解决方法：在 `vite.config.js` 中删除`react()`

```
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import qiankun from 'vite-plugin-qiankun'

export default defineConfig({
  plugins: [
    // 在开发模式下需要把react()关掉
    // https://github.com/umijs/qiankun/issues/1257
    // react(),
    qiankun('react-app', {// 子应用名字，与主应用注册的子应用名字保持一致
      useDevMode: true
    })
  ],
  server: {
    port: '5175',
  }
})
```
完整代码见[Demo](https://github.com/HelloAllenW/qiankun-demo-vite)

<br />

# 可能存在的其他问题

1. 插件`vite-plugin-qiankun`在生产模式下依旧不支持`publicPath`, 需要将`vite.config.js`中`base`配置写死。导致多环境部署不便捷。无法像在webpack结合`window.INJECTED_PUBLIC_PATH_BY_QIANKUN` + `publicpath`来解决。

   ![](https://cdn.jsdelivr.net/gh/HelloAllenW/BlogAssets/images/202312251709635.png)

2. 关于`vue3 + vite + typescript`项目中出现 `“Error: The package "@esbuild/win32-x64" could not be found, and is needed by esbuild.” `的错误。

   ![](https://cdn.jsdelivr.net/gh/HelloAllenW/BlogAssets/images/202312251711450.png)

   在运行dev之前先运行`·node node_modules/esbuild/install.js`命令来解决esbuild安装问题。然后再启动项目，发现已经能正常运行。



