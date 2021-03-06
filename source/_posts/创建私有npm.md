title: 如何创建私有npm
date: 2018/8/13 14:07:12  
categories: JavaScript
tags: 
	- npm
	
---

当需要使用npm管理包，并且这些包不适合放到公网上时，就需要搭建私有npm了。搭建私有npm的方式很多。一般来说，比较推荐使用 verdaccio。这是sinopia 的 fork。sinopia 已经很久没有更新了。

<!--more-->

## 准备工作

### 安装

使用命令行安装：

```bash
sudo npm install -g verdaccio --unsafe-perm
```

安装完毕后，会在用户目录下的 `.config` 目录下生成一个 `verdaccio` 文件夹，配置文件 `config.yaml` 就在这个文件夹内。

### 启动

直接在命令行键入 `verdaccio` 启动。控制台会答应几行warn：

```bash
 warn --- config file  - /Users/zachary/.config/verdaccio/config.yaml
 warn --- Plugin successfully loaded: htpasswd
 warn --- Plugin successfully loaded: audit
 warn --- http address - http://localhost:4873/ - verdaccio/3.5.1
```

这其实是提示信息，告诉你 verdaccio 已经在 `localhost:4873` 端口启动。你可以打开查看：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_1.png?raw=true)

### ip 访问

上面启动后，我们会发现可以使用 `localhost:4873` 访问，但是不能使用本机 ip 访问。这样肯定是不行的。我们需要配置 `config.yaml`。

在 `config.yaml` 文件内添加监听端口：

```shell
listen: 0.0.0.0:4873  # listen on all addresses 
```

这样其它机器就可以通过 ip 访问了。

### 添加用户

**在 verdaccio 启动的情况下**，我们通过 npm 注册用户：

```bash
npm adduser --registry  http://10.26.5.252:4873
```

> npm adduser 是 npm login 的别名

之后会让你输入用户名密码以及邮箱。完成注册：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_2.png?raw=true)

添加用户后，会在 verdaccio 文件夹内生成一个 `htpasswd` 文件，其中包含了所有注册的用户:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_7.png?raw=true)

添加用户之后，默认就会在 `.npmrc` 中添加一个 token，如果后面要 publish 就得靠它：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_5.png?raw=true)

> 上面 registry= 表示的是下载源；
>
> 两个注释的带有 `authToken` 的表示的是注册用户的token。

> npm adducer 是用来注册的，如果已经注册了用户无法再次注册。可以直接在网页端登录生成 token，然后手动设置到 .npmrc 中。

## 切换下载源

一般情况下我们切换npm下载源的方式如下：

```shell
npm set registry http://localhost：4873 
```

我们可以通过 nrm 更快的操作。nrm是 npm registry 管理工具, 能够查看和切换当前使用的registry。不安装也可以，安装会更高效。

### 安装 nrm

```bash
npm install -g nrm
```

### 添加私服地址到nrm

我们需要给私服添加别名 `mynpm` 以供切换：

```shell
nrm add mynpm http://localhost:4873
```

通过 `nrm ls` 可以查看我们可以使用的镜像源，可以看到 `mynpm` 已经添加进去了：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_3.png?raw=true)

### 切换私服地址

如下命令切换私服：

```shell
nrm use mynpm
```

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_4.png?raw=true)

切换的其实是 `.npmrc` 中的配置:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_5.png?raw=true)



## 发布包

### 初始化一个测试包

创建一个文件夹，并且新建一个 `index.js` 文件。使用 `npm init` 方法可以帮助我们快速创建一个 `package.json`

### 发布包

使用 `npm publish` 发布包:

```shell
npm publish           #已经切换到我们私服地址的情况下
npm publish --registry http://localhost:4873   #未切换到我们的私服时，直接加后缀可以发布到私服上。
```

登录到之前的网页可以看到发布成功的包，也可以在 `.config/storage` 找到相应文件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/verdaccio_6.png?raw=true)

### 下载

只要切换了下载源，下载就和普通的 npm 一样,使用 `npm install xxx`

## 其他

### 修改配置

```shell
listen: 0.0.0.0:4873	#监听所有地址
storage: ./storage		#npm包保存位置
plugins: ./plugins		#插件保存位置

# 网页端的样式
web:
  # WebUI is enabled as default, if you want disable it, just uncomment this line
  #enable: false
  title: Verdaccio

auth:
  htpasswd:
  	#用户列表保存的位置
    file: ./htpasswd
    #可注册的用户数
    #max_users: 1000

#配置上游的npm服务器，主要用于请求的仓库不存在时到上游服务器去拉取。
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
  taobao:
    url: https://registry.npm.taobao.org/

packages:
  '@*/*':
    #@/ 表示某下属的某项目
    #所有人都可以拉取，只有授权的用户可以publish，上游服务器为uplinks中的淘宝
    access: $all
    publish: $authenticated
    proxy: taobao

  '**':
	#匹配项目名称
    access: $all
    publish: $authenticated
    proxy: taobao

# To use `npm audit` uncomment the following section
middlewares:
  audit:
    enabled: true

# log settings
logs:
  - {type: stdout, format: pretty, level: http}
  #- {type: file, path: verdaccio.log, level: info}
```

### 永久运行

可以通过 [forever](https://github.com/nodejitsu/forever) 这个库将 verdaccio 永久运行。

```shell
#全局运行
sudo npm install -g forever
#使用forever启动verdaccio
forever start `which verdaccio`
```

不过使用的时候最好还是看看官方文档

## npm 后续使用

### 创建与运行 npm script

通过 `npm init` 命令创建 package.json。

在 package.json 的 scripts 字段中新增命令，如 eslint：

```json
{
  "scripts": {
    "eslint": "eslint *.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
}
```

对于一些内置的 npm 命令，直接执行 `npm+脚本名`，如上面的 `npm test` 或者 `npm start`。自己定义的脚本需要使用 `npm run +脚本名`，如 `npm run eslint`。

### npm script 钩子

npm script 在执行之前会检查是否有`pre` 和 `post` 的钩子脚本。即如果运行 `npm run test` 脚本。会先检查是否有 `pretest` 脚本，如果有则执行，然后执行 `test` 最后执行 `posttest`:

```json
{
  "scripts": {
    "pretest": "npm run lint",
    "test": "mocha tests/",
    "posttest": "echo \"complete\""
  },
}
```

### git hook 中执行 npm script

我们使用 npm 库 `husky`。使用 `npm install husky --save-dev` 保存为 dev-denpendancy 中。然后我们的代码仓库的 `.git/hooks` 目录，会发现里面的钩子都被 husky 替换掉了:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/husky_1.png?raw=true)  

> --save-dev 表示只保存为开发环境下使用的模块。在生产和发布的过程中不会将这部分模块打入包中。

接下来需要在 scripts 对象中增加 husky 能识别的 Git Hooks 脚本 `precommit` 和 `prepush`：

```json
"scripts": {
    "precommit": "npm run lint",
    "prepush": "npm run test",
}
```

### @scope

有时候，我们看别人的 package.json 的时候，会看到 dependency 中有类似这样的东西：

```json
{
  "dependencies": {
    "@somescope/someprojectname": "^1.0.0",
  },
}
```

所有的私有模块都是 scoped package 的，所以上面表示的是一个私有仓库。只有当你获得相应权限的时候才可以拉取使用。





