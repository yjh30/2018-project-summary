# 移动端Vue单页应用 与 大数据埋点最佳实践

## 1、确认page事件埋点发射地点
- 了解pageshow,pagehide,focus,blur,visibilitychange
- [了解VueRouter导航解析流程](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E5%AE%8C%E6%95%B4%E7%9A%84%E5%AF%BC%E8%88%AA%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B)
- 在文档visibilitychange事件中发射页面载入，页面卸载事件，如果客户端app webview不支持，需要寻求客户端支持或者通过jsBridge实现
- 在VueRouter全局守卫beforeResolve内发射page进入，page离开事件，beforeResolve是在导航被确认之前，同时在所有组件内守卫和异步路由组件被解析之后调用
```js
// 为什么选择beforeResolve，而不是全局beforeEach，afterEach守卫，请看下列代码中的注释部分
const router = new Router({
  // ...配置省略
});
router.beforeResolve(async (to, from, next) => {
  await next();
  // 在这里发射日志page事件
  // 这里执行的代码是在路由页面组件beforeRouteEnter守卫内next回调中的代码执行之后
});
export default router;
```
- 路由页面性能page日志埋点（页面数据渲染后发送性能日志埋点）
```js
// 首页路由页面组件
export default {
  async mounted() {
    await this.$request.post('/api') // 请求接口数据
    this.$clogger.emit("render");
  }
};
```

## 2、如何最佳处理click埋点
- 方式1、在dom元素上绑定一个databig-click-id属性，然后通过该事件代理将click事件绑定在document.body上，通过event.target判断是否有databig-click-id属性，没有继续遍历父元素...，缺点1：可能需要遍历；缺点2：事件不冒泡，其他开发者执行了event.stopPropagation
- 方式2、全局mixin一个组件，该组件包含一个处理click埋点的方法；缺点：影响所有vue组件
- 方式3、将该方法封装在一个实例对象上，返回一个promise，该promise总是resolve


## 3、Clogger类封装，实例化应用程序共享实例
- 创建一个继承EventEmitter事件发射器的Clogger类
- 在Clogger类构造器中监听页面载入routeEnter，页面卸载routeLeave事件，click事件处理，页面数据渲染render事件处理
- 创建一个Clogger类的实例对象，挂载在Vue原型对象上
```js
const clogger = new Clogger({
  urlPath: 'https://xxx.com',
  extra: {
    //... 配置扩展字段
  }
})
Vue.prototype.$clogger = clogger
```

## 4、在navBar公共组件中处理退出(关闭)webview
```js
async exitWebview() {
  this.$clogger.emit("routeLeave", location.href);
  await new Promise(resolve => {
    setTimeout(resolve, 300);
  });
  // 执行jsBridge封装好的关闭客户端webview方法
}
```

## 5、处理click事件埋点
```js
  /**
   * backHome 返回首页
   * @return {undefined}
   */
  async backHome() {
    await this.$clogger.click("8.8.1");

    // 如果后续代码没有路由跳转行为，你也可以这样使用this.$clogger.emit("click", "8.8.1");
    this.$router.push({
      path: "/home"
    });
  }
```

## 6、处理日志extra扩展字段
```js
// 首页路由页面组件
export default {
  beforeRouteEnter(to, from, next) {
    next(vm => {
      // 在首页路由埋点扩展对象中增加name字段
      vm.$clogger.extra.name = 'yjh'
    });
  }
};
```