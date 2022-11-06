---
title: 使用 Webpack5 从零架一个 React+TS 项目（一）
date: 2021-11-27
tags:
  - 工程化
  - Webpack
---

近期要做一个 react+ts 的小项目，之前写 react 都是直接拿 `creat-react-app` 脚手架一键配置的，趁这次机会学习一下 webpack 从零配置项目，顺便记录一下这个过程加深印象。

# 最基本的配置

架项目的第一步，创建新文件夹，然后 `npm init`，其中选择我们的入口文件为 src 文件夹下的 `index.ts`。

```纯文本
> mkdir nano
> cd nano
> npm init
{
  "name": "nano",
  "version": "1.0.0",
  "description": "",
  "main": "src/index.ts",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

然后简单创建一下入口文件 `src/index.ts`。

```纯文本
nano
├── package-lock.json
├── package.json
├── src
│   └── index.ts
├── tsconfig.json
└── webpack.config.js
```

既然要使用 webpack5，那我们接下来就装一下 webpack 和 cli。

```纯文本
> npm i webpack webpack-cli -D
```

新建一个 `webpack.config.js`，配置 webpack。下为 `webpack.config.js`。

```javascript
// webpack.config.js
const path = require('path'); // 导入path模块

module.exports = {
  // 入口文件
  entry: './src/index.ts',
  // 出口文件
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
};
```

然后我们可以在 `package.json` 里面编写我们的编译脚本了。

```json
// package.json
{
  "name": "nano",
  "version": "1.0.0",
  "description": "",
  "main": "src/index.ts",
  "scripts": {
    "build": "webpack --mode=production"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "webpack": "^5.64.4",
    "webpack-cli": "^4.9.1"
  }
}
```

于是我们可以激动地编写我们的 `index.ts` 文件了，简单尝试一点 ts 语法吧。

```typescript
// src/index.ts
const a: string = 'a';
console.log(a);
```

直接运行 `npm run build`！

```javascript
ERROR in ./src/index.ts 1:7
Module parse failed: Unexpected token (1:7)
You may need an appropriate loader to handle this file type, currently no loaders are configured to process this file. See https://webpack.js.org/concepts#loaders
> const a: string = 'a';
| console.log(a);
| 

webpack 5.64.4 compiled with 1 error in 222 ms
```

然后毫无疑问地出现了问题。这是因为我们要编译的文件 `index.ts` 是 ts 文件，但 webpack 本身是不能解析 ts 的，所以我们需要单独处理 ts。此处参考 webpack 官网 TypeScript 一章。

## TypeScript

首先安装解析 ts 要用的工具。

```javascript
> npm i typescript ts-loader -D
```

然后我们需要在项目中单独配置 `tsconfig.json`。

```json
// tsconfig.json
{
    "compilerOptions": {
        "outDir": "./dist",
        "target": "es5", // 指定编译到的 es 版本 es5
        "module": "es6", // 指定编译到的模块系统 es6
        "strict": true, // 使用严格类型检查
        "allowJs": true, // 允许使用 js 语法
        "jsx": "react", // 指定 jsx 语法 react
        "allowSyntheticDefaultImports": true // 允许使用类似 import React from 'react' 的语法
    }
}
```

以及 webpack 配置：

```javascript
// webpack.config.js
const path = require('path'); // 导入path模块

module.exports = {
  // 入口文件
  entry: './src/index.ts',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/, // 排除 node_modules 目录
      },
    ],
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'], // 自动解析确定的扩展
  },
  // 出口文件
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
};

```

到此我们完成了 webpack 下 ts 的简单配置，为了避免“站在 bug 的肩膀上开发”的问题，我们先来测试一下至今为止的配置是否正确有效。再次运行 `npm run build`！

```javascript
asset bundle.js 40 bytes [compared for emit] [minimized] (name: main)
./src/index.ts 46 bytes [built] [code generated]
webpack 5.64.4 compiled successfully in 2815 ms
```

成功了！在 `dist/bundle.js` 里看下我们编译的成果吧。

```javascript
// dist/bundle.js
(()=>{"use strict";console.log("a")})();
```

用 node 跑一下，发现确实是可以输出 a 的。那么至此 ts 部分我们就配置完成了。

## React

还没完，我们还需要 react。

```javascript
> npm i react react-dom
> npm i @types/react @types/react-dom -D  # 使用 ts 时记得安装相关声明文件
```

好了。没错这样就好了，因为我们在 `tsconfig.json` 中配置过 `"jsx": "react"`，这样在编译 tsx 时 ts 就能自动帮我们将 tsx 编译成 react 了。口说无凭，不如我们来测试下，写一个 `App.tsx`。

```react&#x20;tsx
// src/App.tsx
import React from 'react';

const App: React.FC = () => {
  return (
    <div>
      <h1>Hello React + TypeScript!</h1>
    </div>
  );
}

export default App;

```

记得在 `index.tsx` 中导入 `App.tsx` 并渲染出来。（因为 `index.ts` 中要写 tsx 所以它变成了 `index.tsx`，顺便，记得改 webpack 入口）

```react&#x20;tsx
// src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';

ReactDOM.render(<App />, document.getElementById('root'));

