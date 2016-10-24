title: NodeJs Web开发
date: 2016/10/5 10:07:12  
categories: JavaScript
tags:
	- 学习笔记
	- NodeJs

---

本篇是NodeJs的学习笔记，参考自[廖雪峰的博客](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/001434501579966ab03decb0dd246e1a6799dd653a15e1b000)

<!--more-->

## koa
koa是Express的下一代基于Node.js的web框架，目前有1.x和2.0两个版本。和koa 1相比，koa2完全使用Promise并配合`async`来实现异步。

### koa入门
#### 创建koa2工程
首先，创建一个目录文件夹`hello-koa`,然后创建`app.js`,写入：

```javascript
// 导入koa，和koa 1.x不同，在koa2中，我们导入的是一个class，因此用大写的Koa表示:
const Koa = require('koa');

// 创建一个Koa对象表示web app本身:
const app = new Koa();

// 对于任何请求，app将调用该异步函数处理请求：
app.use(async (ctx, next) => {
    await next();
    ctx.response.type = 'text/html';
    ctx.response.body = '<h1>Hello, koa2!</h1>';
});

// 在端口3000监听:
app.listen(3000);
console.log('app started at port 3000...');
```

对于每一个http请求，koa将调用我们传入的异步函数来处理：

```javascript
async (ctx, next) => {
    await next();
    // 设置response的Content-Type:
    ctx.response.type = 'text/html';
    // 设置response的内容:
    ctx.response.body = '<h1>Hello, koa2!</h1>';
}
```

其中，参数`ctx`是由koa传入的封装了request和response的变量，我们可以通过它访问request和response，`next`是koa传入的将要处理的下一个异步函数。 
上面的异步函数中，我们首先用`await next();`处理下一个异步函数，然后，设置response的Content-Type和内容。 
由`async`标记的函数称为异步函数，在异步函数中，可以用`await`调用另一个异步函数，这两个关键字将在ES7中引入。

那么koa这个包怎么装，`app.js`才能正常导入它？
在`hello-koa`这个目录下创建一个`package.json`，这个文件描述了我们的`hello-koa`工程会用到哪些包:

```javascript
{
	"name": "hello-koa2",
    "version": "1.0.0",
    "description": "Hello Koa 2 example with async",
    "main": "start.js",
    "scripts": {
        "start": "node start.js"
    },
    "dependencies": {
        "babel-core": "6.13.2",
        "babel-polyfill": "6.13.0",
        "babel-preset-es2015-node6": "0.3.0",
        "babel-preset-stage-3": "6.5.0",
        "koa": "2.0.0"
    }
}
```

其中，`dependencies`描述了我们的工程依赖的包以及版本号。然后，我们在`hello-koa`目录下执行`npm install`就可以把所需包以及依赖包一次性全部装好。
注意，任何时候都可以直接删除整个`node_modules`目录，因为用`npm install`命令可以完整地重新下载所有依赖。并且，这个目录不应该被放入版本控制中。

另外，这里的`"main": "start.js"`将在下面讲到。

但是由于现在的Node.js只支持ES6，不支持ES7，无法识别新的`async`语法。需要使用Babel把ES7代码“转换”为ES6代码。

#### Bebel
Babel是一个JavaScript编写的转码器，它可以把高版本的JavaScript代码转换成低版本的JavaScript代码，并保持逻辑不变，这样就可以在低版本的JavaScript环境下运行。

例如，我们用ES7编写的JavaScript代码，用Babel转换成ES6以后，就可以在Node环境下执行。如果某些JavaScript代码需要在更低版本的环境下执行，例如IE 6，就可以用Babel转换成ES5的代码。

用Babel转码时，需要指定presets和plugins。
presets是规则，我们`stage-3`规则，`stage-3`规则是ES7的stage 0~3的第3阶段规则。
plugins可以指定插件来定制转码过程，一个preset就包含了一组指定的plugin。

我们编写一个`start.js`文件，在这个文件中，先加载`babel-core/register`，再加载`app.js`：

```javascript
var register = require('babel-core/register');

register({
    presets: ['stage-3']
});

require('./app.js');
```

现在我们在`hello-koa`目录下又多了一个`start.js`文件。在`npm install`后，可以直接执行`node start.js`也可以用`npm start`启动。`npm start`命令会让npm执行定义在`package.json`文件中的`start`对应命令

