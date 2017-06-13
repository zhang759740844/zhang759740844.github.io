
title: HTML+CSS 使用方式
date: 2017/5/31 11:07:12
categories: JavaScript
tags:
	- 学习笔记
	- HTML

---

从最简单的看起，这篇文章知识为了备忘

<!--more-->

## HTML

### 基本的四个 HTML 标签

HTML **标题**（Heading）是通过 `<h1> - <h6>` 等标签进行定义的。

```html
<h1>This is a heading</h1>
```

HTML **段落**是通过 `<p> `标签进行定义的。

```html
<p>This is a paragraph.</p>
```

HTML **链接**是通过 `<a>` 标签进行定义的。在 href 属性中指定链接的地址。

```html
<a href="http://www.w3school.com.cn">This is a link</a>
```

HTML **图像**是通过 `<img>` 标签进行定义的。

```html
<img src="w3school.jpg" width="104" height="142" />
```

### HTML 元素

HTML 元素指的是从开始标签（start tag）到结束标签（end tag）的所有代码。

`<p>`元素定义了 HTML 文档中的一个段落。

```html
<p>This is my first paragraph.</p>
```

`<body>` 元素定义了 HTML 文档的主体

```html
<body>
<p>This is my first paragraph.</p>
</body>
```

`<html>` 元素定义了整个 HTML 文档。

```html
<html>

<body>
<p>This is my first paragraph.</p>
</body>

</html>
```

没有内容的 HTML 元素被称为空元素。空元素是在开始标签中关闭的。所有元素都必须被关闭，在开始标签中添加斜杠，比如` <br />`，是关闭空元素的正确方法。

### HTML 属性

HTML 标签可以拥有属性，总是以名称/值对的形式出现，比如：**name="value"**。属性总是在 HTML 元素的开始标签中规定，例如：

```html
<table border="1">
```

属性值应该始终被包括在引号内。双引号是最常用的，在某些个别的情况下，比如属性值本身就含有双引号，那么您必须使用单引号，例如：

```html
name='Bill "HelloWorld" Gates'
```

常用属性：

```
class:	规定元素的类名
id:		规定元素的唯一 id
style:	规定元素的行内样式
title:	规定元素的额外信息
```

### HTML 标题

标题（Heading）是通过 `<h1> - <h6>`等标签进行定义的。

```html
<h1>This is a heading</h1>
<h2>This is a heading</h2>
<h3>This is a heading</h3>
```

`<hr /> `标签在 HTML 页面中创建水平线。hr 元素可用于分隔内容。

```html
<p>This is a paragraph</p>
<hr />
<p>This is a paragraph</p>
<hr />
<p>This is a paragraph</p>
```

### HTML 段落

段落是通过 `<p>` 标签定义的。

```html
<p>This is a paragraph</p>
<p>This is another paragraph</p>
```

对于 HTML，您无法通过在 HTML 代码中添加额外的空格或换行来改变输出的效果。如果希望在不产生一个新段落的情况下进行换行（新行），请使用  `<br />` 标签：

```html
<p>This is<br />a para<br />graph with line breaks</p>
```

### HTML 样式

style 属性用于改变 HTML 元素的样式。具体在 CSS 系列中能够学到

```html
<html>

<body>
<h1 style="font-family:verdana">A heading</h1>
<p style="font-family:arial;color:red;font-size:20px;">A paragraph.</p>
</body>

</html>
```

### HTML 文本格式化

```
<b>		定义粗体文本。
<big>		定义大号字。
<em>		定义着重文字。
<i>		定义斜体字。
<small>	定义小号字。
<strong>	定义加重语气。
<sub>		定义下标字。
<sup>		定义上标字。
<ins>		定义插入字。
<del>		定义删除字。
```

### HTML 引用

`<q> `元素定义短的引用。浏览器通常会为 `<q>` 元素包围引号。

```html
<p>WWF 的目标是：<q>构建人与自然和谐共存的世界。</q></p>
```

`<blockquote>` 元素定义被引用的节。浏览器通常会对 `<blockquote> `元素进行缩进处理。

```html
<p>以下内容引用自 WWF 的网站：</p>
<blockquote>
五十年来，WWF 一直致力于保护自然界的未来。
世界领先的环保组织，WWF 工作于 100 个国家，
并得到美国一百二十万会员及全球近五百万会员的支持。
</blockquote>
```

