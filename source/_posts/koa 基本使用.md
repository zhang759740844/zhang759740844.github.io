## koa 基本使用

安装

```bash
npm install --save koa
```

创建 `app.js`:

```js
const Koa = require('koa')
const app = Koa()

// express 的老写法
// app.use(function (req, res) {
// 	res.send('haha')
// })

app.use(async ctx => {
  ctx.body = 'hahh'
})

app.listen(3000)
```



## koa 路由

安装：

```bash
npm install --save koa-router
```

### 静态路由

#### 示例

```js
const Koa = require('koa')
const Router = require('koa-router')
const app = new Koa()
const router = new Router()

// 配置路由
router.get('/', async (ctx) => {
  ctx.body = '首页'  // 相当于原生里的 res.writeHedr() res.end()
}).get('/news', async (ctx) => {
  ctx.body = '这是一个新闻页面'
})

// 
app.use(router.routes()) // 启动路由
	.use(router.allowedMethods())  // 可配置可不配置，建议配置。会根据 ctx.status 设置 response 响应头
app.listen(3000)
```

#### 获取 get 传值

```js
router.get('/news', async (ctx) => {
	// 从 ctx 中读取get传值,获取的是对象；等同于 ctx.request.query
  console.log(ctx.query)
  // 获取 url; 等同于 ctx.request.url
  console.log(ctx.url)
  // 获取整个request
  console.log(ctx.request)
})
```

### 动态路由

静态路由要求路径完全匹配，动态路由可以前部匹配，后部可以动态获取。koa 中通过 `:` 作为动态路由的标识：

```js
// 动态路由匹配 /news/xxxx
router.get('/news/:aid', async (ctx) => {
	// 动态路由的传值
  console.log('以下是动态路由路由:' + (ctx.params).toString())
})

// 比如输入 localhost:3000/news/haha
// => 以下是动态路由的路由: {aid: 'haha'}
```

动态路由可以匹配多个值。如 `/news/:aid/:cid`

## Koa 中间件

中间件就是匹配路由前和匹配路由后做的一系列操作。中间件的功能包括：

- 修改请求和响应对象
- 终止请求和响应循环
- 调用堆栈中的下一个中间件

中间件按照功能可以分为：

- 应用级中间件
-  路由中间件
- 错误处理中间件
- 第三方中间件

### 应用级中间件

应用级中间件，我觉得可以叫做全局中间件，就是所有请求都要经过的。举例：对于每一个请求，打印它的请求时间：

```js
const Koa = require('koa')
const Router = require('koa-router')
const app = new Koa()
const router = new Router()

// 应用级中间件
app.use(async (ctx, next) => {
  // 打印时间
  console.log(new Date())
  // 等待后续中间件的执行
  await next()
})

router.get('/', async (ctx, next) {
	ctx.body = 'hello koa'           
})
// 总的路由的中间件也是应用级中间件
app.use(router.routes())
	.use(router.allowedMethods())

app.listen(3000, () => {
  console.log('starting at port 3000')
})
```

类比 redux 中的中间件，中间件通过 `use` 注册，内部保存在一个中间件的数组中。可以通过调用 `next()` 方法执行下一个中间件。如果不调用，那么后面的中间件都不会继续执行。

### 路由级中间件

路由级中间件可以使一个请求匹配多个路由:

```js
const Koa = require('koa')
const Router = require('koa-router')
const app = new Koa()
const router = new Router()


router.get('/', async (ctx, next) {
	console.log('调用 next 匹配下一个路由')
	// 执行 next 方法，可以匹配下一个路由
	await next()
})

router.get('/', async (ctx, next) {
	ctx.body = 'hello koa again'           
})

app.use(router.routes())
	.use(router.allowedMethods())

app.listen(3000, () => {
  console.log('starting at port 3000')
})
```

不同于应用级中间件，路由级中间件只针对路由。可以大概猜测出，应用级中间件中的 `next` 是 koa 提供的，而路由级中间件的 `next` 方法应该是 koa-router 提供的。

### 错误处理中间件

其实就是一个应用级中间件。错误处理用在没有匹配任何路由，返回一个 404 页面：

```js
const Koa = require('koa')
const Router = require('koa-router')
const app = new Koa()
const router = new Router()

// 全局中间件。这个地方用来处理错误
app.use(async (ctx, next)) {
  // 执行普通的路由匹配
  await next()
  
  // 没有匹配上任何路由，那么状态就是404
  if (ctx.status === 404) {
    ctx.body = '这是一个404页面'
  }
}

router.get('/', async (ctx, next) {
	ctx.body = 'hello koa again'           
})

app.use(router.routes())
	.use(router.allowedMethods())

app.listen(3000, () => {
  console.log('starting at port 3000')
})
```

