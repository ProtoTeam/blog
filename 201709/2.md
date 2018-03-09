# TypeScript体系调研报告

>作者简介：aoto 蚂蚁金服·数据体验技术团队


> Q：为什么要写这边文章？这篇文章要表达什么？
>
> A：我们考虑在SPA应用中使用TS作为开发语言，我们需要一篇系统性介绍TS本身及周边的文章来论证在项目中使用TS作为开发语言是科学合理的，而且是顺势而为的。

## 导引

- TS是什么
- 为什么要用TS
- TS能干点什么
- 使用TS的成本
- 社区发展
- 周边生态
- 深入解读TS
- 接受TS
- 权衡

## TS是什么

![TypeScript](https://user-gold-cdn.xitu.io/2017/9/22/8f164338a7ef32ecf35c8acb246f0adc)

TypeScript = Type + Script（标准JS）。我们从TS的官方网站上就能看到定义：`TypeScript is a typed superset of JavaScript that compiles to plain JavaScript`。TypeScript是一个编译到纯JS的有类型定义的JS超集。

## 为什么要用TS
**目标：生命周期较长（常常持续几年）的复杂SPA应用，保障开发效率的同时提升代码的可维护性和线上运行时质量。**
- **从开发效率上看**，虽然需要多写一些类型定义代码，但TS在VSCode、WebStorm等IDE下可以做到智能提示，智能感知bug，同时我们项目常用的一些第三方类库框架都有TS类型声明，我们也可以给那些没有TS类型声明的稳定模块写声明文件，如我们的前端KOP框架(目前还是蚂蚁内部框架，类比dva)，这在团队协作项目中可以提升整体的开发效率。
- **从可维护性上看**，长期迭代维护的项目开发和维护的成员会有很多，团队成员水平会有差异，而软件具有**熵**的特质，长期迭代维护的项目总会遇到可维护性逐渐降低的问题，有了强类型约束和静态检查，以及智能IDE的帮助下，可以降低软件腐化的速度，提升可维护性，且在**重构**时，强类型和静态类型检查会帮上大忙，甚至有了类型定义，会不经意间增加重构的频率（更安全、放心）。
- **从线上运行时质量上看**，我们现在的SPA项目的很多bug都是由于一些调用方和被调用方（如组件模块间的协作、接口或函数的调用）的数据格式不匹配引起的，由于TS有编译期的静态检查，让我们的bug尽可能消灭在编译器，加上IDE有智能纠错，编码时就能提前感知bug的存在，我们的线上运行时质量会更为稳定可控。

> 一个复杂软件的常规研发流程，大致分为定义问题、需求分析、规划构建、软件架构、详细设计、编码调试、单元测试、集成测试、集成、系统测试、保障维护。构建活动（主要是编码调试）在中大型项目中的工作量占比大于50%。同时，一个中大型项目，bug由构建阶段引起的比例占到50%~75%，对于一个成功的项目来说，构建活动是必须要做的，而且是工程师更为可控的。【代码大全】

TS适合大规模JavaScript应用，正如他的官方宣传语`JavaScript that scales`。从以下几点可以看到TS在团队协作、可维护性、易读性、稳定性（编译期提前暴露bug）等方面上有着明显的好处：

- 加上了类型系统，对于阅读代码的人和编译器都是友好的。对阅读者来说，类型定义加上IDE的智能提示，增强了代码的易读型；对于编译器来说，类型定义可以让编译器揪出隐藏的bug。
- 类型系统+静态分析检查+智能感知/提示，使大规模的应用代码质量更高，运行时bug更少，更方便维护。
- 有类似VSCode这样配套的IDE支持，方便的查看类型推断和引用关系，可以更方便和安全的进行重构，再也不用全局搜索，一个个修改了。
- 给应用配置、应用状态、前后端接口及各种模块定义类型，整个应用都是一个个的类型定义，使协作更为方便、高效和安全。


## TS能干点什么

### 静态检查

> 这类问题是ESLint等工具检测不出来的。

#### 低级错误

```typescript
const peoples = [{
  name: 'tim',
  age: 20
}, {
  name: 'alex',
  age: 22
}];
const sortedPeoples = peoples.sort((a, b) => a.name.localCompare(b.name));
```

执行TS编译命令`tsc`，检测到错误：

```
error TS2339: Property 'localCompare' does not exist on type 'string'.
```

如果是在支持TS的IDE中（VS Code、WebStorm等），则不需等到编译，在IDE中就可以非常明显在localCompare位置提示出错误信息。

localCompare这种输入手误（或者手滑不小心删除或添加了字符）时有发生，如果没有编译器静态检查，那有可能就是一个字符引发的血案：埋下了一个`隐藏的`运行时bug。如果在SPA应用中，这个问题需要较长的操作路径才能被发现，一旦用户触发这个地雷，那它就会爆炸：应用直接crash（在没有页面刷新的SPA中问题尤为凸显）。

#### 非空判断

```typescript
let data = {
  list: null,
  success: true
};
const value = data.list.length;
```

执行`tsc`编译：

```
error TS2532: Object is possibly 'null'.
```

`data.list.length`这行直接引用了data.list的属性，但data.list的数据格式有不是数组的可能性，这种场景在前端处理后端接口返回时经常出现，接口返回的数据层级可能非常深，如果在某一级缺少了非空判断逻辑，那就意味着埋下了一个不知道什么时候就会引爆的炸弹。

#### 类型推断

```javascript
const arr = [];
arr.toUpperCase();

class Cat {
  miao() {}
}

class Dog {
  wang() {}
}
const cat = new Cat();
cat.wang();
```

执行`tsc`编译：

```
error TS2339: Property 'toUpperCase' does not exist on type 'any[]'.
error TS2339: Property 'wang' does not exist on type 'Cat'.
```

TS有类型推断，给不同类型的执行对象调用错误的方法都将被检查出来。

### 面向对象编程增强

#### 访问权限控制

```typescript
class Person {
  protected name: string;
  public age: number;
  constructor(name: string) { this.name = name; }
}

class Employee extends Person {
  static someAttr = 1;
  private department: string;

  constructor(name: string, department: string) {
    super(name);
    this.department = department;
  }
}
let howard = new Employee("Howard", "Sales");
console.log(howard.name);
```

执行`tsc`编译：

```
error TS2445: Property 'name' is protected and only accessible within class 'Person' and its subclasses.
```

Person中name属性是protected类型，只能在自己类中或者子类中使用。访问权限控制在面向对象编程中很有用，他能帮忙我们做到信息隐藏，JS面向对象编程的一个大问题就是没有提供原生支持信息隐藏的方案（很多时候都是通过约定编码方式来做）。信息隐藏有助于更好的管理系统的复杂度，这在软件工程中显得尤为重要。

#### 接口

```typescript
interface Machine {
  move(): void
}

interface Human {
  run(): void
}

class Base {
}

class Robot extends Base implements Machine, Human {
  run() {
    console.log('run');
  }
  move() {
    console.log('move');
  }
}
```

Robot类可以继承Base类，并实现Machine和Human接口，这种可以组合继承类和实现接口的方式使面向对象编程更为灵活、可扩展性更好。

#### 泛型

```typescript
class GenericNumber<T> {
    zeroValue: T;
    add: (x: T, y: T) => T;
}

let myGenericNumber = new GenericNumber<number>();
myGenericNumber.zeroValue = 0;
myGenericNumber.add = function(x, y) { return x + y; };
```

定义了一个模板类型T，实例化GenericNumber类时可以传入内置类型或者自定义类型。泛型（模板）在传统面向对象编程语言中是很常见的概念了，在代码逻辑是通用模式化的，参数可以是动态类型的场景下比较有用。

### 类型系统

```typescript
interface SystemConfig {
  attr1: number;
  attr2: string;
  func1(): string;
}

interface ModuleType {
  data: {
    attr1?: string,
    attr2?: number
  },
  visible: boolean
}

const config: SystemConfig = {
  attr1: 1,
  attr2: 'str',
  func1: () => ''
};

const mod: ModuleType = {
  data: {
    attr1: '1'
  },
  visible: true
};
```

我们定义了一个系统配置类型`SystemConfig`和一个模块类型`ModuleType`，我们在使用这些类型时就不能随便修改`config`和`mod`的数据了。**每个被调用方负责自己的对外类型展现，调用者只需关心被调用方的类型，不需关心内部细节**，这就是类型约束的好处，这对于多人协作的团队项目非常有帮助。

### 模块系统增强

```typescript
namespace N {
  export namespace NN {
    export function a() {
      console.log('N.a');
    }
  }
}

N.NN.a();
```

TS除了支持ES6的模块系统之外，还支持命名空间。这在管理复杂模块的内部时比较有用。

## 使用TS的成本

### 学习成本

理论上学习并应用一门新语言是需要很高成本的，但好在TS本身是JS的超集，这也意味着他本身是可以支持现有JS代码的，至少理论上是这样。学习一下类型系统的相关知识和面向对象的基础知识，应该可以hold住TS，成本不会很高。[官方文档](http://www.typescriptlang.org/index.html)是最好的学习材料。

### 应用成本
#### 老项目
对于老项目，由于TS兼容ES规范，所以可以比较方便的升级现有的JS（这里指ES6及以上）代码，逐渐的加类型注解，渐进式增强代码健壮性。迁移过程：
1. npm全局安装typescript包，并在工程根目录运行`tsc --init`，自动产生`tsconfig.json`文件。
默认的3个配置项：[更多配置项说明](https://www.typescriptlang.org/docs/handbook/compiler-options.html)
	- `"target":"es5"`: 编译后代码的ES版本，还有es3，es2105等选项。
	- `"module":"commonjs"`:编译后代码的模块化组织方式，还有amd，umd，es2015等选项。
	- `"strict":true`:严格校验，包含不能有没意义的any，null校验等选项。
2. 初始化得到的`tsconfig.json`无需修改，增加`"allowJs": true`选项。
3. 配置webpack配置，增加ts的loader，如awesome-typescript-loader。(如果是基于atool-build来构建的项目，则它内置了ts编译，这步省略)
	```json
	loaders: [
		// All files with a '.ts' or '.tsx' extension will be handled by 'awesome-typescript-loader'.
		{ test: /\.tsx?$/, loader: "awesome-typescript-loader" }
	]
	```

4. 此时你可以写文件名为ts和tsx（React）后缀的代码了，它可以和现有的ES6代码共存，VSCode会自动校验这部分代码，webpack打包也没问题了。
5. 逐渐的，开始打算重构以前的ES6代码为TS代码，只需将文件后缀改成ts(x)就行，就可以享受TS及IDE智能感知/纠错带来的好处。


更多迁移教程：[官方迁移教程](http://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html)、[官方React项目迁移教程](https://github.com/Microsoft/TypeScript-React-Conversion-Guide)、[社区教程1](https://basarat.gitbooks.io/typescript/content/docs/types/migrating.html)、[社区教程2](https://medium.com/@clayallsopp/incrementally-migrating-javascript-to-typescript-565020e49c88)。


#### 新项目
- 对于新项目，微软提供了非常棒的一些[Starter项目](http://www.typescriptlang.org/samples/index.html)，详细介绍了如何用TS和其他框架、库配合使用。如果是React项目，可以参考这个Starter：[TypeScript-React-Starter](https://github.com/Microsoft/TypeScript-React-Starter)

### 成本对比
> 星多表示占优

| 成本点 | ES | TS | 说明 |
| --- | --- | --- | --- |
| 学习和踩坑成本 | ※※※※※ | ※※※ | 虽然是JS超集，但还是要学习TS本身及面向对象基础知识，开发环境搭建、使用中的问题和坑也需要自己趟，好在TS社区比较成熟，网上沉淀的资料很多 |
| 整体代码量 | ※※※※※ | ※※※※ | TS代码增加比较完善的类型定义的话整体代码量比原生ES多5%~10%左右 |
| 原生JS（标准ES、浏览器端、服务器端）| ※※※ | ※※※※※ | IDE内置了详尽的类型声明，可以智能提示方法和参数说明，提升了效率 |
| 依赖外部库（React、Lodash、Antd） | ※※※ | ※※※※※ |  有TS类型声明库，IDE智能提示和分析，效率提升 |
| 内部公共库、模块 | ※※※ | ※※※※ | 团队内部自行编写类型定义文件，有一定工作量，但开发效率可以有一些提升，逐步完善类型定义后，效率进一步提升 |
| 团队协作效率 | ※※ | ※※※※※ | 对系统配置、外部接口、内部模块做类型定义后，实例对象属性就不能随意改了，每个被调用方负责自己的对外类型展现（可以理解为形状），调用者只需关心被调用方的类型，不需关心内部细节 |
| 代码可维护性 | ※※ | ※※※※ | 由于团队成员水平差异，和软件的**熵**的特质，长期迭代维护的项目总会遇到可维护性的问题，有了强类型约束和静态检查，以及智能IDE的帮助下，可以降低软件腐化的速度，提升可维护性，且在**重构**时，强类型和静态类型检查会帮上大忙，甚至有了类型定义，会不经意间增加重构的频率 |
| 运行时稳定性 | ※※ | ※※※※ | 由于TS有静态类型检查，很多bug都会被消灭在上线前 |


### 小结
从上面的对比中可以看到，使用大家都熟悉的ES作为开发语言只在学习和踩坑成本以及整体代码量上占优，如果只是短期项目，那用ES无可厚非，但我们的项目生命周期持续好几年，是持续迭代升级的，目前TS社区已经比较成熟，学习资料也很多，而且TS带来的是内部协作开发效率、可维护性、稳定性的提升，所以从长远来看这个代价是值得付出的。而且各种类型声明定义文件的存在，是可以提升开发效率的；而且静态类型检查可以减少bug数量和排查bug的难度，变相也提升了效率，而且使整个项目相对变得更为稳定可控。

## 社区发展

从Stackoverflow的[2017年开发者调查报告](https://insights.stackoverflow.com/survey/2017#technology-most-loved-dreaded-and-wanted-languages)、[Google趋势](https://trends.google.com/trends/explore?date=today%205-y&q=%2Fm%2F0n50hxv)、[npm下载量趋势](http://www.npmtrends.com/typescript)上可以到看，TypeScript社区发展很快，特别是最近几年。特别是伴随着VS Code的诞生（TS写的，对TS支持非常友好），VS Code + TypeScript的组合让前端圈产生了一股清流，生产力和规范性得到了快速提升。从Google对TS的支持（Angular高于2的版本是TS写的）看到，国际大厂也是支持的。

从蚂蚁集团内部看，Ant Design、Basement等产品也是基于TS写的（至少是在大量使用），虽然有一些反对的声音，但总体还是看好的，有合适的土壤就会快速发展，如Ant Design。

## 周边生态

### 类型声明包

React、及其他各种著名框架、库都有TS类型声明，我们可以在项目中通过`npm install @types/react`方式安装，可以在[这个网站](http://microsoft.github.io/TypeSearch/)搜索你想要安装的库声明包。安装后，写和那些框架、库相关的代码将会是一种非常爽的体验，函数的定义和注释将会自动提示出来，开发效率将会得到提升。

### IDE

VS Code、WebStorm等前端圈流行的IDE都对TS有着非常友好的支持，VS Code甚至自身就是TS写成的。


## 深入解读TS
### TS语言设计目标

#### 目标

编译期可以做静态检查，为大规模代码提供结构化装置，编译出符合习惯、易读的JS代码，和ECMAScript标准对齐，使用一贯的、可删除的、结构化的类型系统，保护编译后的JS代码的运行时行为等等。

#### 非目标

模仿现有语言，优化编译后代码的性能，应用“正确”的类型系统，增加运行时类型信息等等。

[TS设计目标原文](https://github.com/Microsoft/TypeScript/wiki/TypeScript-Design-Goals)

### TS简史

在最近几年，随着V8平台、各大现代浏览器的起来，JS的运行平台再不断完善，但，JS对于大型应用的开发是非常困难的，JS语言设计出来的目的不是为了大型应用，他是一门脚本语言，他没有静态类型校验，但更重要的是，他没有提供大型应用必须的classes、modules/namespaces、interfaces等结构化的装置，中间也出来过GWT等为了其他语言开发者开发大型JS应用的项目，这些项目可以让你利用Java等面向对象语言开发大型Web应用，也可以利用到Eclipse等好用的IDE，但这些项目不是用JS写代码，所以如果你想用JS里的一些东西，你可能要比较费劲的在其他语言里把它给实现出来，所以我们考虑如何**增强**JS语言，提供如静态类型检查、classes、modules/namespaces、interfaces等大型应用装置，这就是TS语言：**TS是一种开发大型JS应用的语言，更详细一点来说，TS是有类型的编译到纯JS的JS超集。**所以一般来说，JS代码也是TS代码。本身TS编译器也是TS写的，运行Node.js环境。【[Anders Hejlsberg: Introducing TypeScript 2012](https://channel9.msdn.com/posts/Anders-Hejlsberg-Introducing-TypeScript)】


TS作者在最近微软Build大会给出的一个图：

![JS feature间隔](https://user-gold-cdn.xitu.io/2017/9/22/693bf5a67c81cbe007fc83311f29571f)

如图，Web和Node平台的JS始终与JS最新规范有一段距离，Web平台的距离更远，TS可以填充这个间隙，让使用者在Web和Node平台都能用上最新的Feature，用上优雅的JS，提高生产力。【[Anders Hejlsberg: What's new in TypeScript? 2017](https://channel9.msdn.com/Events/Build/2017/B8088?ocid=player)】

### 和Flow 、Babel的对比

####  vs Flow

从[这篇文章](https://github.com/niieani/typescript-vs-flowtype)可以看出，基础的类型检查功能发展到现在已经差别不大了，但在周边生态、文档完整性、社区资源方面TS胜过Flow。

#### vs Babel

Babel也是很不错的ES6 to 5编译工具，有不错的插件机制，社区发展也不错，但在同样一段代码编译出的JS代码里可以看到，TS编译后的代码是更符合习惯、简洁易读一些（都用的是官方网站的Playground工具）。我曾经维护过TS编译后的JS代码（TS源码丢失），感觉还OK。

Babel编译后：
![Babel编译](https://user-gold-cdn.xitu.io/2017/9/22/755936c632e1ef5290fac2bf3201d70b)

TS编译后：
![TS编译](https://user-gold-cdn.xitu.io/2017/9/22/68a505993139e80d7d33bf2b7244ed16)

### TS基础知识及核心概念

#### 基础类型

```typescript
let isDone: boolean = false;

let decimal: number = 6;

let color: string = "blue";

// 数组，有两种写法
let list: number[] = [1, 2, 3];
let list: Array<number> = [1, 2, 3];

// 元组(Tuple)
let x: [string, number] = ["hello", 10];

// 枚举
enum Color {Red = 1, Green = 2, Blue = 4}
let c: Color = Color.Green;

// 不确定的可以先声明为any
let notSure: any = 4;

// 声明没有返回值
function warnUser(): void {
    alert("This is my warning message");
}

let u: undefined = undefined;

let n: null = null;

// 类型永远没返回
function error(message: string): never {
    throw new Error(message);
}

// 类型主张，就是知道的比编译器多，主动告诉编译器更多信息，有两种写法
let someValue: any = "this is a string";
let strLength: number = (<string>someValue).length;
let strLength: number = (someValue as string).length;
```

更多介绍可以直接看[官方文档](基础知识可以直接看[官方文档](https://www.typescriptlang.org)，下面介绍下他不同于ES的几个核心概念：)。

#### interface

TS中的一个核心原则之一就是类型检查关注的是值的形状，有时就叫做“鸭子辨型”（duck typing）或“结构化子类型”（structural subtyping）。TS中interface就承担了这样的角色，定义形状与约束，在内部使用或者和外部系统协作。一个例子：

```typescript
interface SystemConfig {
  	attr1: string;
  	attr2: number;
    func1(): string;
    func2(): void;
}
```

我们给软件定了一个系统参数的配置接口，他定义了系统配置的形状，有两个属性`attr1`和`attr2`，两个方法`func1`和`func2`，这样如果定义了`const systemConfig: SystemConfig = {} `,那systemConfig就不能随意修改了，他有形状了。

 在Java中，我们提倡面向接口编程，接口优于抽象类【Effective Java】，在TS的类系统中，接口也可以承担这样的角色，我们可以用`implements`来实现接口，这样可以实现类似更为灵活的继承，如：

```typescript
class A extends BaseClass implements BaseInterface1, BaseInterface2 {}
```

类A继承了BaseClass，并且继承了BaseInterface1和BaseInterface2两个接口。

#### module/namespace

导出到外部的模块写法和ES6一样，内部模块现在推荐用namespace，如：

```typescript
namespace Module1 {
  export interface SubModule1 {}

  export interface SubModule2 {}
}
const module: Module1.SubModule = {}
```

命名空间在JS中用对象字面量也可以实现，早些年的很多JS库都是这种模式，但显然有了这种显示的命名空间声明，代码的易读性更好，且不能随意的改变，不像用原生JS对象时容易被覆盖。

#### 访问权限控制

TS有着和传统面向对象语言类似的public、protected、private等访问权限，这个在大型应用中很实用，这里不展开。

#### 其他

除了泛型相对难以掌握，其他class、decorator、async/await等都和ES6、ES7写法类似。

### Type和Type System

TypeScript = Type + Script，那么编程语言中的Type是怎么定义的呢？

> In [computer science](https://en.wikipedia.org/wiki/Computer_science) and [computer programming](https://en.wikipedia.org/wiki/Computer_programming), a **data type** or simply **type** is a classification of data which tells the compiler or interpreter how the programmer intends to use the data.【[维基百科](https://en.wikipedia.org/wiki/Data_type)】

在计算机科学中，数据类型或者简单说类型，是数据的类别，用来告诉编译器/解释器程序员想怎么使用数据。基本的数据类型如整数、布尔值、字符等，组合数据类型如数组、对象等，也有抽象数据类型如队列、栈、集合、字典等等。数据类型用在类型系统中，类型系统提供了类型定义、实现和使用的方式，每种编程语言都有各自的类型系统实现（如果有的话）。

我们来看看Type System（类型系统）的定义：

> In [programming languages](https://en.wikipedia.org/wiki/Programming_language), a **type system** is a set of rules that assigns a property called [type](https://en.wikipedia.org/wiki/Type_(computer_science)) to the various constructs of a [computer program](https://en.wikipedia.org/wiki/Computer_program), such as [variables](https://en.wikipedia.org/wiki/Variable_(computer_science)), [expressions](https://en.wikipedia.org/wiki/Expression_(computer_science)), [functions](https://en.wikipedia.org/wiki/Function_(computer_science)) or [modules](https://en.wikipedia.org/wiki/Modular_programming).[[1\]](https://en.wikipedia.org/wiki/Type_system#cite_note-FOOTNOTEPierce20021-1) These types formalize and enforce the (otherwise implicit) categories the programmer uses for data structures and components (ex: "string", "array of float", "function returning boolean"). The main purpose of a type system is to reduce possibilities for [bugs](https://en.wikipedia.org/wiki/Bug_(computer_programming)) in computer programs[[2\]](https://en.wikipedia.org/wiki/Type_system#cite_note-FOOTNOTECardelli20041-2) by defining interfaces between different parts of a computer program, and then checking that the parts have been connected in a consistent way. 【[维基百科](https://en.wikipedia.org/wiki/Type_system)】

在编程语言中，类型系统是一个规则集合，给程序中的变量、表达式、函数、模块等程序构建元素分配叫做类型的属性。这些类型明确并强制（也可能是含蓄的）程序员如何使用数据结构。**类型系统的主要目的是通过定义程序不同部分间协作的接口，并检查不同部分以始终如一的方式协作，来减少程序可能产生的bug。**这种检查可能是静态的（编译期）或动态的（运行时），或者既有静态的也有动态的。

#### 类型系统的好处

##### 检测错误

> The most obvious benefit of static typechecking is that it allows early detection of some programming errors. Errors that are detected early can be fixed immediately, rather than lurking in the code to be discovered much later,when the programmer is in the middle of something else—or even after the program has been deployed. Moreover, errors can often be pinpointed more accurately during typechecking than at run time, when their effects may not become visible until some time after things begin to go wrong.【[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)】

> As your app grows, you can catch a lot of bugs with typechecking. 【[React typechecking](https://facebook.github.io/react/docs/typechecking-with-proptypes.html)】

静态类型检查最明显的好处是可以尽早的检查出程序中的错误。错误被尽早的检查出来可以使它得到快速的修复，而不是潜伏在代码中，在中期甚至部署上线后才被发现。而且，错误在编译期可以被更精确的定位出来，而在运行时，错误产生的影响在程序出现问题前可能是不容易被发现的。

程序会有各种各样的数据结构，如果改了一个数据类型，前端很多时候都是通过全局查找来处理这种重构问题的。而静态类型检查则可以使再次编译后就能探知所有可能的错误，加上IDE的智能错误提示，重构起来更放心、更方便。

##### 抽象化

> Another important way in which type systems support the programming process is by enforcing disciplined programming. In particular, in the context of large-scale software composition, type systems form the backbone of the module languages used to package and tie together the components of large systems. Types show up in the interfaces of modules (and related structures such as classes); indeed, an interface itself can be viewed as “the type of a module,” providing a summary of the facilities provided by the module—a kind of partial contract between implementors and users.
>
> Structuring large systems in terms of modules with clear interfaces leads to a more abstract style of design, where interfaces are designed and discussed independently from their eventual implementations. More abstract thinking about interfaces generally leads to better design.【[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)】

类型系统支持编程阶段的另外一个重要方式是强制让编程遵守纪律。在大规模软件系统中，类型系统组成了组件协作系统的脊梁。类型展现在模块（或者相关的结构如类）的接口中。接口可以看做“模块的类型”，展示了模块所能提供的功能，是一种实现者和用户间的合约。

在大量模块协作组成的大规模结构化软件系统中清晰的接口可以使设计更为抽象，接口的设计和讨论独立于它们的实现。一般来说，对接口更为抽象的思考可以使得做出更好的设计。

> Types enable programmers to think at a higher level than the bit or byte, not bothering with low-level implementation. For example, programmers can begin to think of a string as a set of character values instead of as a mere array of bytes. Higher still, types enable programmers to think about and express [interfaces](https://en.wikipedia.org/wiki/Interface_(computer_science)) between two of *any*-sized subsystems. This enables more levels of localization so that the definitions required for interoperability of the subsystems remain consistent when those two subsystems communicate.【[维基百科](https://en.wikipedia.org/wiki/Type_system)】

类型会让程序员在一个更高的维度思考，而不是在底层的计算机实现细节纠缠。例如，我们可以把字符串想成字符集，而不仅仅是比特数组。更高维度，类型系统可以让我们用接口来思考和表达任意子系统/子程序之间的协作，接口定义可以让子系统/子程序之间的通信方式始终如一。

##### 文档化

> Types are also useful when reading programs. The type declarations in procedure headers and module interfaces constitute a form of documentation,giving useful hints about behavior. Moreover, unlike descriptions embedded in comments, this form of documentation cannot become outdated, since it
> is checked during every run of the compiler. This role of types is particularly important in module signatures.【[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)】

类型对于阅读程序也是有用的。在程序头部的类型声明和模块接口形成了文档的形状，提供程序的行为提示。此外，不同于在注释中的描述，这种形式的文档不会过期，因为每次编译都会校验，这在模块签名里特别重要。

> In more expressive type systems, types can serve as a form of [documentation](https://en.wikipedia.org/wiki/Documentation) clarifying the intent of the programmer. For example, if a programmer declares a function as returning a timestamp type, this documents the function when the timestamp type can be explicitly declared deeper in the code to be an integer type.【[维基百科](https://en.wikipedia.org/wiki/Type_system)】

在复用表现力的类型系统中，类型可以看做是一种描述程序员意图的表述方式。例如，我们声明一个函数返回一个时间戳，这样就相当于明确说明了这个函数在更深层次的代码调用中会返回整数类型。

##### 语言安全

> The term “safe language” is, unfortunately, even more contentious than “type system.” Although people generally feel they know one when they see it, their notions of exactly what constitutes language safety are strongly influenced by the language community to which they belong. Informally, though, safe
> languages can be defined as ones that make it impossible to shoot yourself in the foot while programming.【[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)】

安全语言这个说法是有争议的。受到该语言社区的严重影响。不正式的来说，安全语言可以被定义为在编程时不可能从底层把自己杀死。

> A type system enables the [compiler](https://en.wikipedia.org/wiki/Compiler) to detect meaningless or probably invalid code. For example, we can identify an expression `3 / "Hello, World"` as invalid, when the rules do not specify how to divide an [integer](https://en.wikipedia.org/wiki/Integer) by a [string](https://en.wikipedia.org/wiki/String_(computer_science)). Strong typing offers more safety, but cannot guarantee complete *type safety*.【[维基百科](https://en.wikipedia.org/wiki/Type_system)】

类型系统会允许编译器检查无意义或者可能不合法的代码。例如，我们知道`3/'hello world'`不合法，强类型提供了更多的安全性，但也不能完全做到类型安全。

##### 效率

> Static type-checking may provide useful compile-time information. For example, if a type requires that a value must align in memory at a multiple of four bytes, the compiler may be able to use more efficient machine instructions.【[维基百科](https://en.wikipedia.org/wiki/Type_system)】

静态类型检查会提供有用的编译期信息。例如，如果一个类型需要在内存中占四个字节，编译器可能会使用更有效率的机器指令。

### 静态类型、动态类型和弱类型、强类型
- 静态类型：编译期就知道每一个变量的类型。类型错误编译失败是语法问题。如Java、C++。
- 动态类型：编译期不知道类型，运行时才知道。类型错误抛出异常发生在运行时。如JS、Python。
- 弱类型：容忍隐式类型转换。如JS，`1+'1'='11'`，数字型转成了字符型。
- 强类型：不容忍隐式类型转换。如Python，`1+'1'`会抛出`TypeError`。


## 接受TS

TS刚出来时我是有点抵触的，或者对她的感觉就跟和`CoffeeScript`、`Dart`等编译到JS语言差不多，感觉就是其他语言往JS渗透的产物，近一两年，社区中TS的声音越来越强，而我也开始做大型JavaScript应用，随之逐渐重新认识TS，逐渐认识到TS的类型系统、TSC的静态检查、VS Code等IDE的强力支持对于开发出可维护性好、稳定性高的大型JavaScript应用的重要性。


## 权衡

如何更好的利用JS的动态性和TS的静态特质，我们需要结合项目的实际情况来进行综合判断。一些建议：

- 如果是中小型项目，且生命周期不是很长，那就直接用JS吧，不要被TS束缚住了手脚。
- 如果是大型应用，且生命周期比较长，那建议试试TS。开源项目如[VS Code](https://github.com/Microsoft/vscode)、[GitHub桌面端](https://github.com/desktop/desktop)，不开源的如[Slack桌面端]([https://slack.engineering/typescript-at-slack-a81307fa288d](https://slack.engineering/typescript-at-slack-a81307fa288d))、[Asana]([https://blog.asana.com/2014/11/asana-switching-typescript/](https://blog.asana.com/2014/11/asana-switching-typescript/))、[Palantir](https://medium.com/@clayallsopp/incrementally-migrating-javascript-to-typescript-565020e49c88)。
- 如果是框架、库之类的公共模块，那更建议用TS了。如[Ant Design](https://github.com/ant-design/ant-design)、[Angular](https://github.com/angular/angular) 、[Ionic](https://github.com/ionic-team/ionic)。

**至于到底用不用TS，还是要看实际项目规模、项目生命周期、团队规模、团队成员情况等实际情况综合考虑。**