### HTML 计算机代码元素

 `<code>` 元素定义编程代码,不保留多余的空格和折行,示例：

```html
<p>Coding Example:</p>
<code>
var person = { firstName:"Bill", lastName:"Gates", age:50, eyeColor:"blue" }
</code>
```

如需保留空格折行，需要使用 `<pre>` :

```html
<p>Coding Example:</p>

<code>
<pre>
var person = {
    firstName:"Bill",
    lastName:"Gates",
    age:50,
    eyeColor:"blue"
}
</pre>
</code>
```

### HTML 注释

注释是这样写的，开始括号之后（左边的括号）需要紧跟一个叹号，结束括号之前（右边的括号）不需要。：

```html
<! This is a comment >
```

### HTML CSS

当样式需要被应用到很多页面的时候，外部样式表将是理想的选择。使用**外部样式表**，你就可以通过更改一个文件来改变整个站点的外观。

```html
<head>
<link rel="stylesheet" type="text/css" href="/html/csstest1.css">
</head>
```

> 这里的路径是相当于网站根路径的  

当单个文件需要特别样式时，就可以使用**内部样式表**。你可以在 head 部分通过 `<style>` 标签定义内部样式表。

```html
<head>
<style type="text/css">
body {background-color: red}
p {margin-left: 20px}
</style>
</head>
```

当特殊的样式需要应用到个别元素时，就可以使用**内联样式**:

```html
<p style="color: red; margin-left: 20px">
This is a paragraph
</p>
```

### HTML 链接

开始标签和结束标签之间的文字被作为超级链接来显示。”链接文本" 不必一定是文本。图片或其他 HTML 元素都可以成为链接。

```html
<a href="http://www.w3school.com.cn/">Visit W3School</a>
```

使用 Target 属性，你可以定义被链接的文档在何处显示。

```html
// 这段代码会在新窗口显示
<a href="http://www.w3school.com.cn/" target="_blank">Visit W3School!</a>
```

name 属性规定锚（anchor）的名称。可以使用 name 属性创建 HTML 页面中的书签。当使用命名锚（named anchors）时，我们可以创建直接跳至该命名锚（比如页面中某个小节）的链接，这样使用者就无需不停地滚动页面来寻找他们需要的信息了。

首先，我们在 HTML 文档中对锚进行命名（创建一个书签）：

```html
<a name="tips">基本的注意事项 - 有用的提示</a>
```

然后，我们在同一个文档中创建指向该锚的链接：

```html
<a href="#tips">有用的提示</a>
```

您也可以在其他页面中创建指向该锚的链接：

```html
<a href="http://www.w3school.com.cn/html/html_links.asp#tips">有用的提示</a>
```

在上面的代码中，我们将 **#** 符号和锚名称添加到 URL 的末端，就可以直接链接到 tips 这个命名锚了。

### HTML 图像

图像由 `<img>` 标签定义。要在页面上显示图像，你需要使用**源属性**

```html
<img src="http://www.w3school.com.cn/images/boat.gif" />
```

alt 属性用来为图像定义一串预备的可替换的文本。在浏览器无法载入图像时，替换文本属性告诉读者她们失去的信息。

```html
<img src="boat.gif" alt="Big Boat">
```

### HTML 表格

表格由 `<table>` 标签来定义。每个表格均有若干行（由 `<tr>` 标签定义），每行被分割为若干单元格（由 `<td>` 标签定义）

使用边框属性来显示一个带有边框的表格

表格的表头使用 `<th>` 标签进行定义。大多数浏览器会把表头显示为粗体居中的文本。

使用`<caption>` 属性描述表格标题，标题的显示位置：表格上方。

```html
<table border="1">
<caption>标题文本</caption>
<tr>
<th>Heading</th>
<th>Another Heading</th>
<tr>
<td>row 1, cell 1</td>
<td>row 1, cell 2</td>
</tr>
<tr>
<td>row 2, cell 1</td>
<td>row 2, cell 2</td>
</tr>
</table>
```

### HTML 列表

无序列表是一个项目的列表，此列项目使用粗体圆点（典型的小黑圆圈）进行标记。无序列表始于 `<ul>` 标签。每个列表项始于 `<li>`。

