# home键项目总结

## 需求
由大数据发起的一个项目，在app中的某个h5页面，用户按下home键需要上报一个页面离开日志，再次回到h5页面时，上报一个进入页面日志

## 技术分析
- 使用浏览器端visibilityChange事件实现，但是app webview不支持，没用人或者会去实现，就像微信webview已经实现
- 通过现有的jsBridge做交互实现，监听viewOnPause报离开，监听viewOnResume报进入页面日志

## 后期遇到的一些问题
- 关闭webview需要上报页面离开日志，但是执行关闭webview方法，客户端会触发viewOnPause事件(页面已经监听了该事件会上报页面离开日志)，这时需要拦截关闭webview方法，且移除viewOnPause事件监听器
- 在ios11.3 app uiwebview h5页面中，viewOnResume，viewOnPause 回调中不能执行同步clearTimeout语句，否则app crash
```js
{
  methods: {
    viewOnResumeListener() {
      // 在ios11.3 app uiwebview h5页面中 同步执行了clearTimeout方法, app crash
      clearTimeout(timeoutId)

      // 下面代码正常
      setTimeout(() => {
        clearTimeout(timeoutId)
      }, timeout)
    }
  }
}
```