为什么先加载`babel-core/register`，再加载`app.js`，魔法就会生效？原因是第一个`require()`是Node正常加载`babel-core/register`的过程，然后，Babel会用自己的`require()`替换掉Node的`require()`，随后用`require()`加载的所有代码均会被Babel自动转码后再加载。

#### koa middleware
koa的核心代码是：

```javascript
app.use(async (ctx, next) => {
    await next();
    ctx.response.type = 'text/html';
    ctx.response.body = '<h1>Hello, koa2!</h1>';
});
```

每收到一个http请求，koa就会调用通过`app.use()`注册的`async`函数，并传入`ctx`和`next`参数。

我们可以对`ctx`操作，并设置返回内容。但是为什么要调用`await next()`？
原因是`koa`把很多`async`函数组成一个处理链，每个`async`函数都可以做一些自己的事情，然后用`await next()`来调用下一个`async`函数。我们把每个`async`函数称为middleware，这些middleware可以组合起来，完成很多有用的功能。

例如，可以用以下3个middleware组成处理链，依次打印日志，记录处理时间，输出HTML：

```javascript
app.use(async (ctx, next) => {
    console.log(`${ctx.request.method} ${ctx.request.url}`); // 打印URL
    await next(); // 调用下一个middleware
});

app.use(async (ctx, next) => {
    const start = new Date().getTime(); // 当前时间
    await next(); // 调用下一个middleware
    const ms = new Date().getTime() - start; // 耗费时间
    console.log(`Time: ${ms}ms`); // 打印耗费时间
});

app.use(async (ctx, next) => {
    await next();
    ctx.response.type = 'text/html';
    ctx.response.body = '<h1>Hello, koa2!</h1>';
});
```

middleware的顺序很重要，也就是调用`app.use()`的顺序决定了middleware的顺序。如果一个middleware没有调用`await next()`，后续的middleware将不再执行。

### 处理URL
前面的工程中，没有对URL进行判定，实际上，一个网站，对于不同的网页`request`应该返回不同的`response`。譬如像这样：

```javacsript
app.use(async (ctx, next) => {
    if (ctx.request.path === '/') {
        ctx.response.body = 'index page';
    } else {
        await next();
    }
});
```

但是实际使用中，不能一个个去判断`request.path`，应该有一个能集中处理URL的middleware，它根据不同的URL调用不同的处理函数。

#### koa-router
使用`koa-router`这个中间件，来处理URL映射。

先在`package.json`中添加依赖项，并`npm install`:

```javascript
"koa-router": "7.0.0"
```

接下来，我们修改`app.js`，使用`koa-router`来处理URL：

```javascript
const Koa = require('koa');

// 注意require('koa-router')返回的是函数:
const router = require('koa-router')();

const app = new Koa();

// log request URL:
app.use(async (ctx, next) => {
    console.log(`Process ${ctx.request.method} ${ctx.request.url}...`);
    await next();
});

// add url-route:
router.get('/hello/:name', async (ctx, next) => {
    var name = ctx.params.name;
    ctx.response.body = `<h1>Hello, ${name}!</h1>`;
});

router.get('/', async (ctx, next) => {
    ctx.response.body = '<h1>Index</h1>';
});

// add router middleware:
app.use(router.routes());

app.listen(3000);
console.log('app started at port 3000...');
```

注意导入`koa-router`的语句最后的`()`是函数调用。

#### 处理post请求
用`router.get('/path', async fn)`处理的是get请求。如果要处理post请求，可以用`router.post('/path', async fn)`。

post请求通常会发送一个表单，或者JSON，它作为request的body发送，但无论是Node.js提供的原始request对象，还是koa提供的request对象，都不提供解析request的body的功能！所以，又需要引入另一个middleware来解析原始request请求，然后，把解析后的参数，绑定到`ctx.request.body`中。

`koa-bodyparser`可以实现解析功能，我们在`package.json`中添加依赖项：

```javascript
"koa-bodyparser": "3.2.0"
```

下面，修改`app.js`，引入`koa-bodyparser`：

```javascript
const bodyParser = require('koa-bodyparser');
```

在合适的位置加上注册，由于middleware的顺序很重要，这个`koa-bodyparser`必须在`router`之前被注册到`app`对象上:

```javacsript
app.use(bodyParser());
```