```

来试着编译一下 `npm run build`，我们可以看到 `bundle.js` 确实变成了一大堆，我们确实编译成功了。

等等，好像少了点什么，html 呢？我们要如何查看写出来的效果？

在解决这个问题之前，我们先用一点笨办法来测试我们现在写的对不对，那就是直接在 dist 里写一个 html 运行。

```html
// dist/index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="root"></div>
  <script src="./bundle.js"></script>
</body>
</html>
```

现在打开 `index.html` 我们就可以看到我们的成果了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e48f1434c0d4428d90739b54d618b6e0\~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

看，用 webpack 从零配置一个 react+ts 开发环境也没那么难。

# 优化我们的配置

## HtmlWebpackPlugin

当然，直接在 dist 里新建一个 `index.html` 有那么“一点”不优雅。我们想要的是，只需要写一个 `index.html` 模板，甚至都不需要提供要引入的 script 标签，webpack 就能帮我们自动引入，并且把它放到 dist 目录下。`HtmlWebpackPlugin` 就是做这个的。

```javascript
> npm i html-webpack-plugin -D
```

在 webpack 中使用。

```javascript
// webpack.config.js
const path = require('path'); // 导入path模块
const HtmlWebpackPlugin = require('html-webpack-plugin'); // 导入html-webpack-plugin模块

module.exports = {
  // 入口文件
  entry: './src/index.tsx',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/, // 排除 node_modules 目录
      },
    ],
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'], // 自动解析确定的扩展
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, './template/index.html'), // 模板位置
      filename: 'index.html', // 输出后的文件名，路径是 output.path
      title: 'Nano', // 传给模板的变量
    }),
  ],
  // 出口文件
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
};

```

调整一下 `index.html` 的内容和位置。

```html
// template/index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= htmlWebpackPlugin.options.title %></title>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

好，我们现在可以编译查看结果了。

```html
<!doctype html><html lang="en"><head><meta charset="UTF-8"><meta http-equiv="X-UA-Compatible" content="IE=edge"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Nano</title><script defer="defer" src="bundle.js"></script></head><body><div id="root"></div></body></html>
```

虽然它很丑，但是我们还是可以看出它确实在 dist 目录下，也确实把 bundle.js 加入进去了，甚至还替换了 `htmlWebpackPlugin.options.title`，这就是我们想要的结果。现在我们可以优雅地写 react+ts 了。

## Webpack Dev Server

但是我们的开发体验还是不够爽，我们每次修改一点，都要重新执行 `npm run build`，它还慢得要死，然后我们还得手动打开 html 文件，才能看到我们刚刚修改的那行代码对不对。这样并不适合开发！我们需要的是一个可以实时监听我们所做的改动并重新编译，编译时效率还高的东西。这个时候我们有两个选择，第一个是使用 webpack 的 `--watch` 选项，它可以监听我们对文件做的改动并自动重新编译；第二个是使用 `webpack-dev-server`。我们选择后者，因为它配置项更丰富，更能满足需求（自动刷新页面）。

依旧是安装 `npm i webpack-dev-server -D`。然后在 `webpack.config.js` 中调整相应配置 `devServer`。

```javascript
// webpack.config.js
...
  devServer: {
    static: {
      directory: path.resolve(__dirname, 'dist'), // 告诉服务器从指定目录中提供静态文件，也即打包后的文件
    },
    port: 8080, // 设置端口
    open: true, // 自动打开浏览器
    hot: true, // 开启热更新
  },
...
```

最后设置下 npm 脚本 `"dev": "webpack serve --mode=development"`，就可以轻松地实现更新后自动编译了。

## Webpack 缓存

我们可以发现，在每次 `npm run dev` 时，我们都会把 react 重新编译一遍，尽管它根本没有变过。因此我们自然可以想到可以用缓存来提高我们编译的效率。Webpack5 引入了缓存来提高二次构建速度，只要在 `webpack.config.js` 中配置一下 `cache` 就可以实现缓存，轻松提高编译效率。

```javascript
// webpack.config.js
...
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename], // 构建依赖的config文件（通过 require 依赖）内容发生变化时，缓存失效
    },
  },
...
```

这样，在二次构建时，构建速度便明显提高了。

## git

最后，我们使用 git 创建一下仓库 `git init`，同时我们还需要有一个 `.gitignore` 文件，毕竟我们可不想把全宇宙最重的东西 `node_modules` 提交上去。

```javascript
// .gitignore
node_modules
dist
```

好了，现在我们可以愉快地提交了。

# 总结

现在目录结构如下：

```javascript
nano
├── dist
├── node_modules
├── package-lock.json
├── package.json
├── src
│   ├── App.tsx
│   └── index.tsx
├── template
│   └── index.html
├── tsconfig.json
└── webpack.config.js
```

虽然目前已经在项目中配置了 webpack、ts、react 了，但这离我们的项目初始化还很远。首先，我们希望在写 tsx 时可以 import 更多种类的文件，比如 css、less、图片等，这就需要配置 webpack loader 以满足对其他文件的解析。同时，为了保证协同开发中的规范性，我们要使用更多的协作规范工具，比如 eslint、prettier、commitlint、commitizen 等等，这样能保证我们团队协作更加规范。
