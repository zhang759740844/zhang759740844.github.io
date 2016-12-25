
title: HTTP详解
date: 2016/12/24 16:07:12  
categories: 计算机
tags:
	- 学习笔记

---

网络协议这块一直没有搞透彻，趁着苹果强制 ATS（App Transport Security）未果，学习一下这方面的知识。

<!--more-->

## HTTP的构成
### HTTP Request
http 请求主要分为三部分，如下图所示。`request line`，`header` 和 `body`，中间的CRLF为换行符。

![http请求组成](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/http请求.png?raw=true)

下面将以一个实际的 http 请求来详细看看其内部构造。假设我们的请求URL为：
`http://www.baidu.com/res/static/thirdparty/connect.jpg?t=1480992153433`

#### Request Line

请求行的结构为：

```
Request-Line   = Method SP Request-URI SP HTTP-Version CRLF
```

- **Method**:请求类型，例如 `post/get`。
- **SP**:分隔符，一般就是空格。
- **Request-URI**:对应上述请求为：`/res/static/thirdparty/connect.jpg?t=1480992153.564331`。注意，Request-URI 也可以是带 host 的完整 uri。不带的话，就要在 `Header` 里添加 host。
- **HTTP-Version**:代表我们当前使用的版本，例如 `HTTP/1.1`
- **CRLF**:`CR` 对应回车键，`LF`对应换行键，合起来就是我们平常所说的 `\r\n`。

所以上述请求的 Request-Line 的文本展示：
>GET /res/static/thirdparty/connect.jpg?t=1480992153.564331 HTTP/1.1CRLF

#### Header
header 其本质上是一些文本键值对，一个典型的例子如下图所示：

![header](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/http_header.png?raw=true)

结构为：

```
Key:空格ValueCRLF
```

上面讲述 Request-URI 的时候，缺失的 `Host` 就以键值对的形式存在于 header 中，比如， `Host： pan.baidu.com`。

将若干个上述格式的键值对组合起来，就成了我们 HTTP 请求的完整 header。最后一个键值对之后再跟一个 `CRLF`，就表示我们的 header 结束了。

#### Body
body 里面包含请求的实际数据。

对于 `Method=GET` 的请求来说，body 体是为空的，Header 最后的两个 `CRLF` 就标识着请求的结尾。我们一般调用请求的业务参数是通过 Request Line 当中的 Request-URI 来传递的，比如上述请求中的 `?t=1480992153.564331`，也就是 URI 的 query string 部分。这部分同样是以键值对的形式存在，不过是位于 Request Line 当中。

对于 `Method=POST` 的请求来说，实际的业务数据都存放于 body 当中。**POST 请求可以根据 Header 中的 `Content-Type` 值，以不同的形式将数据保存在 body 体中。获取到请求后，可以仍然通过 Header 中的 `Content-Length` 值，读取固定长度的body体，直接传递给应用层** 

### HTTP Response
response 的结构和 request 大致相似，不过是将 Request Line 换成了 Status Line 。

![http_response](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/http_response.png?raw=true)

Status Line 的结构如下：
```
Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

### Content-Type
Content-Type 用于指定内容类型，一般是指网页中存在的 Content-Type，Content-Type 属性指定**请求和响应**的 HTTP 内容类型。如果未指定  ContentType，默认为 `text/html`。

一些常见的 Content-Type:
- text/html
- text/plain
- text/css
- text/javascript
- application/x-www-form-urlencoded
- multipart/form-data
- application/json
- application/xml

前面几个都很好理解，都是 html，css，javascript 的文件类型，**后面四个是POST 的发包方式**。

那我们什么时候用 `application/x-www-form-urlencoded`，什么时候用 `multipart/form-data` 呢？大文件如文件图片用后者，小文件如键值对用前者。

## HTTPS

## 抓包工具 mitmproxy 使用
mitmproxy 是一款命令行抓包工具，它除了可以抓包查看 http/https 请求，还可以拦截并修改 request 或者 response。

### 安装
首先需要安装 python 的包管理工具 pip。 OSX El Capitan 及以上的系统版本在安装时会出现 six 模块依赖错误，所以需要执行以下命令进行安装：

```
sudo pip install mitmproxy --ignore-installed six
```

### 配置
手机和电脑在同一局域网下，设置手机的 http 代理。服务器为电脑当前的 ip，如 192.168.3.10；端口可任意设置，一般设置为8080。网络环境配置好后，在终端输入 `mitmproxy -p 8080`，进入抓包模式。

如果想要抓取 https 的包，需要进一步配置。用 iPhone 打开 Safari 浏览器并输入 mitm.it，这时你会看到如下页面，选择对应平台并安装证书，安装完成后就可以抓 https 的包了。注意，用浏览器打开时需要已经在抓包模式，否则是无法看到上述页面的。

![下载证书](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/httpproxy_certificate.png?raw=true)

### 使用
随便打开一个 app，就可以看到如下的一个请求列表。
![http请求列表](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/http_request.png?raw=true)

常用快捷键：
- enter: 查看详细请求
- Tab: 切换顶部导航栏
- z: 清空列表
- f: 过滤请求。可以参照帮助中的 Filter expression 对过滤关键字进行编辑。删除过滤就是将过滤关键字清空。
- d: 删除请求
- C: 复制请求。直接选中复制可能会复制进空格非常麻烦，可以通过 C 来选择复制的内容。

关于如何拦截和修改 request 和 response 可以参见[抓包教程](http://ios.jobbole.com/90841/)