```html
<ul>
<li>Coffee</li>
<li>Milk</li>
</ul>
```

有序列表也是一列项目，列表项目使用数字进行标记。有序列表始于 `<ol>` 标签。每个列表项始于 `<li>` 标签。

```html
<ol>
<li>Coffee</li>
<li>Milk</li>
</ol>
```

### HTML `<div>` 和 `<span>`

大多数 HTML 元素被定义为块级元素或内联元素。块级元素在浏览器显示时，通常会以新行来开始和结束，如：

```html
<h1>, <p>, <ul>, <table>
```

内联元素在显示时通常不会以新行开始:

```html
<b>, <td>, <a>, <img>
```

HTML `<div>` 元素是块级元素，它是可用于组合其他 HTML 元素的容器。与 CSS 一同使用，`<div>` 元素可用于对大的内容块设置样式属性。`<div>` 元素的另一个常见的用途是文档布局。

HTML `<span>` 元素是内联元素，可用作文本的容器。当与 CSS 一同使用时，`<span>` 元素可用于为部分文本设置样式属性。

### HTML 类

对 HTML 进行分类（设置类），使我们能够为元素的类定义 CSS 样式。

```html
<!DOCTYPE html>
<html>
<head>
<style>
.cities {
    background-color:black;
    color:white;
    margin:20px;
    padding:20px;
} 
</style>
</head>

<body>
<div class="cities">
<h2>London</h2>
<p>
London is the capital city of England. 
It is the most populous city in the United Kingdom, 
with a metropolitan area of over 13 million inhabitants.
</p>
</div> 
</body>
</html>
```

这里的 `.cities` 表示任意名为 `cities` 的类，可以在前面加上前缀 `div` 表示只有 `<div>` 才能使用这个类.

还可以二级元素，如 `<table>` 中的 `<th>` 等：

```html
//html
<body>

<table class="lamp">
<caption>标题文本</caption>
<tr>
  <th>
    <img src="/images/lamp.jpg" alt="Note" style="height:32px;width:32px">
  </th>
  <td>
    The table element was not designed to be a layout tool.
  </td>
</tr>
</table>

</body>

//css
<style>
table.lamp {
    width:100%;
    border:1px solid #d4d4d4;
}
table.lamp th, td {
    padding:10px;
}
table.lamp td {
    width:40px;
}
</style>
```

### HTML 布局

`<div>` 元素常用作布局工具：

```html
// html
<body>

<div id="header">
<h1>City Gallery</h1>
</div>

<div id="nav">
London<br>
Paris<br>
Tokyo<br>
</div>

<div id="section">
<h1>London</h1>
<p>
London is the capital city of England. It is the most populous city in the United Kingdom,
with a metropolitan area of over 13 million inhabitants.
</p>
<p>
Standing on the River Thames, London has been a major settlement for two millennia,
its history going back to its founding by the Romans, who named it Londinium.
</p>
</div>

<div id="footer">
Copyright W3School.com.cn
</div>

</body>

//css
<style>
#header {
    background-color:black;
    color:white;
    text-align:center;
    padding:5px;
}
#nav {
    line-height:30px;
    background-color:#eeeeee;
    height:300px;
    width:100px;
    float:left;
    padding:5px; 
}
#section {
    width:350px;
    float:left;
    padding:10px; 
}
#footer {
    background-color:black;
    color:white;
    clear:both;
    text-align:center;
    padding:5px; 
}
</style>
```

使用了 `id` 表示元素，对应需要加上 `#`

### HTML 脚本

`<script>` 标签用于定义客户端脚本，script 元素既可包含脚本语句，也可通过 src 属性指向外部脚本文件。

### HTML 头部元素

`<head>` 元素是所有头部元素的容器。`<head>` 内的元素可包含脚本，指示浏览器在何处可以找到样式表，提供元信息。

`<title>` 标签定义文档的标题。能够定义浏览器工具栏中的标题;提供页面被添加到收藏夹时显示的标题;显示在搜索引擎结果中的页面标题.

```html
<!DOCTYPE html>
<html>
<head>
<title>Title of the document</title>
</head>

<body>
The content of the document......
</body>

</html>
```

`<base>` 标签为页面上的所有链接规定**默认地址**或**默认目标（target）**：

