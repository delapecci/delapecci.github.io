---
title: "Node.js构造真正的全局单例"
date: 2018-04-24T12:36:09+08:00
# lastmod: 2017-08-23T18:03:09+08:00
draft: false
tags: ["Node.js", "技术干货", "翻译"]
categories: ["技术", "翻译"]
author: '李冲'

weight: 10

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
# comment: false
# toc: false
autoCollapseToc: true
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
# contentCopyright: '<a href="https://hackernoon.com/tokenize-all-the-things-and-what-happens-next-513d78add55" rel="noopener" target="_blank">原文链接</a>'
reward: false
mathjax: false
---

> 翻译 [Creating A True Singleton In Node.js, With ES6 Symbols](https://derickbailey.com/2016/03/09/creating-a-true-singleton-in-node-js-with-es6-symbols/)

Node.js 通常使用 module.exports 来暴露一个对象实例，并且允许其他 js 文件引用这个相同的对象实例。很多人把这个叫做 Node.js 的单例模式，但是其实这并不是真正的单例。

<!--more-->

![](http://derickbailey.com/wp-content/uploads/2016/03/singleton.jpeg)

不，这只是个被缓存了的对象实例——不能保证在所有文件里可以被唯一引用。

## [Require]的问题

当你在 Node 里面使用 require 函数的时候，它会使用被 require 的文件路径作为缓存键。如果你在不同的多个文件里 require 同一个文件，通常你都会得到这个 module 的相同的一个缓存副本。

这对于保持内存甚至生成一个单例的弱拷贝而言很不错。但是，require 所提供的缓存对象功能可以很容易地被击破！在这两种场景下，这种方式就没用了：

* 碰巧写错了大小写
* 其他 module 依赖到相同的 module

## 大小写敏感

Windows 和 macOS(默认情况下)的文件系统是文件名大小写不敏感的。你可以使用"FOO.js"或者"foo.js"，在同样的目录下找到同样的文件，不管搜索的文件名关键字和实际的文件名是否大小写一致。但是，这样的方式在 Windows 或 macOS 上是不能得到不同的结果的——foo.js 和 FOO.js，Windows 和 macOS 不吊我们。

然而 Node.js 是不知道"foo.js"和"FOO.js"是同一个文件的，因为 Node*是*对文件名大小写敏感的，它会认为是两个不同的 module。因此在 Windows 和 macOS 上的缓存魔法很容易就被拆穿了。上代码试试看。创建一个"foo.js"文件。

```js
module.exports = {
  foo: "一个值"
};
```

在另一个文件 index.js 里 require 两次

```js
var foo = require("./foo");
var FOO = require("./FOO");

FOO.foo = "另一个值";

console.log("the foo object");
console.log(JSON.stringify(foo, null, 2));

console.log("");

console.log("the FOO object");
console.log(JSON.stringify(FOO, null, 2));
```

运行这个文件，看看结果

```shell
$ node index.js
the foo object
{
  "foo": "一个值"
}

the FOO object
{
  "foo": "另一个值"
}
```

在这个例子里，大小写不同的文件名是 require 的缓存键的一部分，因此得到了不同的缓存对象，虽然操作系统返回的是同一个 js 文件。因为 require 了两次，那么缓存过程也执行了两次，每一次都生成了一个对象实例。同时这样的做法还会带来其他问题。如果在对文件名称大小写敏感的操作系统里运行这个例子，会得到不同的结果，因为大小写不同的文件可能不存在。诚然如果时刻保证引用文件的大小写拼写正确就可以规避这个问题，但是这很难完全做到。因此一旦出现这样的笔误，在文件名大小写不敏感的系统上就会产生意料之外的结果——错误的 module 缓存。

## NPM module 的传递依赖

还有另一种情况会使 module 缓存失效，那就是当你从 npm 安装了两个或更多依赖相同 module 的 module 时。也就是说，如果你的 project 依赖 Foo 和 Bar 两个 module，而这两个 module 又同时都依赖于 Quux 这个 module，NPM(2.0 或更低版本)会为每个依赖 Quux 的 module 安装一个 Quux 的副本。😳
虽然我知道 NPM 3.0 以来通过平铺依赖关系列表来尝试解决这个问题，即 Foo 和 Bar 同时依赖相同(或兼容)版本的 Quux 时，只会有一个 Quux 的副本被安装。然而，如果 Foo 和 Bar 真的需要依赖不同或不能相互兼容版本的 Quux 时，NPM 依然会安装两个 Quux 副本，Foo 和 Bar 无法共享相同的缓存实例了。

## 获得缓存破坏免疫

既然调用 require 无法获得真正的单例对象，而是一个缓存的副本，那该怎样才能创建一个真正的单例呢？很不幸，答案虽有但是并不推荐甚至应该避免使用的，那就是使用 global 变量。
Node 确实允许 module 里定义并暴露 global 变量，尽管它在尽量禁止这样做。参考下面例子 global.js 文件，需要使用*global*关键字：

```js
global.foo = {
  foo: "全局变量值"
};
```

现在可以在其他文件的任何地方通过引入这个 module 来时用 global.foo 这个变量了，参考下面的 index.js 文件：

```js
require("./global");

console.log(JSON.stringify(foo, null, 2));
```

输出看起来和原始值一样，但这种方式并没有真正拿到 global module 的引用。

```shell
$ node index.js
{
  "foo": "全局变量值"
}
```

尽管可能会造成危险的后果，JSHint 也会给你定义的 foo 变量一个红色的 ✘，但是这种技巧确实可以在 Node 里面创建一个真正的单例。如果想要采用稍微安全一点的做法，可以使用 ES6 里面的 symbol。

## 真正单例的核心

采用 global 变量是危险的([各种原因，你懂得](http://wiki.c2.com/?GlobalVariablesAreBad))，几乎整个 JavaScript 社区都开始改用 module 来避免各种因为 global 变量导致的问题。但是随着[ES6 Symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)的出现，使得在不破坏应用程序的完整性的前提下使用 global 变量成为可能。参考下面的 singleton_1.js 代码：

```js
const FOO_KEY = Symbol("foo");

global[FOO_KEY] = {
  foo: "Symbol指向的全局变量值"
};
```

使用 Symbol，可以相对安全的向 global 对象里添加变量。但是这只是半吊子的单例实现方案，后面会详述。在讲述终极方案之前，一个单例应该提供一个特定的 API 来匹配这种设计模式——定义一个 instance 属性来让外部获取这个单例对象。使用[Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)和[Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)，可以很简单地为单例 API 添加一个安全保证，参考 singleton_2.js 代码：

```js
// define the actual singleton instance
// ------------------------------------

const FOO_KEY = Symbol("foo");

global[FOO_KEY] = {
  foo: "Symbol指向的全局变量值"
};

// 定义一个单例包装对象
// ------------------------

var singleton = {};

Object.defineProperty(singleton, "instance", {
  get: function() {
    return global[FOO_KEY];
  }
});

// 锁定单例包装对象API
// -------------------------------

Object.freeze(singleton);

// 只暴露单例包装对象而不是global symbol
// -----------------------------

module.exports = singleton;
```

之后多次 require 相同的这个文件，都可以获得唯一的一个 singleton 对象。但是这样还是不能搞定之前提到的文件名大小写不敏感时的文件加载问题，以及 NPM 传递依赖的问题，修改 index.js：

```js
var foo = require("./singleton_2").instance;
var FOO = require("./SINGLETON_2").instance;
```

```shell
$ node index.js
the foo object
{
  "foo": "Symbol指向的全局变量值"
}

the FOO object
{
  "foo": "另一个值"
}
```

我擦，和前面的问题输出一样，所以这只是个半吊子。让我们修复它。

## 创建一个真正的单例

最后一块拼图是确保这个 js 文件被加载的任何版本都不会覆盖 global symbol。不幸的是现在这种 global symbol 的写法，当文件每次被加载都会生成一个新的 symbol 实例 😓
要修复它，你得使用 global symbol 的缓存；此外，还需要检查 global 对象上是否已经有 symbol 的值；最后，需要给这个 symbol 一个唯一的名字，因为你很可能正在把这个 symbol 暴露给其他开发者。参考下面 ture_singleton.js 代码：

```js
// create a unique, global symbol name
// -----------------------------------

const FOO_KEY = Symbol.for("My.App.Namespace.foo");

// check if the global object has this symbol
// add it if it does not have the symbol, yet
// ------------------------------------------

var globalSymbols = Object.getOwnPropertySymbols(global);
var hasFoo = globalSymbols.indexOf(FOO_KEY) > -1;

if (!hasFoo) {
  global[FOO_KEY] = {
    foo: "bar"
  };
}

// define the singleton API
// ------------------------

var singleton = {};

Object.defineProperty(singleton, "instance", {
  get: function() {
    return global[FOO_KEY];
  }
});

// ensure the API is never changed
// -------------------------------

Object.freeze(singleton);

// export the singleton API only
// -----------------------------

module.exports = singleton;
```

这段代码里通过[Symbol.for](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for)来获取一个全局共享的唯一名称，接着使用 Object.getOwnPropertySymbols(global)来[获取所有 global symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols)来检查将要声明的 symbol 是否已经存在，如果已经存在不进行覆盖。再次调用上面修改后的 index.js：

```shell
$ node index.js
the foo object
{
  "foo": "另一个值"
}

the FOO object
{
  "foo": "另一个值"
}
```

## 终于有了真正的单例，但是值得吗...

使用 ES6 Symbol，你现在得到了一个相对安全的真正的单例，可以在任何地方多次引用它，即便碰巧小错了文件名大小写，甚至从 NPM 那里安装了相同依赖的多个不同版本，都可以得到一个唯一的单例对象。这种方案虽然可以满足需求，但是仍然存在潜在的风险。例如前面提到的 NPM 安装了相同依赖的不同版本问题，在这种方案下，不同版本将不复存在。第一个被加载的将成为唯一单例，这将会带来一些意想不到的后果，使得依赖的使用变得不可预测。或许你可以通过添加版本号给单例来规避这个问题，不过这就有变成半吊子的单例了......
而且，这种方式得到的单例并不真正安全。采用 global symbol 的方法，有可能其他人会把相同的 global symbol 用在不同的地方起不同作用。这种情况不一定但总有可能——其实任何时候一个问题都会进入这样一个吊诡的逻辑死循环”不一定，但总有可能“，这在某些人某些地方那里一定会是个真正的问题。

## 这两种方案的代价与收益分析

在 Node.js 里调用 require 会得到一个缓存的实例，当 require 的调用子句和 module 是兼容版本的情况下，这个实例是一个单例的弱拷贝。另一种方案是使用 ES6 symbol 和 global 变量方式可以得到真正的单例，但是也存在其自身的问题。

经验谈，采用第一种方式在绝大多数情况下足够满足需求了，只需要仔细查看自己的大小写。

如果你确信你非得要一种真正的单例实现方案来让你的单例可以随意使用，可以使用第二种。必须提醒一下，在你选择走这条路前，一定要知道当一个方案满足你的需要的同时，它也可能存在自身的风险。
