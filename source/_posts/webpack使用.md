title: webpack 简单使用
date: 2018/3/29 14:07:12  
categories: JavaScript
tags: 

	- 学习笔记
	

---

努力成为一名webpack配置工程师~~~

<!--more-->

### 安装

使用 npm 安装，一般安装为 devdependency：

```bash
npm install webpack --save-dev
```

然后再 package.json 中添加一个 script ：

```json
"scripts": {
	"build": "webpack --mode production"
},
"devDependencies": {
	"webpack": "^4.1.1",
    "webpack-cli": "^2.0.12",
 }
```

执行了 `npm run bulid` 的时候，就会在目录下寻找 `webpack.config.js`  配置文件并执行。

### 入口

入口表示打包的依赖的开始。使用 `entry` 字段表示：

```javascript
// 或者配置多个入口
module.exports = {
  entry: {
    foo: './src/page-foo.js',
    bar: './src/page-bar.js', 
    // ...
  }
}

// 使用数组来对多个文件进行打包
module.exports = {
  entry: {
    main: [
      './src/foo.js',
      './src/bar.js'
    ]
  }
}
```

### loader

loader 是一种转换器，将某种类型的文件格式转为 webpack 可以支持的打包的模块。比如可以通过 vue-loader 解析 .vue 文件，将其转换为 js 代码。

使用 `module.rules` 表示：

```javascript
module: {
  // ...
  rules: [
    {
      test: /\.jsx?/, // 匹配文件路径的正则表达式，通常我们都是匹配文件类型后缀
      include: [
        path.resolve(__dirname, 'src') // 指定哪些路径下的文件需要经过 loader 处理
      ],
      use: 'babel-loader', // 指定使用的 loader
    },
  ],
}
```

###  plugin

模块代码转换的工作由 loader 来处理，除此之外的其他任何工作都可以交由 plugin 来完成。

比如使用 **uglifyjs-webpack-plugin** 压缩 JS 代码

```bash
const UglifyPlugin = require('uglifyjs-webpack-plugin')

module.exports = {
  plugins: [
    new UglifyPlugin()
  ],
}
```

**DefinePlugin** 是 webpack 内置的插件，可以使用 `webpack.DefinePlugin` 直接获取。这个插件用于创建一些在编译时可以配置的全局常量：

```javascript
module.exports = {
  // ...
  plugins: [
    new webpack.DefinePlugin({
      PRODUCTION: JSON.stringify(true), // const PRODUCTION = true
      VERSION: JSON.stringify('5fa3b9'), // const VERSION = '5fa3b9'
      BROWSER_SUPPORTS_HTML5: true, // const BROWSER_SUPPORTS_HTML5 = 'true'
      TWO: '1+1', // const TWO = 1 + 1,
      CONSTANTS: {
        APP_VERSION: JSON.stringify('1.1.2') // const CONSTANTS = { APP_VERSION: '1.1.2' }
      }
    }),
  ],
}
```

有了上面的配置，就可以在应用代码文件中，访问配置好的变量了，如:

```javascript
console.log("Running App version " + VERSION);
```

**copy-webpack-plugin** 用来把资源文件移动到指定目录下：

```javascript
const CopyWebpackPlugin = require('copy-webpack-plugin')

module.exports = {
  // ...
  plugins: [
    new CopyWebpackPlugin([
      { from: 'src/file.txt', to: 'build/file.txt', }, // 顾名思义，from 配置来源，to 配置目标路径
      { from: 'src/*.ico', to: 'build/*.ico' }, // 配置项可以使用 glob
      // 可以配置很多项复制规则
    ]),
  ],
}
```

### 输出

输出即指 webpack 最终构建出来的静态文件，使用 `output` 字段：

```bash
module.exports = {
  // ...
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
}

// 或者多个入口生成不同文件
module.exports = {
  entry: {
    foo: './src/foo.js',
    bar: './src/bar.js',
  },
  output: {
    filename: '[name].js',
    path: __dirname + '/dist',
  },
}
```

### 解析

`resolve` 表示如何解析。

使用 `resolve.alias` 来创建别名：

```javascript
module.exports = {
  //...
  resolve: {
    alias: {
      Utilities: path.resolve(__dirname, 'src/utilities/'),
      Templates: path.resolve(__dirname, 'src/templates/')
    }
  }
};
```

如果原来使用相对路径：

```javascript
import Utility from '../../utilities/utility';
```

现在就可以使用绝对路径了：

```javascript
import Utility from 'Utilities/utility';
```

使用 `resolve.extensions` 解析相应类型：

```javascript
module.exports = {
  //...
  resolve: {
    extensions: ['.js', '.vue', '.json'],
  }
};
```

使用 `resolve.modules` 配置模块的搜索路径：

```javascript
resolve: {
  modules: ['node_modules'],
},
```



