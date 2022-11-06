---
title: 使用 Webpack5 从零架一个 React+TS 项目（二）
date: 2021-11-29
tags:
  - 工程化
  - Webpack
---

# css & less 配置

书接上文，我们已经完成了 tsx 的基本配置，可以直接写 tsx 了。但问题来了，我们编写网页不可能只写 html 和 js 吧，还需要 css 文件，所以不妨让我们来简单写一下 css。

```react&#x20;tsx
// src/App.tsx
import React from 'react';
import './App.css';

const App: React.FC = () => {
  return (
    <div>
      <h1 id="title">Hello React + TypeScript!</h1>
    </div>
  );
};

export default App;

```

```css
// src/App.css
#title {
  color: red;
  border: 1px solid aqua;
}
```

编译之后发现报错了，很显然是因为我们用 `import` 把 `App.css` 文件交给了 webpack 去解析，但 webpack 本身也无法解析 css 文件。所以归根到底我们还是需要增加一些 loader 来解析 css。这里我们需要的是 `css-loader` 和 `style-loader`，前者用于解析我们 import 的 css 文件（包括使用 `@import` 和 `url()` 引入的文件），后者用于把解析后的样式直接插入到 dom 中，无需在 html 中引入 css 文件。

安装完成后需要在 webpack 中对 css 文件进行配置。

> webpack loader 的执行顺序是链式的，即从右到左，因此下面配置中对 css 的解析顺序实际上是 css-loader -> style-loader

```javascript
// webpack.config.js
...
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
...
```

再次编译我们就可以看到页面上红色的文字了。

于是我们就可以愉快地开发页面了，不如让我们来试着写一个 Header component。

```react&#x20;tsx
// src/components/Header/index.tsx
import React from 'react';
import './index.css';

function Header() {
  return (
    <div>
      <h2 className="subtitle">This is my header</h2>
    </div>
  );
}

export default Header;

```

```css
// src/components/Header/index.css
.subtitle {
  color: red;
  text-decoration: overline;
}
```

然后在 `App.tsx` 中引用组件。

```react&#x20;tsx
// src/App.tsx
import React from 'react';
import Header from './components/Header';
import './App.css';

const App: React.FC = () => {
  return (
    <div>
      <Header />
      <h1 id="title">Hello React + TypeScript!</h1>
      <h2 className="subtitle">This is a subtitle</h2>
    </div>
  );
};

export default App;

```

```css
// src/App.css
#title {
  color: red;
  border: 1px solid aqua;
}

.subtitle {
  color: orange;
}
```

我们预期的结果应该是页面顶部有一个红色的 Header，之后是一行红色一行橘色的文字，但结果并非如此。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d35570bdd39b47c8928cf362c9c453ef\~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

我们发现 `App.tsx` 和 `index.css` 中两个 `.subtitle` 样式冲突了！这会给我们日后的开发带来诸多不便，我们必须时刻提防样式污染，小心谨慎地设置类名，采用 BEM 的方式去保证类名的唯一性。这样做的效率并不算高，我们需要更便捷一些，我们需要的是 css 模块化！好消息是我们可以在 `css-loader` 中设置 css module 来实现模块化。

```javascript
// webpack.config.js
...
      {
        test: /\.css$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: true,
            },
          },
        ],
      },
...
```

这样一来 css 编译后会自动将标识符导出，如 `exports.locals = { title: 'V8B7rqnYmo_Ur5uY5B7S', subtitle: 'zJbxMA7L77HGYjVzSX2h' }`，所以在 tsx 中我们需要 `import style from './App.css'` 以导入这个 style 对象，然后使用 `className={style.subtitle}` 以使用它。

但是你应该已经发现了，当我们使用 ts 进行以上操作时，它会报错 `Cannot find module './index.css' or its corresponding type declarations.`。我们的 `style` 对象并没有相关类型声明，自然也无法在 tsx 中使用。这个问题的解决办法有很多，比如可以将 `import style from './index.css'` 改为 `const style = require('./index.css')`，也可以使用 `typings-for-css-modules-loader` 解决。我这里采用了较为折中一点办法，手动增加一个模块类型声明文件。

```typescript
// src/typings/index.d.ts
declare module '*.css' {
  const content: { [selector: string]: string };
  export default content;
}
```

然后在 `tsconfig.json` 中增加声明文件路径就可以使用了。注意，tsconfig 中 `typeRoots` 和 `types` 的设定会替代默认的 `node_modules/@types` 路径，所以设置时要根据具体需求来设置。