```html
<head>
<base href="http://www.w3school.com.cn/images/" />
<base target="_blank" />
</head>
```

`<link>` 标签定义文档与外部资源之间的关系。最常用于连接样式表：

```html
<head>
<link rel="stylesheet" type="text/css" href="mystyle.css" />
</head>
```

`<style>` 标签用于为 HTML 文档定义样式信息:

```html
<head>
<style type="text/css">
body {background-color:yellow}
p {color:blue}
</style>
</head>
```

`<meta>` 标签提供关于 HTML 文档的元数据。元数据不会显示在页面上，但是对于机器是可读的。典型的情况是，meta 元素被用于规定页面的描述、关键词、文档的作者、最后修改时间以及其他元数据。`<meta>` 标签始终位于 head 元素中。

`<script>` 标签用于定义客户端脚本，比如 JavaScript。

### HTML 字符实体

在 HTML 中，某些字符是预留的.

比如不能直接使用 `<` 因为会被认为是标签，必须写成 `&lt;`。再比如不能直接使用空格，需要空格则必须写成`&nbsp `，具体可以查表。

### HTML 统一资源定位器

统一资源定位器（URL）用于定位万维网上的文档，遵守以下的语法规则：

```html
scheme://host.domain:port/path/filename

scheme - 定义因特网服务的类型。最常见的类型是 http
host - 定义域主机（http 的默认主机是 www）
domain - 定义因特网域名，比如 w3school.com.cn
:port - 定义主机上的端口号（http 的默认端口号是 80）
path - 定义服务器上的路径（如果省略，则文档必须位于网站的根目录中）。
filename - 定义文档/资源的名称
```

### HTML URL字符编码

URL 只能使用 ASCII 字符集来通过因特网进行发送。由于 URL 常常会包含 ASCII 集合之外的字符，URL 必须转换为有效的 ASCII 格式。URL 编码使用 "%" 其后跟随两位的十六进制数来替换非 ASCII 字符。

### HTML <!DOCTYPE>

`<!DOCTYPE>` 不是 HTML 标签。它为浏览器提供一项信息（声明），即 HTML 是用什么版本编写的。最常用的带有 HTML5 DOCTYPE 的 HTML 文档：

```html
<!DOCTYPE html>
```

### HTML 表单

HTML 表单用于搜集不同类型的用户输入。`<form>` 元素定义 HTML 表单。表单元素指的是不同类型的 input 元素、复选框、单选按钮、提交按钮等等。

`<input>` 元素是最重要的表单元素。`<input>` 元素有很多形态，根据不同的 type 属性。

`<input type="text">` 定义用于文本输入的单行输入字段：

```html
<form>
First name:<br>
<input type="text" name="firstname">
<br>
Last name:<br>
<input type="text" name="lastname">
</form> 
```

> 上面代码说明不要任何标签也是可以显示出字符的  

`<input type="radio">` 定义单选按钮:

```html
<form>
<input type="radio" name="sex" value="male" checked>Male
<br>
<input type="radio" name="sex" value="female">Female
</form> 
```

`<input type="submit”>`定义用于向表单处理程序提交表单的按钮:

```html
<form action="action_page.php" target="_blank">
First name:<br>
<input type="text" name="firstname" value="Mickey">
<br>
Last name:<br>
<input type="text" name="lastname" value="Mouse">
<br><br>
<input type="submit" value="Submit">
</form> 	
```

**action** 属性定义在提交表单时执行的动作。在上面的例子中，指定了某个服务器脚本来处理被提交表单：

```html
<form action="action_page.php">
```

**target**属性表示规定 action 属性中地址的目标（默认：_self）。本例中表示打开新的网页：

```html
<form action="action_page.php" target="_blank">
```

**method** 属性规定在提交表单时所用的 HTTP 方法（GET 或 POST），默认 GET：

```html
<form action="action_page.php" method="GET">
```

如果要正确地被提交，每个输入字段必须设置一个 **name** 属性，**相当于键**。上面的例子只会提交 "Last name" 输入字段，其 http 请求是：

```html
baseUrl/action_page.php?lastname=Mouse
```

每个输入字段的**value**属性是被提交的值，上面的 value 是默认的填充。

### HTML 表单元素

