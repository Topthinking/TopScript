# Next.js 源码分析 - Topthinking

## 整体分析
        next.js 采用了react的getInitialProps方法实现了服务端的数据远程异步获取的操作，使用react的虚拟dom在服务端实现了render工作，同时利用react-dom的renderToString方法最终将render的返回的虚拟dom对象转为html字符串，最后将得到的字符串返回给客户端
        期间next.js内部封装了webpack功能，同时利用node的http模块实现了服务器，再通过next.js定义的路由规则去读取pages文件夹下的文件作为请求的返回的html文档，最终实现了整个功能
        技术栈：nodejs模块 + webpack + react + B/S架构思维 + 数据渲染原理

## 开始
        首先要先去github上将源码资源下载下来， https://github.com/zeit/next.js
        react实现服务端渲染的技术，主要包括两块文件夹{server,client},从字面意思就很明显这是一个B/S架构，接下来是从server文件夹着手分析，在到client文件夹
        在这之前，我们需要对Next.js的next命令进行解读，他其实是我们整个Next.js项目执行的起步器

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
> index.js > constructor
    dev==true 需要解读 hotReloader 也就是
```js
getHotReloader (dir, options) {
    const HotReloader = require('./hot-reloader').default
    return new HotReloader(dir, options)
}
```
   > hot-reloader.js > constructor
   返回的是HotReloader对象

>config.js
        
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

