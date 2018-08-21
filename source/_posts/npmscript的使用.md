title: npm script 的使用
date: 2018/8/20 14:07:12  
categories: Javascript
tags: 
	- npm
	
---

在掘金小册上学习了一番 npm script 的使用，记录一下。

<!--more-->



### 创建与运行 npm script

在 package.json 的 scripts 字段中新增命令，如 eslint：

```json
{
  "scripts": {
    "eslint": "eslint *.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
}
```

