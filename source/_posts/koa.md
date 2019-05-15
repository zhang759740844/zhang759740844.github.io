title: Koa的基本使用
date: 2019/4/23 14:07:12  
categories: JavaScript
tags: 

 - 学习笔记
	
---

本篇是学习 node.js 基础框架 Koa 的学习总结

<!--more-->

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



## Koa 中间件

中间件就是匹配路由前和匹配路由后做的一系列操作。中间件的功能包括：

- 修改请求和响应对象
- 终止请求和响应循环
- 调用堆栈中的下一个中间件

中间件按照功能可以分为：

- 应用级中间件
- 路由中间件
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

> 利用这种方式可以做消息转发

## koa 路由中间件

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

#### 获取 post 传值

安装 koa-bodyparser

```bash
npm install --save koa-bodyparser
```

```js
const Koa = require('koa')
const Router = require('koa-router')
const BodyParser = require('koa-bodyparser')
const app = new Koa()
const router = new Router()
const bodyParser = new BodyParser()

// 使用 bodyParser 中间件
app.use(bodyParser)

router.post('/', async (ctx, next) 
	// 通过 ctx.request.body 获取表单提交的数据
	// 获取的 post 的 body 已经被转为对象
	console.log(ctx.request.body)     
})

app.use(router.routes())
	.use(router.allowedMethods())

app.listen(3000, () => {
  console.log('starting at port 3000')
})
```

#### get 和 post 都支持

如果一个请求，既要支持 get 又要支持 post 也是可以的，可以使用 `.all` 方法：

```js
router.all('/:pid', async (ctx) => {
  console.log(ctx.url)
  const buffer = await fse.readFile('./test.json')
  ctx.body = buffer.toString()
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



## 设置 Cookie 和 Session

### 设置 cookie

#### 设置 cookie

```js
router.get('/', async (ctx) => {
  ctx.cookies.set('userInfo', 'zachary', {
    // 设置过期时间
    maxAge: 60*1000*60,
    // 配置可以访问的页面
    path: '/news',
    // 当有多个子域名的时候使用，也可以不写，默认是请求的域名
    domain: '.baidu.com',
    // true 表示这个 cookie 只有服务器端可以方位，false 表示客户端也可以访问
    httpOnly: true
  })
})
```

#### 获取 cookie

```js
router.get('/', async (ctx) => {
  console.log(ctx.cookies.get('userInfo'))
})
```

> 如果客户端需要拿到 cookie，那么必须设置 httpOnly 为 false

### 设置 Session

**session** 是另一种记录客户状态的机制，不同的是 **Cookie** 保存在客户端浏览器中，而
**session** 保存在服务器上。

#### Session 的工作流程

当浏览器访问服务器并发送第一次请求时，服务器端会创建一个 session 对象，生
成一个类似于 key,value 的键值对， 然后将 key(cookie)返回到浏览器(客户)端，浏览
器下次再访问时，携带 key(cookie)，找到对应的 session(value)。 客户的信息都保存
在 session 中。

#### 设置 Session

```bash
npm install --save koa-session
```

使用：

```js
// 引入
const session = require('koa-session')

// 设置中间件
app.keys = ['some secret hurr']

const CONFIG = {
	key: 'koa:sess', //cookie key (default is koa:sess)
	maxAge: 86400000, // cookie 的过期时间 maxAge in ms (default is 1 days) 
	httpOnly: true, //cookie 是否只有服务器端可以访问 httpOnly or not (default true) signed: true, //签名默认 true
	rolling: false, //在每次请求时强行设置cookie，这将重置cookie过期时间(默认:false)
	renew: true, // 快过期的时候重新设置
}

app.use(session(CONFIG, app))

// 设置
router.get('/login', async (ctx) => {
  // 设置 session
  ctx.session.userInfo = 'zachary'
  ctx.body = '登录成功'
})

