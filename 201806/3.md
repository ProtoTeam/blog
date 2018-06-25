# Webpack 4 配置最佳实践

> 作者 Daniel 蚂蚁金服·数据体验技术团队

Webpack 4 发布已经有一段时间了。Webpack 的版本号已经来到了 4.12.x。但因为 Webpack 官方还没有完成迁移指南，在文档层面上还有所欠缺，大部分人对升级 Webpack 还是一头雾水。

不过 Webpack 的开发团队已经写了一些零散的文章，官网上也有了新版配置的文档。社区中一些开发者也已经成功试水，升级到了 Webpack 4，并且总结成了博客。所以我也终于去了解了 Webpack 4 的具体情况。以下就是我对迁移到 Webpack 4 的一些经验。

本文的重点在：

* Webpack 4 在配置上带来了哪些便利？要迁移需要修改配置文件的哪些内容？
* 之前的 Webpack 配置最佳实践在 Webpack 4 这个版本，还适用吗？

<!-- more -->

### Webpack 4 之前的 Webpack 最佳实践

这里以 Vue 官方的 Webpack 模板 [vuejs-templates/webpack](https://github.com/vuejs-templates/webpack) 为例，说说 Webpack 4 之前，社区里比较成熟的 Webpack 配置文件是怎样组织的。

#### 区分开发和生产环境

大致的目录结构是这样的：

```
+ build
+ config
+ src

```

在 build 目录下有四个 webpack 的配置。分别是：

* webpack.base.conf.js
* webpack.dev.conf.js
* webpack.prod.conf.js
* webpack.test.conf.js

这分别对应开发、生产和测试环境的配置。其中 webpack.base.conf.js 是一些公共的配置项。我们使用 [webpack-merge](https://github.com/survivejs/webpack-merge) 把这些公共配置项和环境特定的配置项 merge 起来，成为一个完整的配置项。比如 webpack.dev.conf.js 中：

```javascript
'use strict'
const merge = require('webpack-merge')
const baseWebpackConfig = require('./webpack.base.conf')

const devWebpackConfig = merge(baseWebpackConfig, {
   ...
})
```

这三个环境不仅有一部分配置不同，更关键的是，每个配置中用 `webpack.DefinePlugin` 向代码注入了 `NODE\_ENV` 这个环境变量。

这个变量在不同环境下有不同的值，比如 dev 环境下就是 development。这些环境变量的值是在 config 文件夹下的配置文件中定义的。Webpack 首先从配置文件中读取这个值，然后注入。比如这样：

*build/webpack.dev.js*

```javascript
plugins: [
  new webpack.DefinePlugin({
    'process.env': require('../config/dev.env.js')
  }),
]
```

*config/dev.env.js*

```javascript
module.exports ={
  NODE_ENV: '"development"'
}
```

至于不同环境下环境变量具体的值，比如开发环境是 development，生产环境是 production，其实是大家约定俗成的。

框架、库的作者，或者是我们的业务代码里，都会有一些根据环境做判断，执行不同逻辑的代码，比如这样：

```javascript
if (process.env.NODE_ENV !== 'production') {
  console.warn("error!")
}
```

这些代码会在代码压缩的时候被预执行一次，然后如果条件表达式的值是 true，那这个 true 分支里的内容就被移除了。这是一种编译时的死代码优化。这种区分不同的环境，并给环境变量设置不同的值的实践，让我们开启了编译时按环境对代码进行针对性优化的可能。

#### Code Splitting && Long-term caching

Code Splitting 一般需要做这些事情：

* 为 Vendor 单独打包（Vendor 指第三方的库或者公共的基础组件，因为 Vendor 的变化比较少，单独打包利于缓存）
* 为 Manifest （Webpack 的 Runtime 代码）单独打包
* 为不同入口的公共业务代码打包（同理，也是为了缓存和加载速度）
* 为异步加载的代码打一个公共的包

Code Splitting 一般是通过配置 CommonsChunkPlugin 来完成的。一个典型的配置如下，分别为 vendor、manifest 和 vendor-async 配置了 CommonsChunkPlugin。

```javascript
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor',
      minChunks (module) {
        return (
          module.resource &&
          /\.js$/.test(module.resource) &&
          module.resource.indexOf(
            path.join(__dirname, '../node_modules')
          ) === 0
        )
      }
    }),

    new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest',
      minChunks: Infinity
    }),

    new webpack.optimize.CommonsChunkPlugin({
      name: 'app',
      async: 'vendor-async',
      children: true,
      minChunks: 3
    }),
```

CommonsChunkPlugin 的特点就是配置比较难懂，大家的配置往往是复制过来的，这些代码基本上成了模板代码（boilerplate）。如果 Code Splitting 的要求简单倒好，如果有比较特殊的要求，比如把不同入口的 vendor 打不同的包，那就很难配置了。总的来说配置 Code Splitting 是一个比较痛苦的事情。

而 Long-term caching 策略是这样的：给静态文件一个很长的缓存过期时间，比如一年。然后在给文件名里加上一个 hash，每次构建时，当文件内容改变时，文件名中的 hash 也会改变。浏览器在根据文件名作为文件的标识，所以当 hash 改变时，浏览器就会重新加载这个文件。

Webpack 的 Output 选项中可以配置文件名的 hash，比如这样：

```javascript
output: {
  path: config.build.assetsRoot,
  filename: utils.assetsPath('js/[name].[chunkhash].js'),
  chunkFilename: utils.assetsPath('js/[id].[chunkhash].js')
},
```

### Webpack 4 下的最佳实践

#### Webpack 4 的变与不变

Webpack 4 这个版本的 API 有一些 breaking change，但不代表说这个版本就发生了翻天覆地的变化。其实变化的点只有几个。而且只要你仔细了解了这些变化，你一定会拍手叫好。

迁移到 Webpack 4 也只需要检查一下 [checklist](https://dev.to/flexdinesh/upgrade-to-webpack-4---5bc5)，看看这些点是否都覆盖到了，就可以了。

#### 开发和生产环境的区分

Webpack 4 引入了 [mode](https://webpack.js.org/concepts/mode/) 这个选项。这个选项的值可以是 development 或者 production。

设置了 mode 之后会把 `process.env.NODE\_ENV` 也设置为 development 或者 production。然后在 production 模式下，会默认开启 UglifyJsPlugin 等等一堆插件。

Webpack 4 支持零配置使用，可以从命令行指定 entry 的位置，如果不指定，就是 `src/index.js`。mode 参数也可以从命令行参数传入。这样一些常用的生产环境打包优化都可以直接启用。

我们需要注意，Webpack 4 的零配置是有限度的，如果要加上自己想加的插件，或者要加多个 entry，还是需要一个配置文件。

虽然如此，Webpack 4 在各个方面都做了努力，努力让零配置可以做的事情更多。这种内置优化的方式使得我们在项目起步的时候，可以把主要精力放在业务开发上，等后期业务变复杂之后，才需要关注配置文件的编写。

在 Webpack 4 推出 mode 这个选项之前，如果想要为不同的开发环境打造不同的构建选项，我们只能通过建立多个 Webpack 配置且分别设置不同的环境变量值这种方式。这也是社区里的最佳实践。

Webpack 4 推出的 mode 选项，其实是一种**对社区中最佳实践的吸收**。这种思路我是很赞同的。开源项目来自于社区，在社区中成长，从社区中吸收养分，然后回报社区，这是一个良性循环。最近我在很多前端项目中都看到了类似的趋势。接下来要讲的其他几个 Webpack 4 的特性也是和社区的反馈离不开的。

那么上文中介绍的使用多个 Webpack 配置，以及手动环境变量注入的方式，是否在 Webpack 4 下就不适用了呢？其实不然。**在Webpack 4 下，对于一个正经的项目，我们依然需要多个不同的配置文件**。如果我们对为测试环境的打包做一些特殊处理，我们还需要在那个配置文件里用 `webpack.DefinePlugin` 手动注入 `NODE\_ENV` 的值（比如 test）。

> Webpack 4 下如果需要一个 test 环境，那 test 环境的 mode 也是 development。因为 mode 只有开发和生产两种，测试环境应该是属于开发阶段。

#### 第三方库 build 的选择

在 Webpack 3 时代，我们需要在生产环境的的 Webpack 配置里给第三方库设置 alias，把这个库的路径设置为 production build 文件的路径。以此来引入生产版本的依赖。

比如这样：

```javascript
resolve: {
  extensions: [".js", ".vue", ".json"],
  alias: {
    vue$: "vue/dist/vue.runtime.min.js"
  }
},
```

在 Webpack 4 引入了 mode 之后，对于部分依赖，我们可以不用配置 alias，比如 React。React 的入口文件是这样的：

```javascript
'use strict';

if (process.env.NODE_ENV === 'production') {
  module.exports = require('./cjs/react.production.min.js');
} else {
  module.exports = require('./cjs/react.development.js');
}

```

这样就实现了 0 配置自动选择生产 build。

但大部分的第三库并没有做这个入口的环境判断。所以这种情况下我们还是需要手动配置 alias。

#### Code Splitting

Webpack 4 下还有一个大改动，就是废弃了 CommonsChunkPlugin，引入了 `optimization.splitChunks` 这个选项。

`optimization.splitChunks` 默认是不用设置的。如果 mode 是 production，那 Webpack 4 就会开启 Code Splitting。

> 默认 Webpack 4 只会对按需加载的代码做分割。如果我们需要配置初始加载的代码也加入到代码分割中，可以设置 `splitChunks.chunks` 为 `'all'`。

Webpack 4 的 Code Splitting 最大的特点就是配置简单（0配置起步），和__基于内置规则自动拆分__。内置的代码切分的规则是这样的：

* 新 bundle 被两个及以上模块引用，或者来自 node\_modules
* 新 bundle 大于 30kb （压缩之前）
* 异步加载并发加载的 bundle 数不能大于 5 个
* 初始加载的 bundle 数不能大于 3 个

简单的说，Webpack 会把代码中的公共模块自动抽出来，变成一个包，前提是这个包大于 30kb，不然 Webpack 是不会抽出公共代码的，因为增加一次请求的成本是不能忽视的。

具体的业务场景下，具体的拆分逻辑，可以看 [SplitChunksPlugin 的文档](https://webpack.js.org/plugins/split-chunks-plugin/)以及 [webpack 4: Code Splitting, chunk graph and the splitChunks optimization](https://medium.com/webpack/webpack-4-code-splitting-chunk-graph-and-the-splitchunks-optimization-be739a861366) 这篇博客。这两篇文章基本罗列了所有可能出现的情况。

如果是普通的应用，Webpack 4 内置的规则就足够了。

如果是特殊的需求，Webpack 4 的 `optimization.splitChunks` API也可以满足。

splitChunks 有一个参数叫 cacheGroups，这个参数类似之前的 CommonChunks 实例。cacheGroups 里每个对象就是一个用户定义的 chunk。

之前我们讲到，Webpack 4 内置有一套代码分割的规则，那用户也可以自定义 cacheGroups，也就是自定义 chunk。那一个 module 应该被抽到哪个 chunk 呢？这是由 cacheGroups 的抽取范围控制的。每个 cacheGroups 都可以定义自己抽取模块的范围，也就是哪些文件中的公共代码会抽取到自己这个 chunk 中。不同的 cacheGroups 之间的模块范围如果有交集，我们可以用 priority 属性控制优先级。Webpack 4 默认的抽取的优先级是最低的，所以模块会优先被抽取到用户的自定义 chunk 中。

> 


splitChunksPlugin 提供了两种控制 chunk 抽取模块范围的方式。一种是 test 属性。这个属性可以传入字符串、正则或者函数，所有的 module 都会去匹配 test 传入的条件，如果条件符合，就被纳入这个 chunk 的备选模块范围。如果我们传入的条件是字符串或者正则，那匹配的流程是这样的：首先匹配 module 的路径，然后匹配 module 之前所在 chunk 的 name。

比如我们想把所有 node\_modules 中引入的模块打包成一个模块：

```javascript
  vendors1: {
    test: /[\\/]node_modules[\\/]/,
    name: 'vendor',
    chunks: 'all',
  }
```

因为从 node_modules 中加载的依赖路径中都带有 node_modules，所以这个正则会匹配所有从 node_modules 中加载的依赖。

test 属性可以以 module 为单位控制 chunk 的抽取范围，是一种细粒度比较小的方式。splitChunksPlugin 的第二种控制抽取模块范围的方式就是 chunks 属性。chunks 可以是字符串，比如 `'all'|'async'|'initial'`，分别代表了全部 chunk，按需加载的 chunk 以及初始加载的 chunk。chunks 也可以是一个函数，在这个函数里我们可以拿到 `chunk.name`。这给了我们通过入口来分割代码的能力。这是一种细粒度比较大的方式，以 chunk 为单位。

举个例子，比如我们有 a, b, c 三个入口。我们希望 a，b 的公共代码单独打包为 common。也就是说 c 的代码不参与公共代码的分割。

我们可以定义一个 cacheGroups，然后设置 chunks 属性为一个函数，这个函数负责过滤这个 cacheGroups 包含的 chunk 是哪些。示例代码如下：

```
  optimization: {
    splitChunks: {
      cacheGroups: {
        common: {
          chunks(chunk) {
            return chunk.name !== 'c';
          },
          name: 'common',
          minChunks: 2,
        },
      },
    },
  },
```

上面配置的意思就是：我们想把 a，b 入口中的公共代码单独打包为一个名为 common 的 chunk。使用 `chunk.name`，我们可以轻松的完成这个需求。


在上面的情况中，我们知道 chunks 属性可以用来按入口切分几组公共代码。现在我们来看一个稍微复杂一些的情况：对不同分组入口中引入的 node_modules 中的依赖进行分组。

比如我们有 a, b, c, d 四个入口。我们希望 a，b 的依赖打包为 vendor1，c, d 的依赖打包为 vendor2。

这个需求要求我们对入口和模块都做过滤，所以我们需要使用 test 属性这个细粒度比较小的方式。我们的思路就是，写两个 cacheGroup，一个 cacheGroup 的判断条件是：如果 module 在 a 或者 b chunk 被引入，并且 module 的路径包含 `node\_modules`，那这个 module 就应该被打包到 vendors1 中。 vendors2 同理。

```javascript
  vendors1: {
    test: module => {
      for (const chunk of module.chunksIterable) {
			if (chunk.name && /(a|b)/.test(chunk.name)) {
				if (module.nameForCondition() && /[\\/]node_modules[\\/]/.test(module.nameForCondition())) {
                 return true;
             }
			}
	   }
      return false;
    },
    minChunks: 2,
    name: 'vendors1',
    chunks: 'all',
  },
  vendors2: {
    test: module => {
      for (const chunk of module.chunksIterable) {
			if (chunk.name && /(c|d)/.test(chunk.name)) {
				if (module.nameForCondition() && /[\\/]node_modules[\\/]/.test(module.nameForCondition())) {
                 return true;
             }
			}
	   }
      return false;
    },
    minChunks: 2,
    name: 'vendors2',
    chunks: 'all',
  },
};

```

#### Long-term caching

Long-term caching 这里，基本的操作和 Webpack 3 是一样的。不过 Webpack 3 的 Long-term caching 在操作的时候，有个小问题，这个问题是关于 chunk 内容和 hash 变化不一致的：

__在公共代码 Vendor 内容不变的情况下，添加 entry，或者 external 依赖，或者异步模块的时候，Vendor 的 hash 会改变__。

之前 Webpack 官方的专栏里面有一篇文章讲这个问题：[Predictable long term caching with Webpack](https://medium.com/webpack/predictable-long-term-caching-with-webpack-d3eee1d3fa31)。给出了一个解决方案。

这个方案的核心就是，Webpack 内部维护了一个自增的 id，每个 chunk 都有一个 id。所以当增加 entry 或者其他类型 chunk 的时候，id 就会变化，导致内容没有变化的 chunk 的 id 也发生了变化。

对此我们的应对方案是，使用 `webpack.NamedChunksPlugin` 把 chunk id 变为一个字符串标识符，这个字符包一般就是模块的相对路径。这样模块的 chunk id 就可以稳定下来。



![Screen Shot 2018-06-03 at 12.59.28 AM.png | left](https://user-gold-cdn.xitu.io/2018/6/25/16434b5106555162?w=1130&h=422&f=png&s=110259 "")


*这里的 vendors1 就是 chunk id*

> [HashedModuleIdsPlugin](https://webpack.js.org/plugins/hashed-module-ids-plugin/) 的作用和 NamedChunksPlugin 是一样的，只不过 HashedModuleIdsPlugin 把根据模块相对路径生成的 hash 作为 chunk id，这样 chunk id 会更短。因此在生产中更推荐用 HashedModuleIdsPlugin。

这篇文章说还讲到，`webpack.NamedChunksPlugin` 只能对普通的 Webpack 模块起作用，异步模块，external 模块是不会起作用的。

> 异步模块可以在 import 的时候加上 chunkName 的注释，比如这样：import(/* webpackChunkName: "lodash" */ 'lodash').then() 这样就有 Name 了

所以我们需要再使用一个插件：[name-all-modules-plugin](https://github.com/timse/name-all-modules-plugin)

> 这个插件中用到一些老的 API，Webpack 4 会发出警告，这个 [pr](https://github.com/timse/name-all-modules-plugin/pull/2) 有新的版本，不过作者不一定会 merge。我们使用的时候可以直接 copy 这个插件的代码到我们的 Webpack 配置里面。

做了这些工作之后，我们的 Vendor 的 ChunkId 就再也不会发生不该发生的变化了。

### 总结

Webpack 4 的改变主要是对社区中最佳实践的吸收。Webpack 4 通过新的 API 大大提升了 Code Splitting 的体验。但 Long-term caching 中 Vendor hash 的问题还是没有解决，需要手动配置。本文主要介绍的就是 Webpack 配置最佳实践在 Webpack 3.x 和 4.x 背景下的异同。希望对读者的 Webpack 4 项目的配置文件组织有所帮助。

另外，推荐 [*SURVIVEJS - WEBPACK*](https://survivejs.com/webpack/) 这个在线教程。这个教程总结了 Webpack 在实际开发中的实践，并且把材料更新到了最新的 Webpack 4。
 
 > 对我们团队感兴趣的可以关注专栏，关注[github](https://github.com/ProtoTeam/blog)或者发送简历至'tao.qit####alibaba-inc.com'.replace('####', '@')，欢迎有志之士加入~

原文地址：<https://github.com/ProtoTeam/blog/blob/master/201806/3.md>