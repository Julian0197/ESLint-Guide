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

将刚刚配置的eslint插件安装在monorepo架构下的根目录，方便其他packages下的库共享。

`pnpm install eslint-plugin-colint@workspace -w`

再进入测试仓库`eslint-guide`，将上面创建的ESLint插件在`.eslitrc.js`中配置：

~~~js
{
    'plugins': ['colint'],
    'rules': {
        'colint/no-var': ['error']
    }
}
~~~

注意：ESLint 插件在配置项 plugins 中只需要写插件名称即可。`eslint-plugin-` 此部分不用填写。

我们在`test.js`下测试使用var声明变量。就能看到以下报错，说明eslint插件配置生效：

`Unexpected var, use let or const instead.eslintno-var`

eslint是基于ast的，`var`对应的ast节点为`VariableDeclaration`(变量声明)，所以我们只需要在规则函数中创建一个`VariableDeclaration` 的**访问者函数**进行监听对应的 AST 节点然后进行相应的操作即可。

~~~js
create(context) {
    return {
      VariableDeclaration(node) {
        console.log(node)
      }
    }
  }
~~~

通过 `npx eslint test.js` 把插件运行起来，查看 console.log 打印的数据

~~~js
Node {
  type: 'VariableDeclaration',
  start: 0,
  end: 14,
  loc: SourceLocation {
    start: Position { line: 1, column: 0 },
    end: Position { line: 1, column: 14 }
  },
  range: [ 0, 14 ],
  declarations: [
    Node {
      type: 'VariableDeclarator',
      start: 4,
      end: 14,
      loc: [SourceLocation],
      range: [Array],
      id: [Node],
      init: [Node],
      parent: [Circular *1]
    }
  ],
  kind: 'var',
  parent: Node {
    type: 'Program',
    start: 0,
    end: 14,
    loc: SourceLocation { start: [Position], end: [Position] },
    range: [ 0, 14 ],
    body: [ [Circular *1] ],
    sourceType: 'script',
    comments: [],
    tokens: [ [Token], [Token], [Token], [Token] ],
    parent: null
  }
}
~~~

有因为变量声明有 const、let、var 所以又进一步判断是不是 var，如果是 var 那么我们就可以通过上下文对象 context.report() 方法进行发布警告或错误，相关代码如下：

~~~js
module.exports = {
  create(context) {
    return {
      VariableDeclaration(node) {
        if (node.kind === 'var') {
          context.report({
            node,
            message: '不能用var'
          })
        }
      }
    }
  }
}
~~~

接着再运行 `npx eslint test.js`,发现提示报错采用我们所写的提示语

#### 如何修复代码

需要修复代码，必须把`no-var.js`中`meta.fixable`设置为`code`。要修复代码则可以在 `context.report()` 方法的参数对象中传递一个 `fix()` 函数，这个 `fix()` 函数有一个回调参数对象 fixer 就提供了各种修改方法。

修复代码的核心在于：通过当前的 AST 节点信息找到对应的 tokens 中的具体 token，因为只有 tokens 中的 token 才详细记录了字符标记的具体**位置信息**内容。

接下来，我们先通过上下对象 context.getSourceCode() 方法获取到的对象就是 ESLint 中是通过一个 SourceCode 类对通过编译器生成的原始 AST 进行二次封装的实例对象。

这个实例对象提供了非常多方法：

+ getText(node) - 返回给定节点的源码。省略 node，返回整个源码。
+ getAllComments() - 返回一个包含源中所有注释的数组。
+ getCommentsBefore(nodeOrToken) - 返回一个在给定的节点或 token 之前的注释的数组。
+ getCommentsAfter(nodeOrToken) - 返回一个在给定的节点或 token 之后的注释的数组。
+ getCommentsInside(node) - 返回一个在给定的节点内的注释的数组。
+ getJSDocComment(node) - 返回给定节点的 JSDoc 注释，如果没有则返回 null。
+ isSpaceBetweenTokens(first, second) - 如果两个记号之间有空白，返回 true
+ getFirstToken(node, skipOptions) - 返回代表给定节点的第一个 token

我们将通过`getFirstToken`拿到`var`，并通过`context.report`方法中的`fix()`函数的`fixer.replaceText()`进行替换修复。