`<select>` 元素定义下拉列表，`<option>` 元素定义待选择的选项。您能够通过添加 **selected** 属性来定义预定义选项。

```html
<select name="cars">
<option value="volvo">Volvo</option>
<option value="saab">Saab</option>
<option value="fiat" selected>Fiat</option>
<option value="audi">Audi</option>
```

`<textarea>` 元素定义多行输入字段（文本域）:

```html
<textarea name="message" rows="10" cols="30">
The cat was playing in the garden.
</textarea>
```

### HTML Input 属性

**value** 属性规定输入字段的初始值：

```html
<form action="">
 First name:<br>
<input type="text" name="firstname" value="John">
<br>
 Last name:<br>
<input type="text" name="lastname">
</form> 
```

**readonly** 属性规定输入字段为只读:

```html
<form action="">
 First name:<br>
<input type="text" name="firstname" value="John" readonly>
<br>
 Last name:<br>
<input type="text" name="lastname">
</form> 
```

**disabled** 属性规定输入字段是禁用的。被禁用的元素是不可用和不可点击的。被禁用的元素不会被提交。

```html
<form action="">
 First name:<br>
<input type="text" name="firstname" value="John" disabled>
<br>
 Last name:<br>
<input type="text" name="lastname">
</form> 
```

**size** 属性规定输入字段的尺寸（以字符计）：

```html
<form action="">
 First name:<br>
<input type="text" name="firstname" value="John" size="40">
<br>
 Last name:<br>
<input type="text" name="lastname">
</form> 
```

**maxlength** 属性规定输入字段允许的最大长度，该属性不会提供任何反馈。如果需要提醒用户，则必须编写 JavaScript 代码：

```html
<form action="">
 First name:<br>
<input type="text" name="firstname" maxlength="10">
<br>
 Last name:<br>
<input type="text" name="lastname">
</form> 
```

### HTML label标签

label标签不会向用户呈现任何特殊效果，它的作用是为鼠标用户改进了可用性。如果你在 label 标签内点击文本，就会触发此控件。就是说，当用户单击选中该label标签时，浏览器就会自动将焦点转到和标签相关的表单控件上（就自动选中和该label标签相关连的表单控件上）。

语法：`<label for="控件id名称">`

```html
<form>
  <label for="male">男</label>
  <input type="radio" name="gender" id="male" />
  <br />
  <label for="female">女</label>
  <input type="radio" name="gender" id="female" />
  <label for="email">输入你的邮箱地址</label>
  <input type="email" id="email" placeholder="Enter email">
</form>
```

## CSS

### CSS 基础语法

CSS 的声明方式：

```css
selector {property: value; property: value; property: value}
```

如果是若干单词，需要给值加上引号，值之前最好加个冒号，一行最好只写一个属性：

```css
h1 {	color: red; 
		font-size: 14px;
		font-family: "sans serif";
}
```

### CSS 高级语法

可以对选择器进行分组，被分组的选择器就可以分享相同的声明：

```css
h1,h2,h3,h4,h5,h6 {
		color: green;
}
```

### CSS 派生选择器

通过依据元素在其位置的上下文关系来定义样式，你可以使标记更加简洁：

```css
li strong {
    font-style: italic;
    font-weight: normal;
}
```

这样只有 `li` 中的 `strong` 才是如上的定义：

```css
<p><strong>我是粗体字，不是斜体字，因为我不在列表当中，所以这个规则对我不起作用</strong></p>

<ol>
<li><strong>我是斜体字。这是因为 strong 元素位于 li 元素内。</strong></li>
<li>我是正常的字体。</li>
</ol>
```

### id 选择器

id 选择器可以为标有特定 id 的 HTML 元素指定特定的样式。id 选择器以 `#` 来定义:

```css
#red {color:red;}
#green {color:green;}
```

```html
<p id="red">这个段落是红色。</p>
<p id="green">这个段落是绿色。</p>
```

> id 属性只能在每个 HTML 文档中出现一次  

在现代布局中，id 选择器常常用于建立派生选择器:

```css
#sidebar p {
	font-style: italic;
	text-align: right;
	margin-top: 0.5em;
	}
```

与页面中的其他 p 元素明显不同的是，sidebar 内的 p 元素得到了特殊的处理

### CSS 类选择器

类选择器以一个点号显示：

