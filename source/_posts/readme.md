## Genji4mp

Genji4mp 是一个帮助开发者更快捷开发小程序项目的微形框架，对小程序项目侵入小，只有四个文件，解决小程序日常开发中的一些问题

### 小程序开发存在的问题

1. 所有方法和数据都声明在一个对象中交由 Page 注册，无法职责分离。
2. 调用小程序方法的成功和失败回调都需要作为参数传入。
3. 跳转页面需要拼接一个很长的url，比如 `pages/xxxx/xxxx`。
4. 如果跳转页面的 url 中含有特殊字符比如 `&` 就会被小程序自动拆开当成两个参数。
5. 页面返回无法进行回调以及带回参数。
6. 跨页面的状态只能通过 globalData 传递。
7. 不支持 mixin 生命周期，无法做一些统一设置
8. 按钮以及输入等回调需要写很长的诸如 `event.currentTarget.dataset` 或者 `event.detail.value`

### 解决方法

#### 1. 更改小程序的注册方法

提供了 `$Page` 对象，将原本小程序 `Page({})` 的注册方式改为：

```js
// 职责分类
const props = {}
const data = {}
const lifecycle = {}
const privateMethods = {}
const viewAction = {}

// 注册对象
$Page.register(props, data, lifecycle, privateMethods, viewAction)
```

- props: 负责页面中不会影响界面显示的参数
- data: 也就是小程序提供的 data，所有与页面相关的数据都存放在这里
- lifecycle: 生命周期相关的方法放在这里
- viewAction: 和页面交互的方法放在这里
- privateMethod: 其他与上面都无关的方法放在这里

这种分类不只是代码规范上的拆分。对于不同职责代码的分类能更容易的对同一类代码进行统一 hook。

#### 2. 更改调用小程序提供方法的方式

提供了 `$wx` 变量，替换调用小程序方法的全局变量 `wx`中。调用方式采用 Promise 替代原来的 success 和 failure 入参。例：

```js
// 原本的小程序调用方式：
wx.navigateTo({url: 'xxx', success: ()=>{}, failure: ()=>{})
// 现在的调用方式：
$wx.navigateTo(url, param).then(() => {
}).catch(err => {
})
```

#### 3. 解决跳转页面需要填写的很长的 url

原本页面需要记住很长的页面路径，以及拼接参数的操作比较繁琐

```js
wx.navigateTo({url: `pages/my-view/my-view?param1=1&param2=2`})
```

现在需要分成几步完成：

1. 在根目录下创建 router.js，写入如有信息：

```js
// router.js
export default {
  myView1: 'my-view1',
  myView2: 'my-view2'
}
```

2. 在小程序的全局注册文件 `app.js` 中注册路由：

```js
import router from 'router'
// app.js
App({
  onLaunch () {
    $wx.registerRouter(router)
  }
})
```

3. 在需要的地方调用跳转方法：

```js
$wx.navigateTo($wx.router.myView1, {param1: 1, param2: 2})
```