> ******，********和******
>
> 默认所有\_可见的\_"`@types`"包会在编译过程中被包含进来。 `node_modules/@types`文件夹下以及它们子文件夹下的所有包都是\_可见的\_； 也就是说， `./node_modules/@types/`，`../node_modules/@types/`和`../../node_modules/@types/`等等。
>
> 如果指定了`typeRoots`，*只有*`typeRoots`下面的包才会被包含进来。
>
> 如果指定了`types`，只有被列出来的包才会被包含进来。

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
        "allowSyntheticDefaultImports": true, // 允许使用类似 import React from 'react' 的语法
        "moduleResolution": "node", // 指定模块解析器 node 允许直接 import node_modules 中的模块，并且允许解析目录
        "typeRoots": [
            "node_modules/@types",
            "src/typings"
        ] // 指定类型文件的搜索目录
    }
}
```

现在终于正常显示了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7be9ced6526d4df69a25d0828da8d1a8\~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

但是还有一个问题，那就是当我们打开控制台时，我们会发现元素的选择器是一堆“乱码”。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1bc5bffbeab74b68a4646d589f5548d8\~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

css module 在转变选择器时默认直接把选择器替换为 base64 转换后的字符串，但这并不方便我们调试，我们需要手动在 `css-loader` 的 module 选项中设置转换后的名字。

```javascript
// webpack.config.js
...
          {
            loader: 'css-loader',
            options: {
              modules: {
                localIdentName: '[path][name]__[local]--[hash:base64:5]',
              }
            },
          },
...
```

这样类名就比较清晰了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a69d4ae0502540a4b2da9752db648ba4\~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

接着我们来配置 css 预处理器——less。依旧是安装 `npm i less-loader less -D`。然后只需要将所有的 css 文件改成 less 文件，再修改一下 tsx 中的导入，再修改一下 `webpack.config.js` 中对 less 文件的解析，最后再修改一下 `typings` 声明文件就好了。来动手试一下。

```javascript
// webpack.config.js 
...
      {
        test: /\.less$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: {
                localIdentName: '[path][name]__[local]--[hash:base64:5]',
              },
            },
          },
          {
            loader: 'less-loader',
          },
        ],
      },
...
```

这样一番操作之后，我们便可以在项目中使用 less 预处理器开发了，并且和我们之前配置的 css module 也没有冲突。

# 代码规范化

我相信大部分经历过团队开发的人都遇到过代码协作上的问题，要么是有人加分号有人不加分号，要么是有人以 lf 结尾有人以 crlf 结尾。但不论哪种，都会让代码协作变得非常繁琐。这时就急需一种能够检查代码中的问题、统一代码风格的工具，那便是 eslint+prettier+.editorconfig。在此说明，我平常使用的 eslint 规范是 [alloyteam 规范](https://link.juejin.cn?target=https://github.com/AlloyTeam/eslint-config-alloy/blob/master/README.zh-CN.md "alloyteam 规范")，这里也清楚地写了如何使用这一套规范，我们照着说明使用便可。

首先是安装 eslint 及相关规范插件 `npm i eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-plugin-react eslint-config-alloy -D`。

然后在根目录中新建 `.eslintrc.js` 文件，并配置插件，我们直接照着 alloyteam 规范的使用说明配置即可。

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'alloy',
    'alloy/react',
    'alloy/typescript',
  ],
  env: {
    // 你的环境变量（包含多个预定义的全局变量）
    //
    // browser: true,
    // node: true,
    // mocha: true,
    // jest: true,
    // jquery: true
  },
  globals: {
    // 你的全局变量（设置为 false 表示它不允许被重新赋值）
    //
    // myGlobal: false
  },
  rules: {
    // 自定义你的规则
  },
};

```

顺便配置一下 `.eslintignore` 文件。

```javascript
node_modules
*.json
webpack.config.js
*.css
*.less
*.html
```

这时 eslint 便已经配置完毕了，最后再增加一个 npm 脚本用于手动 lint `"lint": "eslint */**"`。

接着我们来配置 prettier。prettier 比 eslint 更专注于代码格式方面，有了 prettier，无论你是哪种编写习惯，我们都能够统一码风。按照惯例先安装 `npm i prettier -D`。

prettier 的配置比 eslint 简单地多，它只需要在根目录的 `.prettierrc.js` 中配置各项规则即可，这里我依旧是使用 alloyteam 推荐配置。

