# 分析hot-reloader.js HotReloader (dev===true的情况才去实现)

> server/hot-reloader.js

    这里会引入webpack的热更新，主要用于开发阶段，即时更新的模式，这里不是整个next.js的主分支，只是更友好的去开发next.js项目

>> 构造方法

定义一些初始化参数，同时文件引入webpack的几个依赖
```js
import WebpackDevMiddleware from 'webpack-dev-middleware'
import WebpackHotMiddleware from 'webpack-hot-middleware'
import onDemandEntryHandler from './on-demand-entry-handler'
import webpack from './build/webpack'
```
同时读入next.js相关的配置，包括next.config.js中的配置信息

>> start方法
    
    这是一个async方法，配合await实现了异步请求等待功能的实现，首先是用webpack进行了相关的实现，具体下面详谈，下面贴出该方法的具体实现，我们将一步一步的解读这个核心方法的具体实现
```js
async start () {
    const [compiler] = await Promise.all([
        webpack(this.dir, { buildId: this.buildId, dev: true, quiet: this.quiet }),
        clean(this.dir)
    ])

    const buildTools = await this.prepareBuildTools(compiler)
    this.assignBuildTools(buildTools)

    this.stats = await this.waitUntilValid()
}
```

首先我们肯定先从webpack这个方法去解读，那么我们直接解读[webpack](./server_build_webpack.md)