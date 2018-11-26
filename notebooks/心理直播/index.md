
# 心理直播h5项目总结
- [微信小程序直播总结](https://github.com/yjh30/miniprogram-pit)
- 前端框架：vue.js 
- 服务端脚本：nodejs 
- 运行平台：微信浏览器 
- 直播微信小程序时间段：小程序调研开发(9/21-9/26)，涉及到直播审核风险后放弃，直播类微信小程序需要 信息网络传播视听节目许可证 或 网络文化经营许可证 或 每年缴纳50万给腾讯才能确保审核通过
- 开发联调测试上线时间段：一期上部分(9/27-10/17)，一期下部分(10/8-10/24)，二期(11/12-11/28)

## 直播相关
#### 1、video标签播放HLS协议直播流
- [本地搭建直播流服务器](./搭建本地直播流服务器.md)
- HLS协议直播流相对RTMP协议延时较高
- video标签在播放直播流的过程中是可以暂停的，为了让所有用户实时观看到最新的直播信息，在暂停的事件监听器中重新渲染video，后台在断流结束直播的时候确保所有用户看到的都是快结束的直播流视频

## Vue相关
#### 2、用户体验（浏览器历史记录前进后退，保留路由页面滚动位置）
见：[Vue Router异步滚动](https://router.vuejs.org/zh/guide/advanced/scroll-behavior.html#滚动行为)
该功能只是针对document.body滚动位置有效，遇到无限滚动加载的页面配合使用Vue内置组件[keep-alive](https://cn.vuejs.org/v2/api/#keep-alive)

#### 3、keep-alive带来的一些问题
被缓存的路由页面组件离开之后再次进入是不会重新渲染的，有这样一个问题，页面中某个dom节点元素注册了transitionEnd事件监听，当路由页面离开之后(dom不在文档树中，保存在内存中)，过渡动画未执行完成，transitionEnd事件监听器是不会执行的，而我恰恰需要在监听器中执行某些动作，如改变某些变量锁的值为false。<br>
**举例**：我写了一个图片轮播组件[vue-ui-slide](https://www.npmjs.com/package/vue-ui-slide)，其中定义了一个animing变量，动画是否正在执行，如果为true则不执行touch事件监听，在transitionEnd事件监听器将其设置为false<br>
**解决办法**：在路由页面组件beforeRouteEnter回调中执行transitionEnd事件监听器

#### 4、确保当前路由页面离开之后不会产生任何的副作用代码
- 清除各种轮询定时器
- 移除绑定在window,document.documentElement,document.body，非dom的全局性事件监听器
- 涉及全局性的操作需要非常谨慎，异步设置全局性的操作需要判断是在当前路由页面操作。如：在获取java接口成功后动态的设置页面标题，配置微信分享
```js
// 路径为/home的页面代码，在请求非常慢未完成的情况下，路由跳转到其他页面，如果不加路由判断那么就会有问题
this.$request.get('/getdata')
  then(res => {
    if (this.$route.path === '/home') {
      // document.title ...
      // wx.config ...
    }
  })
```
#### 5、理解页面路由守卫beforeRouteEnter中执行next(false)
- 首先执行next(false)不会执行路由匹配组件的生命周期函数
- 当前路由的路径是上一个路由的路径而不是当前执行next(false)所在组件的路由path

```js
// 当重定向授权成功后 按历史记录返回键返回时并不是该代码段所在组件匹配到的路由path
beforeRouteEnter(to, from, next) {
  // 客户端未授权
  if (typeof window !== 'undefined' && !window.openId) {
    // 重定向微信授权，逻辑省略...
    next(false)
    return;
  }
  next()
}

// 当重定向授权成功后 按历史记录返回键返回正常
beforeRouteEnter(to, from, next) {
  next(vm => {
    // 服务端执行 或 已授权
    if (vm.$isServer || window.openId) {
      return;
    }
    // 重定向微信授权，逻辑省略...
  });
}
```
#### 6、列表分页无限加载需要注意的问题
- 由于是单页应用，数据驱动，在满足请求下一页的情况下，需做一下延时阻塞处理，否则会出现在滚动满足请求条件的情况下请求多页数据

```js
  async scrollEventListener() {
    if (this.loading || (this.pageIndex !== 1 && this.pageIndex >= this.totalPages)) {
      return;
    }

    const { clientHeight, scrollHeight } = document.documentElement;
    const scrollTop = document.documentElement.scrollTop || document.body.scrollTop || window.scrollTop;

    if (scrollTop + clientHeight >= scrollHeight) {
      this.loading = true;
      this.errorMsg = '';

      // 单页应用，数据驱动，
      await new Promise(r => {
        setTimeout(r, 500);
      });

      this.pageIndex++;
      await this.requestNextPageData();
      this.loading = false;
    }
  }
```
## 微信相关
#### 7、微信分享问题
3.1、排查invalid signature，一般出现这个问题有两点：

  - 检查wx.config配置的appId是否一致
  - 检查请求后端签名接口的url（本人曾意外使用了两次encodeURIComponent）

3.2、版本判断，引用微信jssdk1.4.0，且微信版本大于等于6.7.2，则使用最新的分享朋友，分享到朋友圈api

#### 8、微信浏览器内核的一些问题
- video标签正在播放状态时，路由无法跳转，路由离开之前需暂停video播放
- 安卓微信app支持video小窗播放，需增加x5-playsinlines属性

```html
<video
  v-if="liveData.videoSrc"
  ref="video"
  class="live-player"
  :src="liveData.videoSrc"
  :poster="liveData.coverImageUrl"
  preload
  controls
  x5-playsinline
  webkit-playsinline="isiPhoneShowPlaysinline"
  playsinline="isiPhoneShowPlaysinline">
</video>
```
#### 9、本地测试微信授权，微信分享功能
- 1、nginx监听80端口，server_name设为微信公众平台网页授权域名/JS接口安全域名，路径proxy_pass设为localhost:port
- 2、本地hosts 配置域名解析，127.0.0.1 微信公众平台网页授权域名/JS接口安全域名

## 跨域相关
#### 10、谨慎使用crossorigin属性
我的html页面包含如下一段代码
```html
<script src="https://res.wx.qq.com/open/js/jweixin-1.4.0.js"></script>
```
由于公司cdn js资源地址请求都设置了响应头Access-Control-Allow-Origin:* ，为了捕捉更详细的错误信息，服务端做了统一处理，渲染后变成了如下：
```html
<script crossorigin src="https://res.wx.qq.com/open/js/jweixin-1.4.0.js"></script>
```
浏览器直接报如下类似的跨域请求错误
```bash
Access to script at 'https://res.wx.qq.com/open/js/jweixin-1.4.0.js' from origin 'http://localhost:5000' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header has a value 'http://open.weixin.qq.com' that is not equal to the supplied origin.
```
解决办法：正则匹配，请求公司域名的js地址为script标签加上crossorigin属性 

[推荐文章](https://www.jianshu.com/p/a45c9d089c93)

## 工具函数&组件相关
#### 11、jsonp请求工具函数封装总结
- 无法获取script请求的响应内容
- 如果script标签src地址响应的不是正确的函数执行代码，做超时处理
- qs模块字符串序列化对象时，会对带协议的字段执行encodeURIComponent，需特别注意

```js
import qs from 'qs';

let id = 1;

/**
 * jsonp请求
 */
export default function(url, params = {}) {
  const callbackName = `jsonp${Date.now()}${id++}`;
  const el = document.createElement('script');

  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      reject({
        message: '请求超时~',
        url,
      });
    }, 5000);

    window[callbackName] = res => {
      clearTimeout(timeoutId);
      resolve(res || {});
    };

    el.onerror = function() {
      clearTimeout(timeoutId);
      document.body.removeChild(el);

      reject({
        message: '当前网络不佳，请稍后再试',
        url,
      });
    };

    el.onload = function(e) {
      clearTimeout(timeoutId);
      document.body.removeChild(el);
    };

    let src = `${url.split('#')[0]}?callback=${callbackName}`;

    if (Object.keys(params || {}).length > 0) {
      src = `${src}&${qs.stringify(params)}`;
    }

    el.src = src;
    document.body.appendChild(el);
  });
}
```
#### 12、使用canvas开发一个loading组件总结
- 新思路：当前的弧度由当前时间的秒跟毫秒决定

```js
export default {
  mounted() {
    this.init();
  },
  methods: {
    init() {
      const canvas = this.$refs['canvas'];
      const ctx = canvas.getContext('2d');

      let size = this.size;
      let lineWidth = this.lineWidth;

      if (navigator.userAgent.toLowerCase().match('iphone')) {
        size *= window.devicePixelRatio;
        lineWidth *= window.devicePixelRatio;
      } else {
        size *= 2;
        lineWidth *= 2;
      }

      const radius = size / 2;
      canvas.width = size;
      canvas.height = size;
      ctx.lineWidth = lineWidth;

      let reqAnimFrame = window.requestAnimationFrame || window.webkitRequestAnimationFrame;

      if (typeof reqAnimFrame === 'undefined') {
        reqAnimFrame = callback => {
          setTimeout(callback, 16);
        };
      }

      const draw = () => {
        ctx.clearRect(0, 0, size, size);
        ctx.save();
        ctx.translate(radius, radius);

        const time = new Date();
        const currentTimeRadian =
          ((2 * Math.PI) / 60) * time.getSeconds() + ((2 * Math.PI) / 60000) * time.getMilliseconds();
        ctx.rotate(currentTimeRadian * this.speed);
        ctx.beginPath();
        ctx.arc(0, 0, radius - ctx.lineWidth, 0, Math.PI * 1.5, false);
        ctx.strokeStyle = this.strokeColor;
        ctx.stroke();

        ctx.restore();
        reqAnimFrame(draw);
      };

      reqAnimFrame(draw);
    },
  }
};
```