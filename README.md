## ESLint + 代码规范自动化实战

ESLint作用：检测语法，发现问题，强制代码风格

### 创建可共享的 ESLint 插件

#### 创建eslint开发环境

通过pnpm创建一个monorepo项目，
+ `eslint-guide` 目录下进行测试 ESLint 相关配置，
+ `eslint-plugin-colint` 目录用于创建 ESLint 插件。

~~~csharp
├── README.md
├── package.json
├── packages
│   ├── eslint-guide
│   │   ├── .eslintrc.js
│   │   └── package.json
│   └── eslint-plugin-colint
│       └── package.json
└── pnpm-workspace.yaml
~~~

进入 `eslint-plugin-colint` 目录运行以下命令：`npm install yo generator-eslint -g`，生成一个 ESLint 的插件模板。

运行命令：`yo eslint:plugin`，出现以下命令界面，然后按提示填写：

~~~csharp
? What is your name? msk
? What is the plugin ID? colint
? Type a short description of this plugin: test
? Does this plugin contain custom ESLint rules? Yes
? Does this plugin contain one or more processors? (y/N) n
~~~

ESLint 的插件 npm 包的名称必须以 `eslint-plugin-` 开头，而我们上面通过命令创建的 ESLint 插件则自动命名为了 `eslint-plugin-colint`，我们查看一下 `eslint-plugin-colint` 目录下的 `package.json` 的 name 属性值就知道了。

运行命令：`yo eslint:rule`，生成规则模板，出现以下命令界面，然后按提示填写：

~~~csharp
? What is your name? msk
? Where will this rule be published? (Use arrow keys)

> ESLint Plugin 选择插件
> ESLint Core
? What is the rule ID? no-var
? Type a short description of this rule: test
? Type a short example of the code that will fail:
~~~

接着会在 lib 目录下生成一个 rules 目录及一个 `no-var.js` 的文件。`no-var.js` 的文件内容，也是一个 ESLint 插件的最基本结构：

~~~js
/**
 * @fileoverview test
 * @author msk
 */
'use strict'
module.exports = {
  meta: {
    type: null, // `problem`, `suggestion`, or `layout`
    docs: {
      description: 'test',
      recommended: false,
      url: null // URL to the documentation page for this rule
    },
    fixable: null, // Or `code` or `whitespace`
    schema: [] // Add a schema if the rule has options
  },

  create(context) {
    return {
      // visitor functions for different types of nodes
    }
  }
}
~~~

#### 调试&编写eslint插件