// 获取
router.get('/', async (ctx) => {
	console.log(ctx.session.userInfo)
})
```

`koa-session` 的原理就是在内存中生成一个 session 的散列表。每次生成一个随机的字符串，并将其作为 cookie 下发给客户端，key 对应于 config 中的 key。设置的 `app.keys` 应该就是加密用的秘钥。

当 session 量比较大的时候，就需要考虑通过使用外部存储的方式进行存储。比方说 mysql，redis 等

#### Session 和 Cookie 的区别

1. cookie 数据存放在客户的浏览器上，session 数据放在服务器上。
2. cookie 会暴露用户的信息，所以使用 session，只在 cookie 中保存一个 id，每次请求通过这个 id 从 session 散列表中获取相应的信息。
3. 单个 cookie 保存的数据不能超过 4K，很多浏览器都限制一个站点最多保存 20 个 cookie。

## 访问静态文件

如果不想使用模板渲染或者 nuxt.js 这种服务端渲染框架。可以尝试直接访问 vue 生成的 dist 文件。

### 生成 Vue 的 dist 目录

通过 vue-cli 生成 Vue 项目，通过 `npm run build` 生成 dist 目录，把 dist 目录整个拷贝到 koa 所在目录下：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/koa_1.png?raw=true)

### 访问静态文件

访问静态文件需要使用相关中间件 `koa-static`:

```js
const Koa = require('koa')
const app = new Koa()
// 访问当前目录下的 dist 目录中的文件
const static = require('koa-static')('./dist')

// 配置静态资源文件路径
app.use(static)

app.listen(3000, () => {
  console.log('正在监听3000端口')
})
```

此时当你访问 `localhost:3000` 的时候，koa 就会将 `dist/index.html` 返回显示。

> 如果同时使用了路由中间件，需要把资源文件的中间件先于路由中间件注册。否则会被路由拦截

## 路由模块化

把所有路由放在一个文件中显然是不好维护的，因此需要针对路由进行子路由拆分。

### 实现

#### 创建子路由

首先创建 routes 文件夹，创建子路由文件，比如命名为 admin.js。在其中添加子路由：

```objc
// 在 /routes/admin.js 目录下

const Router = require('koa-router')
const router = new Router()

router.get('/', async (ctx) => {
	ctx.body = 'admin 首页'    
})
  
router.get('/user', async (ctx) => {
	ctx.body = 'admin 用户页'    
})
  
module.exports = router.routes()
```

#### 引入子路由

```js
// app.js

const router = require('koa-router')()
const app = require('koa')()

// 引入子路由
const admin = require('./routes/admin.js')

// 用子路由替代了原来的 async 方法
// 注意，一定是 use，不是 get 或者 post，我在这里卡了很久
router.use('/admin', admin)

app.use(router.routes()).use(router.allowedMethods())
```

这样以下请求就会映射到相应子路由中：

```
http://localhost:3000/admin
http://localhost:3000/admin/user
```

## 实践

### node 自动重启

开发的时候每次修改都要手动重新 run 一次 node 会非常的麻烦。因此需要一个能自动重启的库的帮助。这个库就是 nodemon:

```bash
npm install -g nodemon
```

全局安装 nodemon，然后启动的时候不要通过 node 了，而是 nodemon：

```bash
nodemon app.js
```

### 文件操作

使用 node 提供的 fs 操作文件比较复杂，尤其是使用 koa 的时候本生不支持 promise 写法。

fs-extra 是一个比较完善的文件操作封装库：

```bash
npm install --save fs-extra
```

使用起来也和 fs 无异。下面例子中读取本地的 json 文件并返回：

```js
const fse = require('fs-extra')

