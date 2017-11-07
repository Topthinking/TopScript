# Next.js 源码分析 - Topthinking

## 概述

该框架自身结合react和webpack,加上自己的一套路由体系，实现了服务端的渲染

1.页面刷新(首屏加载),服务端通过路由解析，引入.next文件中对应的编译后的代码，在服务端通过react-dom实现从vdom到字符串的转变，最终渲染出整个页面，同时会在_next文件夹中读取客户端需要的配置文件，将需要的js文件随服务器一同传给客户端即浏览器

2.部分刷新(前端路由跳转),在客户端(浏览器)会根据要跳转的路由，通过建立script标签去服务器的_next文件拉取对应路由所需要的js文件，这里会对该js文件的引用做缓存，除非浏览器刷新

3.至于在通过路由解析，不管是服务器上直接引用资源文件，还是客户端请求服务器上的资源文件，其路由都可以很好的控制组件的加载，也就是说，更好的使用getInitialProps方法

## 整体分析

next.js 采用了react的getInitialProps方法实现了服务端的数据远程异步获取的操作，使用react的虚拟dom在服务端实现了
render工作，同时利用react-dom的renderToString方法最终将render的返回的虚拟dom对象转为html字符串，最后将得到的字符串返回给客户端

期间next.js内部封装了webpack功能，同时利用node的http模块实现了服务器，再通过next.js定义的路由规则去读取pages文件夹下的文件作为请求的返回的html文档，最终实现了整个功能

技术栈：nodejs模块 + webpack + react + B/S架构思维 + 数据渲染原理

## 开始

首先要先去github上将源码资源下载下来， https://github.com/zeit/next.js

react实现服务端渲染的技术，主要包括两块文件夹{server,client},从字面意思就很明显这是一个B/S架构，接下来是从server文件夹着手分析，再到client文件夹

在这之前，我们需要对Next.js的next命令进行解读，他其实是我们整个Next.js项目执行的起步器

注意：整个next.js项目的主流程，我会标记两个✨，想直接了解next.js是如何工作的可以直接看打星星部分

### package.json

- 这里说明一下，调试next.js的方法
- 使用 `npm run build` 实现`next.js`项目的编译，我们可以看到next.js使用的是`taskr`命令，其实他执行的配置文件在taskfile.js中，使用的方式和gulp的task模式是一致的
- 自己创建一个next.js项目，在编译的时候使用的`next`命令使用next.js源码目录中的`dist/bin/next`这个命令去执行当前自定义的项目
- 这样便可以修改next.js源码，`taskr`监听编译后，在运行`dist/bin/next`自己创建的项目，然后观察实际运行的情况。可以添加打印，断点去实现和测试源码中看不懂的部分

### bin 文件夹
    我们的所有命令都是next开头，那么自然next是执行的命令，对应到执行文件，也就是该目录下的next文件，接下一一解读

> next

    根据argv去实现 不同的功能，例如：next dev; next -h; next -v;等等
    现在我们要关注的是next dev这个命令
```js
const bin = join(__dirname, 'next-' + cmd)
const proc = spawn(bin, args, { stdio: 'inherit', customFds: [0, 1, 2] })
```
    这段代码意思就是获取该目录下的【next-dev】的文件，所以我们又被引导到了该文件下

> next-dev

    首先这里也是对argv进行了解读，可以在下面的代码看到
```js
const argv = parseArgs(process.argv.slice(2), {
  alias: {
    h: 'help',
    H: 'hostname',
    p: 'port'
  },
  boolean: ['h'],
  string: ['H'],
  default: { p: 3000 }
})
```
    接下就是实例化Server对象，建立http服务，开启服务端渲染了...
```js
import Server from '../server'
const srv = new Server({ dir, dev: true })
srv.start(argv.port, argv.hostname)
```
    我们的思路最终回到了对服务端源码的解析，而入口的文件显然就是server文件下的index.js文件，那么接下来我们来解读React是如何实现服务端渲染，⛽️🆙💪

### server 文件夹
    首先解读index.js文件中Server对象的构造方法中的内容
> index.js > constructor()
    dev==true 需要解读 hotReloader 也就是
```js
getHotReloader (dir, options) {
    const HotReloader = require('./hot-reloader').default
    return new HotReloader(dir, options)
}
```
>> hot-reloader.js  constructor
   返回的是HotReloader对象

>>config.js
        
    我们可以配置next.config.js文件来定制我们的项目
```js
const defaultConfig = {
  webpack: null,
  webpackDevMiddleware: null,
  poweredByHeader: true,
  distDir: '.next',
  assetPrefix: '',
  configOrigin: 'default',
  useFileSystemPublicRoutes: true
}
```

✨✨Server对象的构造方法中的 `defineRoutes` 需要着重去解读路由的定义,所以我们需要去弄清next.js的route
这里可以去看[route.md](./route.md)中的介绍

OK,我们理清了index.js中的构造方法中的所有配置信息，接下来就是真正要调用Server对象的方法start要做的事情了

> ### ✨✨index.js > start()  这里是程序真正开始执行的地方，之前的说明都在做一些配置信息的准备工作
内容很短，我就把方法贴出来了
```js
async start (port, hostname) {
    await this.prepare()
    this.http = http.createServer(this.getRequestHandler())
    await new Promise((resolve, reject) => {
      // This code catches EADDRINUSE error if the port is already in use
      this.http.on('error', reject)
      this.http.on('listening', () => resolve())
      this.http.listen(port, hostname)
    })
  }
```

这是一个异步的方法，里面的调用都是在处理异步等待返回，映入眼帘的便是`prepare`方法，字面意思是准备，其实实现的内容是在dev环境下才会引用`hot-reloader.js`中的`HotReloader`对象，这里是在之前的构造方法中去获取的，那么这里要另开一个文件解读[hot-reloader.js](./hot-reloader.md)文件