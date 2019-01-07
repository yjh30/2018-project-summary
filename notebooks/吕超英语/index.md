# 吕超英语h5项目总结
## 一、技术重点模块
### [移动端Vue单页应用 与 大数据埋点最佳实践](./clogger.md)
### 改造[connect-history-api-fallback](https://www.npmjs.com/package/connect-history-api-fallback)，开发[koa-history-api-fallback-middleware](https://github.com/yjh30/koa-history-api-fallback-middleware)
- connect-history-api-fallback并不能用于node服务生产环境；
- 使用koa-history-api-fallback-middleware与koa-static一定要注意顺序关系，举个例子（下面代码正常）：
```js
const Koa = require('koa')
const Router = require('koa-router')
const server = new Koa()
const router = new Router()

// 模拟使用koa-history-api-fallback-middleware逻辑
server.use(async (ctx, next) => {
  if (ctx.url === '/aa') {
    ctx.url = '/bb'
  }
  await next()
})

// 模拟使用koa-static中间件逻辑
server.use(async (ctx, next) => {
  if (ctx.url === '/bb') {
    ctx.body = '/aa'
  } else {
    await next()
  }
})

router.get('/a', ctx => {
  ctx.body = ctx.path
})
router.get('/b', ctx => {
  ctx.body = ctx.path
})
router.get('/c', ctx => {
  ctx.body = ctx.path
})

server
  .use(router.routes())
  .use(router.allowedMethods())
server.listen(5002)
```
如果模拟使用koa-static中间件逻辑代码放在模拟使用koa-history-api-fallback-middleware逻辑上面，那么访问localhost:5002/aa 就404了

### 开发[koa-mocker-cli](https://www.npmjs.com/package/koa-mocker-cli)脚手架工具
对process进程uncaughtException事件的重新认识：该事件捕获异步错误，一般如果异步错误耗损cpu及内存，监听该该事件应该尝试退出node进程然后尝试重启服务，如果错误只是简单的如vue ssr同构异步错误，那么为了保证线上其他项目不受影响应该打印错误日志并邮件通知开发，解决错误后重新打包部署服务

## 二、项目业务代码最优实现及难点记录
### navBar公共组件的最优实现
- 组件dom渲染后注册androidBackKey(全局性)事件监听，销毁后移除androidBackKey事件监听；
- 考虑到路由页面组件使用了keep-alive缓存功能，应该在androidBackKeyListener监听器中只执行当前路由navBar逻辑；
```js
export default {
  props: {
    title: {
      type: String
    },
    to: {
      type: [String, Function]
    }
  },

  data() {
    return {
      routerPath: ""
    };
  },

  mounted() {
    this.routerPath = this.$route.path;
    this.$pageEvents.on("androidBackKey", this.androidBackKeyListener);
  },

  beforeDestroy() {
    this.$pageEvents.removeListener(
      "androidBackKey",
      this.androidBackKeyListener
    );
  },

  methods: {
    androidBackKeyListener() {
      // 执行当前路由页面逻辑
      if (this.$route.path !== this.routerPath) {
        return;
      }
      this.slide(this.to);
    },

    async slide(to) {
      if (!to) {
        history.back();
      }

      if (typeof to === "string") {
        if (to === "native" || to === "quitWebView" || to === "closeWebview") {
          this.$clogger.emit("routeLeave", location.href);
          await new Promise(resolve => {
            setTimeout(resolve, 300);
          });
          console.log("closeWebview");
          window.mstJsBridge && mstJsBridge.closeWebview();
        }
      } else if (typeof to === "function") {
        to();
      }
    }
  }
};
```
- 如果一个路由页面有两个nav-bar组件，怎么办？看下面代码处理
```js
let menuBackTimeout = null;

export default {
  /**
   * menuBack 菜单返回
   * @return {undefined}
   */
  menuBack() {
    // 由于该页面存在两个nav-bar组件，点击安卓物理返回键会触发
    // 两个nav-bar组件监听的事件处理逻辑，下面代码会执行两次，故使用定时器处理
    clearTimeout(menuBackTimeout);
    menuBackTimeout = setTimeout(() => {
      history.back();
    }, 100);
  }
};
```
### 匹配重点单词正则
```js
// 需要考虑英文单词后面紧跟. , ; ! ?
const reg = new RegExp(
  `(?:^|\\s+)(${english})(?:$|\\s|\\.|\\,|\\;|\\!|\\?)`,
  "ig"
);
```
之前错误的用法
```js
 // 下面正则匹配 'xxx随便写english'字符串
 // ^在中括号表示非，后面跟什么就是非什么，这个例子是非空，\s在中括号中就是\s字符串而不是表示空字符串
const reg = /[^|\s+]english/ig;
```

### 借助webpack.DefinePlugin将process.env环境变量写入到客户端代码中
package.json有如下配置:
```json
  "scripts": {
    "devServer": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    "mock": "mocker --port=5000 --watch=mock",
    "dev": "npm run mock & npm run devServer --env=mock",
    "dev:mock": "npm run mock & npm run devServer --env=mock",
    "dev:stable": "npm run devServer --env=stable",
    "dev:sit": "npm run devServer --env=sit",
    "start": "npm run dev",
    "build": "node build/build.js --env=production",
    "build:stable": "npm run build --env=stable",
    "build:sit": "npm run build --env=sit"
  }
```
通过webpack.DefinePlugin定义，我们可以在客户端代码中这样写：
```js
const env = process.env.npm_config_env;
let urlPath = "//clog.offline.mistong.com";
if (env === "production") {
  urlPath = "//clog.ewt360.com";
}
```

### vue-router路由history模式无网络模式处理
当点击一个按钮发生路由导航时，如果此时断网，将会无任何反应，因为导航变跟时会请求服务端html文件，此时断网将无响应，因此需判断当navigator.onLine为true时，执行vm.$router.push 无网给出无网络友好提示，而不是无任何响应

### 开发环境下引入腾讯[Vconsole](https://github.com/Tencent/vConsole)模块
```html
<% if(!htmlWebpackPlugin.options.isProduction) { %>
<script src="/static/js/vconsole.min.js"></script>
<script>
  var vConsole = new VConsole();
</script>
<% } %>
```