```css
.center {text-align: center}
```

```html
<h1 class="center">
This heading will be center-aligned
</h1>

<p class="center">
This paragraph will also be center-aligned.
</p>
```

和 id 一样，class 也可被用作派生选择器，此例中类名为 fancy 的更大的元素内部的表格单元都会以灰色背景显示橙色文字：

```css
.fancy td {
	color: #f60;
	background: #666;
	}
```

元素也可以基于它们的类而被选择，任何其他被标注为 fancy 的元素不会受这条规则的影响：

```css
td.fancy {
	color: #f60;
	background: #666;
	}
```

```html
<td class="fancy">
```

### CSS 子选择器

子选择器，大于符号(>)，用于选择指定标签元素的**第一代**子元素

```css
.food>li{border:1px solid red;}
```

这个是类型下面的元素，派生选择器是元素下面的元素

### CSS 包含(后代)选择器

包含选择器，即加入空格,用于选择指定标签元素下的**后辈**元素：

```css
.first  span{color:red;}
```

**>**作用于元素的**第一代后代**，**空格**作用于元素的**所有后代**。

### CSS 通用选择器

它使用一个（*）号指定，它的作用是匹配html中所有标签元素：

```css
* {color:red;}
```

### CSS 属性选择器

为拥有指定属性的 HTML 元素设置样式，而不仅限于 class 和 id 属性。下面的例子为带有 title 属性的所有元素设置样式：

```css
[title] {
	color:red;
}
```

下面的例子为 `title="W3School"` 的所有元素设置样式：

```css
[title=W3School] {
	border:5px solid blue;
}
```

属性选择器在为不带有 class 或 id 的表单设置样式时特别有用：

```css
input[type="text"]
{
  width:150px;
  display:block;
  margin-bottom:10px;
  background-color:yellow;
  font-family: Verdana, Arial;
}

input[type="button"]
{
  width:120px;
  margin-left:35px;
  display:block;
  font-family: Verdana, Arial;
}
```

### CSS 继承

CSS的某些样式是具有继承性的，它允许样式不仅应用于某个特定html标签元素，而且应用于其后代。如某种颜色应用于p标签，这个颜色设置不仅应用p标签，还应用于p标签中的所有子元素文本，这里子元素为span标签：

```css
p{color:red;}
```

```html
<p>三年级时，我还是一个<span>胆小如鼠</span>的小女孩。</p>
```

但注意有一些css样式是不具有继承性的。如 `border:1px solid red;`

### CSS 创建

插入样式表的方法有三种：

**外部样式表**：当样式需要应用于很多页面时，外部样式表将是理想的选择。在使用外部样式表的情况下，你可以通过改变一个文件来改变整个站点的外观。每个页面使用 `<link>` 标签链接到样式表。`<link>` 标签在（文档的）头部：

```html
<head>
<link rel="stylesheet" type="text/css" href="mystyle.css" />
</head>
```

**内部样式表**:当单个文档需要特殊的样式时，就应该使用内部样式表。你可以使用 `<style>` 标签在文档头部定义内部样式表，就像这样:

```html
<head>
<style type="text/css">
  hr {color: sienna;}
  p {margin-left: 20px;}
  body {background-image: url("images/back40.gif");}
</style>
</head>
```

**内联样式**:由于要将表现和内容混杂在一起，内联样式会损失掉样式表的许多优势。请慎用这种方法，例如当样式仅需要在一个元素上应用一次时。要使用内联样式，你需要在相关的标签内使用样式（style）属性。Style 属性可以包含任何 CSS 属性：

```html
<p style="color: sienna; margin-left: 20px">
This is a paragraph
</p>
```

### CSS 注释

和 html 的注释不同，css 的使用 `/*…*/` 注释

```css
/* 注释 */
p{
   font-size:12px;
   color:red;
}
```

### 一些常用的 CSS 属性

```css
body{
	font-size:12px;	
	color:#666;
	font-weight:bold;
	font-style:italic;
	text-decoration:underline;
	text-indent:2em;
	line-height:1.5em;
	letter-spacing:50px;
	text-align:center;
}
```

### CSS 边框的一些属性

```css
div{
    border-width:2px;
    border-style:solid;
    border-color:red;
}
```

### CSS 内边距

padding 属性定义元素的内边距：

