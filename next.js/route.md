# 分析next.js中路由实现的原理和机制  - Topthinking

> server/index.js defineRoutes()

    定义路由匹配规则，可以在接下来的代码中看到是怎么定义的
```js
/**
 * 下面这些键名 都是代码编译后文件真实存放的地址
 * 编译后会有这么几个文件夹
 *  _next : [:buildId] : [page] : [等同于项目里pages里面的文件名称、路径]
            :hash [获取编译后的依赖文件]
    static : [等同于项目里面static文件夹里面的资源文件]

 * 回调的参数 分别是 请求信息、响应信息、参数传递(一般是文件存放的绝对地址，用来获取真正需要请求的文件) 
 */
const routes = {
    '/_next-prefetcher.js': async (req, res, params) => {
        //回调
    },
    '/_next/:buildId/webpack/chunks/:name': async (req, res, params) => {
        //回调
    },
    '/_next/:buildId/webpack/:id': async (req, res, params) => {
        //回调
    },
    '/_next/:hash/manifest.js': async (req, res, params) => {
        //回调
    },
    '/_next/:hash/main.js': async (req, res, params) => {
        //回调
    },
    '/_next/:hash/commons.js': async (req, res, params) => {
        //回调
    },
    '/_next/:hash/app.js': async (req, res, params) => {
        //回调
    },
    '/_next/:buildId/page/_error*': async (req, res, params) => {
        //回调
    },
    '/_next/:buildId/page/:path*': async (req, res, params) => {
        //回调
    },
    '/_next/:path*': async (req, res, params) => {
        //回调
    },
    //存放静态资源，比如图片、字体
    '/static/:path*': async (req, res, params) => {
        //回调
    }
}
```

    接下来，就是讲这些路由存放到内存中，等待被调用，路由里面的回调会找的文件的绝对路径，最终读取文件，返回给客户端，可以在下面的文件中的方法找到
> server/render.js serveStatic()