```javascript
// .prettierrc.js
module.exports = {
  // 一行最多 120 字符
  printWidth: 120,
  // 使用 2 个空格缩进
  tabWidth: 2,
  // 不使用缩进符，而使用空格
  useTabs: false,
  // 行尾需要有分号
  semi: true,
  // 使用单引号
  singleQuote: true,
  // 对象的 key 仅在必要时用引号
  quoteProps: 'as-needed',
  // jsx 不使用单引号，而使用双引号
  jsxSingleQuote: false,
  // 末尾需要有逗号
  trailingComma: 'all',
  // 大括号内的首尾需要空格
  bracketSpacing: true,
  // jsx 标签的反尖括号需要换行
  bracketSameLine: false,
  // 箭头函数，只有一个参数的时候，也需要括号
  arrowParens: 'always',
  // 每个文件格式化的范围是文件的全部内容
  rangeStart: 0,
  rangeEnd: Infinity,
  // 不需要写文件开头的 @prettier
  requirePragma: false,
  // 不需要自动在文件开头插入 @prettier
  insertPragma: false,
  // 使用默认的折行标准
  proseWrap: 'preserve',
  // 根据显示样式决定 html 要不要折行
  htmlWhitespaceSensitivity: 'css',
  // vue 文件中的 script 和 style 内不用缩进
  vueIndentScriptAndStyle: false,
  // 换行符使用 lf
  endOfLine: 'lf',
  // 格式化内嵌代码
  embeddedLanguageFormatting: 'auto',
};

```

然后修改一下我们刚刚配置的 npm 脚本 `"lint": "prettier --write . && eslint */**"`，这里 prettier 的 write 参数是指 prettier 会自动修改不符合规范的地方。

好了，这下我们直接 `npm run lint` 就可以看着我们的代码规范化了。当然如果你想达到“每次保存代码都格式化”的效果，那么你可以在 `.vscode/settings.json` 中进行配置。

```json
// .vscode/settings.json
{
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[typescriptreact]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[vue]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    }
}
```

接下来我们来让我们的 less 文件更规范一些。stylelint 便是一个用于 CSS 审查的工具，它可以统一 CSS 代码。在 webpack 中使用 stylelint 需要安装专用于 webpack 的 plugin，同时我们也可以像 eslint 一样安装一些推荐配置 `npm i stylelint stylelint-webpack-plugin stylelint-config-standard stylelint-config-css-modules stylelint-order -D`。

注意：为了让 stylelint 能够适配 less 语法，我们需要安装 `postcss-less` 插件并在 `stylelint-webpack-plugin` 中设置 `customSyntax` 属性。

使用 stylelint 需要在 `webpack.config.js` 中配置 plugin，并且也是在 `.stylelintrc.js` 中单独配置规则。此处我也使用了网络上一份比较完整的配置方案。

```javascript
// webpack.config.js
...
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, './template/index.html'), // 模板位置
      filename: 'index.html', // 输出后的文件名，路径是 output.path
      title: 'Nano', // 传给模板的变量
    }),
    new StylelintWebpackPlugin({
      configFile: path.resolve(__dirname, './.stylelintrc.js'),
      files: ['src/**/*.{less,css}'],
      customSyntax: 'postcss-less', // 适配 less 语法
      fix: true, // 自动格式化
      lintDirtyModulesOnly: true, // 仅检查变化的代码
      threads: true, // 多线程
    }),
  ],
...
```

```javascript
// .stylelintrc.js
module.exports = {
  extends: ['stylelint-config-standard', 'stylelint-config-css-modules'],
  plugins: ['stylelint-order'],
  rules: {
    'selector-class-pattern': [
      // 命名规范 -
      /^([a-z][a-z0-9]*)(-[a-z0-9]+)*$/,
      {
        message: 'Expected class selector to be kebab-case',
      },
    ],
    'string-quotes': 'single', // 单引号
    'at-rule-empty-line-before': null,
    'at-rule-no-unknown': null,
    'at-rule-name-case': 'lower', // 指定@规则名的大小写
    'length-zero-no-unit': true, // 禁止零长度的单位（可自动修复）
    'shorthand-property-no-redundant-values': true, // 简写属性
    'number-leading-zero': 'never', // 小数不带0
    'declaration-block-no-duplicate-properties': true, // 禁止声明快重复属性
    'no-descending-specificity': true, // 禁止在具有较高优先级的选择器后出现被其覆盖的较低优先级的选择器。
    'selector-max-id': 1, // 限制一个选择器中 ID 选择器的数量
    'max-nesting-depth': 3,
    indentation: [
      2,
      {
        // 指定缩进  warning 提醒
        severity: 'warning',
      },
    ],
    'order/properties-order': [
      // 规则顺序
      'position',
      'top',
      'right',
      'bottom',
      'left',
      'z-index',
      'display',
      'float',
      'width',
      'height',
      'max-width',
      'max-height',
      'min-width',
      'min-height',
      'padding',
      'padding-top',
      'padding-right',
      'padding-bottom',
      'padding-left',
      'margin',
      'margin-top',
      'margin-right',
      'margin-bottom',
      'margin-left',
      'margin-collapse',
      'margin-top-collapse',
      'margin-right-collapse',
      'margin-bottom-collapse',
      'margin-left-collapse',
      'overflow',
      'overflow-x',
      'overflow-y',
      'clip',
      'clear',
      'font',
      'font-family',
      'font-size',
      'font-smoothing',
      'osx-font-smoothing',
      'font-style',
      'font-weight',
      'line-height',
      'letter-spacing',
      'word-spacing',
      'color',
      'text-align',
      'text-decoration',
      'text-indent',
      'text-overflow',
      'text-rendering',
      'text-size-adjust',
      'text-shadow',
      'text-transform',
      'word-break',
      'word-wrap',
      'white-space',
      'vertical-align',
      'list-style',
      'list-style-type',
      'list-style-position',
      'list-style-image',
      'pointer-events',
      'cursor',
      'background',
      'background-color',
      'border',
      'border-radius',
      'content',
      'outline',
      'outline-offset',
      'opacity',
      'filter',
      'visibility',
      'size',
      'transform',
    ],
  },
};

```