```css
h1 {padding: 10px;}
```

还可以按照上、右、下、左的顺序分别设置各边的内边距，各边均可以使用不同的单位或百分比值：

```css
h1 {
  padding-top: 10px;
  padding-right: 0.25em;
  padding-bottom: 2ex;
  padding-left: 20%;
  }
```

百分数值是相对于其父元素的 width 计算的，这一点与外边距一样。所以，如果父元素的 width 改变，它们也会改变。

### CSS 外边距

设置外边距的最简单的方法就是使用 margin 属性。

```css
h1 {margin : 0.25in;}
```

一个规则中可以使用多个这种单边属性:

```css
h2 {
  margin-top: 20px;
  margin-right: 30px;
  margin-bottom: 30px;
  margin-left: 20px;
}
```

### CSS 外边距合并

外边距合并指的是，当两个垂直外边距相遇时，它们将形成一个外边距。
合并后的外边距的高度等于两个发生合并的外边距的高度中的较大者。

外边距合并初看上去可能有点奇怪，但是实际上，它是有意义的。以由几个段落组成的典型文本页面为例。第一个段落上面的空间等于段落的上外边距。如果没有外边距合并，后续所有段落之间的外边距都将是相邻上外边距和下外边距的和。这意味着段落之间的空间是页面顶部的两倍。如果发生外边距合并，段落之间的上外边距和下外边距就合并在一起，这样各处的距离就一致了。

只有普通文档流中块框的垂直外边距才会发生外边距合并。行内框、浮动框或绝对定位之间的外边距不会合并。

### CSS 代码缩写

不论是 padding 还是 margin 都可以简写，顺序为 **上右下左**：

```css
p{
    padding:13px 13px 13px 13px;
    margin:10px 40px 10px 40px;
}
```

关于字体也有很多属性可以进行缩写：

```css
body{
    font-style:italic;
    font-variant:small-caps; 
    font-weight:bold; 
    font-size:12px; 
    line-height:1.5em; 
    font-family:"宋体",sans-serif;
}
=>
body{
    font:italic  small-caps  bold  12px/1.5em  "宋体",sans-serif;
}
```

在缩写时 font-size 与 line-height 中间要加入“/”斜扛。不然浏览器无法知道哪个是 size 哪个是 height



### CSS 相对定位

设置为相对定位的元素框会偏移某个距离。元素仍然保持其未定位前的形状，它原本所占的空间仍保留。

```css
#box_relative {
  position: relative;
  left: 30px;
  top: 20px;
}
```

> 在使用相对定位时，无论是否进行移动，元素仍然占据原来的空间。因此，移动元素会导致它覆盖其它框。  

### CSS 绝对定位

设置为绝对定位的元素框从文档流完全删除，并相对于其包含块定位，包含块可能是文档中的另一个元素或者是初始包含块。元素原先在正常文档流中所占的空间会关闭，就好像该元素原来不存在一样。元素定位后生成一个块级框，而不论原来它在正常流中生成何种类型的框。

```css
#box_relative {
  position: absolute;
  left: 30px;
  top: 20px;
}
```

小技巧：如果有一个元素要放在另一个元素中，比如图片上要防放置一行文字，那么可以将文字设置为绝对定位，放置在图片所在的块中，注意，参照定位的元素必须加入 `position:relative;`：

```css
#box1{
    width:200px;
    height:200px;
    position:relative;	// 这个一定要有  
}
#box2{
 	position:absolute;
	top:20px;
	left:30px;
}
```

```html
<div id="box1">
	<div id="box2">相对参照元素进行定位</div>
</div>
```

> 绝对定位的元素的位置相对于最近的已定位祖先元素，如果元素没有已定位的祖先元素，那么它的位置相对于最初的包含块。  

### CSS 浮动

浮动的框可以向左或向右移动，直到它的外边缘碰到**包含框或另一个浮动框的边框**为止。

```css
.news {
  background-color: gray;
  border: solid 1px black;
  }

.news img {
  float: left;
  }

.news p {
	clear:both;
  }
```

```html
<div class="news">
<img src="news-pic.jpg" />
<p>some text</p>
</div>
```

浮动框会影响后一个元素，使其与自身在同一行中。上面例子中，`<p>` 就会和两个浮动框在同一行中，如果不想让`<p>` 受到浮动的影响，另起一行，需要使用 `clear` 属性。