router.get('/', async (ctx) => {
  const buffer = await fse.readFile('./test.json')
  ctx.body = buffer.toString()
})
```

### 发送请求

我们使用 koa 接收客户端请求。但是很多时候，我们需要对请求做转发。因此，需要使用 nodejs 发送请求。

发送请求可以使用库 *request-promise*。它是 *request* 的 promise 版本：

```bash
npm install --save request-promise
# request 是 request-promise 的依赖项，需要自己手动安装
npm install --save request
```

请求方式很简单：

```js
app.use(async (ctx, next) => {
	// 先让路由响应
  await next()
	// 如果没有一个路由相应
  if (ctx.status === 404) {
    const url = `https://colorlab.lian-shang.cn${ctx.url}`
    try {
      // 尝试请求转发
      let obj = await request.get(url)
      ctx.body = obj
    } catch (err) {
      // 转发请求发送失败
      console.log('发生了error')
      console.log(err.statusCode)
    }
  }
})
```

要注意， `await` 中发生的 error 需要通过 try…catch 捕获。

### 设置环境变量

node 中的环境变量在 `process.env` 中保存，使用库 `cross-env` 可以方便设置兼容 windows 的环境变量：

```bash
npm install --save-dev cross-env
```

package.json 中设置环境变量：

```json
{
  "scripts": {
    "build": "cross-env NODE_ENV=production webpack --config build/webpack.config.js"
  }
}
```

`NODE_ENV` 环境变量将由 cross-env 设置。打印 `process.env.NODE_ENV === 'production'` 为 true



### 允许跨域

跨域的请求需要借助第三方的库 `koa-cors` 进行允许跨域设置：

```bash
npm install koa-cors
```

使用：

```js
app.use(cors({
  origin: function (ctx) {
    if (ctx.url === '/cors') {
      return '*' // 允许来自所有域名请求
    }
    return 'http://localhost:3000'
  },
  exposeHeaders: ['WWW-Authenticate', 'Server-Authorization'],
  credentials: true,
  allowMethods: ['GET', 'POST', 'DELETE'], // 设置允许的HTTP请求类型
  allowHeaders: ['Content-Type', 'Authorization', 'Accept']
}))
```

其中 origin 字段对应于 `access-control-allow-origin`，通过它设置哪些站点发起哪些请求可以进行跨域。上面🌰表示，请求路径为 `/cors` 的这个请求可以进行跨域，但是不能获取 cookie 信息，而 `http://localhost:3000` 这个域名下的所有请求都可以进行跨域，且可以获取 cookie 信息，是受信的站点。

如果设置了 `access-control-allow-origin` 为 `*`，那么就是允许跨域了。但是跨域的客户端请求，无法携带该域名下的 cookie 信息给服务端。

必须设置 `access-control-allow-origin` 为一个特定的域名，而不是 `*`。这样 `Access-Control-Allow-Credentials` 才会被默认置位 true，才可以跨域使用 cookie。

> 所以这是一个三级保证。没有 *，那么无法发出请求，没有特定的域名，没有发送带 cookie 的请求。

### 连接 Mysql

Nodejs 连接 Mysql 需要 install 相关的库：

```bash
npm install --save mysql
```

基本使用：

```js
const mysql = require('mysql')

// 创建连接对象
const con = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: '',
  port: '3306',
  database: 'your_database'
})

// 开始连接
con.connect()

// 执行简单的sql
const sql = 'select * from your_databse'
con.query(sql, (err, result) => {
  if (err) {
 		console.error(err)
    return
  }
  console.log(result)
})

// 关闭连接
con.end()
```

#### 防止 sql 注入

sql 如果是通过字符串拼接的方式注入的参数，那么容易引起 sql 注入。Nodejs 中的 mysql 库引入了 `escape` 方法，会对参数进行转义：

```sql
select id from User where username = ${escape(username)}
```



### redis

redis 是一种内存存储，可以配合 mysql 进行频繁操作的数据的存储。

#### 安装 redis

##### 安装

```bash
brew install redis
```

##### 启动 server

```bash
redis-server
```

##### 启动 client

```bash
redis-client
```

存取值

```bash
# 设置键值对
> set myname zachary
# 取值
> get myname
```



#### 使用 redis

#### 安装 redis nodejs 模块

```bash
npm install redis --save
```

#### node 中使用

```js
const redis = require('redis')

// 创建客户端
// 默认是 6379 端口
const redisClient = redis.createClient(6379, '127.0.0.1')
redisClient.on('error', err => {
  console.log(err)
})

// 测试使用
redisClient.set('myname', 'zachary', redis.print)
redisClient.get('myname', (err, val) => {
  if (err) {
    console.error(err)
    return
  }
  console.log('val', val)
  
  // 退出 redis
  redisClient.quit()
})

```

### PM2

- 进程守护，系统奔溃自动重启
- 启动多进程，充分利用 CPU 和内存
- 自带日志记录

#### 下载

```bash
npm install pm2 -g
```

#### 启动

开发环境下使用 `nodemon`，线上环境可以使用 `pm2`:

```bash
pm2 start app.js
```

#### 常用命令

```bash
# 查看进程列表
pm2 list
# 重启进程
pm2 restart <AppName>/<id>
# 停止进程
pm2 stop <AppName>/<id>
# 删除进程
pm2 delete <AppName>/<id>
# 查看进程信息
pm2 info <AppName>/<id>
# 查看日志(console.log/error/warn)
pm2 log <AppName>/<id>
# 监控
pm2 monit <AppName>/<id>
```

### 命令行

node 写脚本需要配合 shell 语法，`shelljs` 是一个封装的很晚上的库