现在我们就可以处理post请求了。写一个简单的登录表单：

```javascript
router.get('/', async (ctx, next) => {
    ctx.response.body = `<h1>Index</h1>
        <form action="/signin" method="post">
            <p>Name: <input name="name" value="koa"></p>
            <p>Password: <input name="password" type="password"></p>
            <p><input type="submit" value="Submit"></p>
        </form>`;
});

router.post('/signin', async (ctx, next) => {
    var
        name = ctx.request.body.name || '',
        password = ctx.request.body.password || '';
    console.log(`signin with name: ${name}, password: ${password}`);
    if (name === 'koa' && password === '12345') {
        ctx.response.body = `<h1>Welcome, ${name}!</h1>`;
    } else {
        ctx.response.body = `<h1>Login failed!</h1>
        <p><a href="/">Try again</a></p>`;
    }
});
```

用`var name = ctx.request.body.name || ''`拿到表单的`name`字段，如果该字段不存在，默认值设置为`''`。

#### 重构
`app.js`是工程的入口，不可能将所有的`post``get`注册以及回调都放在该文件内，可以自己创建一个`controllers `文件夹，将各个注册及回调放在其中，譬如新建一个有关登录的`index.js`:

```javascript
var fn_index = async (ctx, next) => {
    ctx.response.body = `<h1>Index</h1>
        <form action="/signin" method="post">
            <p>Name: <input name="name" value="koa"></p>
            <p>Password: <input name="password" type="password"></p>
            <p><input type="submit" value="Submit"></p>
        </form>`;
};

var fn_signin = async (ctx, next) => {
    var
        name = ctx.request.body.name || '',
        password = ctx.request.body.password || '';
    console.log(`signin with name: ${name}, password: ${password}`);
    if (name === 'koa' && password === '12345') {
        ctx.response.body = `<h1>Welcome, ${name}!</h1>`;
    } else {
        ctx.response.body = `<h1>Login failed!</h1>
        <p><a href="/">Try again</a></p>`;
    }
};

module.exports = {
    'GET /': fn_index,
    'POST /signin': fn_signin
};
```

这个`index.js`通过`module.exports`把两个URL处理函数暴露出来。

那么现在应该要做的是在`app.js`中，让它自动扫描`controllers`目录，找到所有`js`文件，导入，然后注册每个URL：

```javascript
// 先导入fs模块，然后用readdirSync列出文件
// 这里可以用sync是因为启动时只运行一次，不存在性能问题:
var files = fs.readdirSync(__dirname + '/controllers');

// 过滤出.js文件:
var js_files = files.filter((f)=>{
    return f.endsWith('.js');
}, files);

// 处理每个js文件:
for (var f of js_files) {
    console.log(`process controller: ${f}...`);
    // 导入js文件:
    let mapping = require(__dirname + '/controllers/' + f);
    for (var url in mapping) {
        if (url.startsWith('GET ')) {
            // 如果url类似"GET xxx":
            var path = url.substring(4);
            router.get(path, mapping[url]);
            console.log(`register URL mapping: GET ${path}`);
        } else if (url.startsWith('POST ')) {
            // 如果url类似"GET xxx":
            var path = url.substring(5);
            router.post(path, mapping[url]);
            console.log(`register URL mapping: POST ${path}`);
        } else {
            // 无效的URL:
            console.log(`invalid URL: ${url}`);
        }
    }
}
```

#### Controller Middleware
最后，我们把扫描`controllers`目录和创建`router`的代码从`app.js`中提取出来，作为一个简单的`middleware`使用，命名为`controller.js`：

```javascript
const fs = require('fs');

function addMapping(router, mapping) {
    ...
}

function addControllers(router, dir) {
    ...
}

module.exports = function (dir) {
    let
        controllers_dir = dir || 'controllers', // 如果不传参数，扫描目录默认为'controllers'
        router = require('koa-router')();
    addControllers(router, controllers_dir);
    return router.routes();
};
```

这样一来，我们在`app.js`的代码又简化了：

```javascript
...

// 导入controller middleware:
const controller = require('./controller');

...

// 使用middleware:
app.use(controller());

...
```

经过重新整理后的工程`url2-koa`目前具备非常好的模块化，所有处理URL的函数按功能组存放在`controllers`目录，今后我们也只需要不断往这个目录下加东西就可以了，`app.js`保持不变。






