这样我们在使用 webpack 打包时就可以自动分析 css 并纠正了。同时为了满足手动纠正的需要，我们配置一个 npm 脚本。

```json
"lint:js": "prettier --write . && eslint **/*",
"lint:css": "stylelint \"src/**/*.{less,css}\" --config ./.stylelintrc.js --custom-syntax postcss-less --fix"
```

这样我们就可以规范开发中的样式文件了！

最后我们来简单配置一下 `.editorconfig`。不知道什么是 `.editorconfig` 的点[这里](https://link.juejin.cn?target=https://editorconfig.org/ "这里")，简单来说 `.editorconfig` 文件可以跨编辑器同步一部分配置，如缩进、结尾等。这里我们简单配置一下：

```javascript
// .editorconfig
root = true

[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8
indent_style = space

[*.{js,ts,tsx,html,css}]
indent_size = 2

[*.{json}]
indent_size = 4

```

至此，我们一些基本的代码格式化配置就完成了，这样我们以后在和小伙伴协作开发的时候就再也不用纠结码风听谁的问题了。

# 提交规范化

但还有个问题，你的小伙伴每次都不运行 eslint 怎么办？尽管你辛辛苦苦配置了大半天，他仍然无动于衷甚至想把 eslint 直接 uninstall 掉。这个时候我们就得发话了，“不通过 eslint 的代码不许提交！”，但仍然收效甚微，你的小伙伴在开发时还是会写一堆后直接 commit。有没有什么办法让一个人在他 commit 的时候跳出来跟他说你不能 commit 呢？

有！这就是 git hook。一般我们使用 husky 实现 git hook 功能，它可以在“将要提交前”执行一些代码 `npm i husky -D`。安装完成 husky 之后，我们首先要在根目录中执行 `npx husky install` 操作以启用 git hooks。然后使用命令 `npx husky add .husky/pre-commit "npm run lint:js && npm run lint:css"` 便可以在 commit 之前自动执行 `npm run lint:js && npm run lint:css` 指令。

我们在 tsx 里随便写一个错误，执行 `git add . && git commit -m 'test'` 便可以看到 eslint 的报错了。这样实现了提交时“半强制”检查。

> 实际上如果某些人真的想绕过检查，这种方法也不堪一击。这种情况下真正稳定办法是将检查放到 cicd 里。不过我们团队没有这种“有心人”，所以不必大费周章。

还有最后一个问题，那就是 commit 信息的规范化。相信大家都见过不规范的 git commit，有的过于离谱，有的过于随意。因为怕当事人打我，我就不放截图了。

规范 commit 信息也是规范化不可缺少的一环。我们可以使用 husky 来实现它。它可以在 commit-msg 时运行 `commitlint` 检查 commit 信息。

首先当然是 `npm i @commitlint/config-conventional @commitlint/cli -D`。commitlint 也需要简单配置。

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // 提交的类别
    'type-enum': [
      2,
      'always',
      ['build', 'ci', 'chore', 'docs', 'feat', 'fix', 'perf', 'refactor', 'revert', 'style', 'test'],
    ],
  },
};

```

然后在 husky 中简单增加一个 hook `npx husky set .husky/commit-msg 'npx commitlint --edit "$1"'`。这里的 `$1` 就是老版本中的 `HUSKY_GIT_PARAMS` 也就是提交的信息。

现在我们尝试 `git commit -m 'test'` 的时候，就会给我们报如下错误了：

```javascript
⧗   input: test
✖   subject may not be empty [subject-empty]
✖   type may not be empty [type-empty]

✖   found 2 problems, 0 warnings
ⓘ   Get help: https://github.com/conventional-changelog/commitlint/#what-is-commitlint

husky - commit-msg hook exited with code 1 (error)
```

这样就不会出现奇奇怪怪的提交信息了。有需要的小伙伴可以使用 `commitizen` 来辅助自己提交符合规范的 commit，它需要安装在全局环境下（也可以在项目局部环境中），这里我就不配置了。

终于，小伙伴们可以心平气和地在一起开发了，再也没有奇奇怪怪的问题了。