~~~js
create(context) {
    const sourceCode = context.getSourceCode();
    return {
      VariableDeclaration(node) {
        if (node.kind === "var") {
          context.report({
            node,
            data: { type: "var" },
            messageId: "unexpected",
            fix(fixer) {
              const varToken = sourceCode.getFirstToken(node, {
                filter: (t) => t.value === "var",
              });
              return fixer.replaceText(varToken, "let");
            },
          });
        }
      },
    };
  }
~~~

最后运行`npx eslint test.js --fix`，完成修复，var => let。

### 创建可共享的ESLint配置

我们希望针对一些相同的项目只进行一次ESLint配置然后在其他项目中直接引用即可，这就是ESLint的共享配置，并且还可以把 ESLint 共享配置上传到 npm 服务器供其他人下载下来在其项目中安装使用。

### ESLint 配置文件中 extends、plugin 及 rules 的区别

在 packages 目录中再创建一个 `eslint-config-share` 目录用于创建 ESLint 共享配置。 进入 `eslint-config-share` 目录初始化项目，运行 `pnpm init`。在`index.js` 文件中进行以下配置：

这里的`extends`相当于使用了`eslint-plugin-colint`插件的recommend下rules的配置选项。由于采用monorepo架构已经将该插件安装到工作区，所以可以直接拿到，一般情况eslint规则都会以npm包的形式发布。
~~~js
module.exports = {
  extends: ['plugin:colint/recommended']
}
~~~

修改`no-var.js`：
~~~js
module.exports = {
  rules: requireIndex(__dirname + "/rules"),
  configs: {
    recommended: {
      plugins: ['colint'],
      rules: {
        'colint/no-var': ['error']
      }
    }
  }
}
~~~

#### 规则rules

ESLint 的核心就是它的规则，通过规则进行代码校验和代码修复。

我们将刚刚编写的ESLint插件的`rules`文件夹放到`eslint-guide`测试目录下，对`.eslintrc.js`进行配置：

~~~js
module.exports = {
    'rules': {
        'no-var': ['error']
    }
}
~~~

在运行: `npx eslint test.js --rulesdir rules` `--rulesdir` 指定运行的规则目录

可以发现rules可以在脱离ESLint插件框架后正常运行。但如果想讲插件共享给其他人，ESLint提供了`plugins`插件机制实现。

#### plugins插件

插件通常是针对某一种特定场景定义开发的一系列规则，例如 `eslint-plugin-vue` 就是针对 Vue 项目开发的 ESLint 插件，`eslint-plugin-prettier` 就是针对 Prettier 开发的 ESLint 插件。

.eslintrc配置如下：
~~~js
// .eslintrc.js
{
  plugins: ['prettier'],
  rules: {
    // prettier
  	'prettier/prettier': 'error',
  }
}
~~~

引入插件，需要在rules里指定需要的规则，也可以用插件本身推荐选项，需要进行下面的配置：

~~~js
// .eslintrc.js
{
  extends: ['plugin:vue/vue3-recommended']
  // 即便在 extends 引用了推荐的配置选项，但你还是可以在 rules 选项中进行重新配置相关 rules。
  rules: {
    'vue/no-v-html': 'off'
    }
}
~~~

在 `extends` 选项中进行设置，其中 `plugin:` 表示这是一个插件，后面跟着的就是插件名称，`/` 后面表示的该插件配置集成选项，也就是该插件已经进行相关的 `rules` 设置，使用者只需要使用它推荐的就可以了。

#### extends 继承

`extends` 选项除了配置 ESLint 插件之外，还可以配置一个 `npm` 包，也可以理解为继承他人的 ESLint 配置方案，可以看成实际就是一份别人配置好的 `.eslintrc.js` 。

**注意：**ESLint 的共享配置 npm 包必须是以 `eslint-config-xxx` 开头或者 `@xxx/eslint-config` 此种类型，其中 xxx 是具体的名字，这是 ESLint 共享配置 npm 包名称的约定。

下面时 `element plus` ESlint配置的基础项：

~~~js
extends: [
    'eslint:recommended',
    'plugin:import/recommended', // 插件中的 extends
    'plugin:eslint-comments/recommended',
    'plugin:jsonc/recommended-with-jsonc',
    'plugin:markdown/recommended',
    'plugin:vue/vue3-recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier',
],
~~~