> 浮动的作用是实现文字环绕效果  
> 一般可以用其实现两栏  

### CSS 元素分类

```html
常用的块状元素有：
<div>、<p>、<h1>...<h6>、<ol>、<ul>、<dl>、<table>、<address>、<blockquote> 、<form>

常用的内联元素有：
<a>、<span>、<br>、<i>、<em>、<strong>、<label>、<q>、<var>、<cite>、<code>

常用的内联块状元素有：
<img>、<input>
```

设置 `display:block` 就是将元素显示为**块级元素**。如下代码就是将内联元素 `<a>` 转换为块状元素，从而使 `<a>`元素具有块状元素特点。

```css
a {display:block;}
```

**块级元素**的特点：每个块级元素都从新的一行开始，并且其后的元素也另起一行。元素的高度、宽度、行高以及顶和底边距都可设置。元素宽度在不设置的情况下，是它本身父容器的100%（和父元素的宽度一致），除非设定一个宽度。

当然块状元素也可以通过代码 `display:inline` 将元素设置为**内联元素**。如下代码就是将块状元素 `<div>` 转换为内联元素，从而使 `<div>` 元素具有内联元素特点。

```css
 div {display:inline;}
```

**内联元素**特点：和其他元素都在一行上；元素的高度、宽度及顶部和底部边距不可设置；元素的宽度就是它包含的文字或图片的宽度，不可改变。

**内联块状元素**就是同时具备内联元素、块状元素的特点，代码 `display:inline-block` 就是将元素设置为内联块状元素。`<img>`、`<input>` 标签就是这种内联块状标签。

**内联块状元素**特点：和其他元素都在一行上；元素的高度、宽度、行高以及顶和底边距都可设置。

### CSS 技巧——水平居中

如果被设置元素为文本、图片等**行内元素**时，水平居中是通过给**父元素**设置 `text-align:center` 来实现的。

```css
<style>
  .txtCenter{
    text-align:center;
  }
</style>
```

```html
<body>
  <div class="txtCenter">我想要在父容器中水平居中显示。</div>
</body>
```

当被设置元素为 块状元素 时用 `text-align：center ` 就不起作用了，这时也分两种情况：**定宽块状元素**和**不定宽块状元素**。

满足**定宽**和**块状**两个条件的元素是可以通过设置“左右margin”值为“auto”来实现居中的。我们来看个例子就是设置 div 这个块状元素水平居中：

```css
<style>
div{
    border:1px solid red;/*为了显示居中效果明显为 div 设置了边框*/
    
    width:200px;/*定宽*/
    margin:20px auto;/* margin-left 与 margin-right 设置为 auto */
}
</style>
```

```html
<body>
  <div>我是定宽块状元素，哈哈，我要水平居中显示。</div>
</body>
```

使**不定宽块状元素**水平居中，可以改变块级元素的 `display:inline;` 类型（设置为 行内元素 显示），然后使用 `text-align:center` 来实现居中效果。如下例子：

```css
<style>
.container{
    text-align:center;
}
/* margin:0;padding:0（消除文本与div边框之间的间隙）*/
.container ul{
    list-style:none;
    margin:0;
    padding:0;
    display:inline;
}
/* margin-right:8px（设置li文本之间的间隔）*/
.container li{
    margin-right:8px;
    display:inline;
}
</style>
```

```html
<body>
<div class="container">
    <ul>
        <li><a href="#">1</a></li>
        <li><a href="#">2</a></li>
        <li><a href="#">3</a></li>
    </ul>
</div>
</body>
```

### CSS 技巧——垂直居中

**父元素高度确定的单行文本**的竖直居中的方法是通过设置父元素的 height 和 line-height 高度一致来实现的。(**height**: 该元素的高度，**line-height**: 顾名思义，行高（行间距），指在文本中，行与行之间的 **基线间的距离** )。line-height 与 font-size 的计算值之差，在 CSS 中成为“**行间距**”。分为两半，分别加到一个文本行内容的顶部和底部。

```css
<style>
.container{
    height:100px;
    line-height:100px;
    background:#999;
}
</style>
```

```html
<div class="container">
    hi,imooc!
</div>
```















