# What's new in ECMAScript 2020

## 回顾

ES6 的含义是 5.1 版以后的 JavaScript 的下一代标准，涵盖了 ES2015、ES2016、ES2017 等等。因此网络文章中常常出现的 ES7 ES8 ES9..都是不准确的叫法

任何人都可以向标准委员会（又称 TC39 委员会）提案，要求修改语言标准。

一个提案的生命周期：

- Stage 0 - Strawman（展示阶段）
- Stage 1 - Proposal（征求意见阶段）
- Stage 2 - Draft（草案阶段）
- Stage 3 - Candidate（候选人阶段）
- Stage 4 - Finished（定案阶段）

可在 TC39（标准制定委员会）的 [github](https://github.com/tc39/ecma262) 上看到所有提案以及状态

## 正题

今年的 ES2020 标准在 4月 2 号正式定稿，接下里一起看看我们又要学习什么东西了

### String.prototype.matchAll

#### 使用方式

```js
const matchIterator = str.matchAll(regExp);
```

`String#matchAll` 返回的是一个 Iterator(迭代器)，选择 Iterator 的原因是如果正则中包含大量的捕获组或者是超长文本，总是将数据庞大的匹配结果生成数组会产生性能影响

#### 为什么会有 matchAll

如果有一个字符串和一个带有 sticky 或者 global 的正则表达式，其中有多个捕获组(capturing groups)，如果我想迭代所有的匹配，目前，我的方案有以下几种

第一种：

```js
var regex = /t(e)(st(\d?))/g;
var string = 'test1test2';

string.match(regex);
```

缺陷：返回的值是 `['test1', 'test2']`，并没有我想要的捕获组

第二种：

```js
var matches = [];
var lastIndexes = {};
var match;
lastIndexes[regex.lastIndex] = true;
while (match = regex.exec(string)) {
	lastIndexes[regex.lastIndex] = true;
	matches.push(match);
	// example: ['test1', 'e', 'st1', '1'] with properties `index` and `input`
}
matches; /* gives exactly what i want, but uses a loop,
		* and mutates the regex's `lastIndex` property */
lastIndexes; /* ideally should give { 0: true } but instead
		* will have a value for each mutation of lastIndex */
```

缺陷：为了得到`matches`  使用了 while 循环；循环过程中会改变正则的 `lastIndex` 属性；最终的`lastIndexes`在每次循环时都被添加了新属性

第三种：

```js
var matches = [];
string.replace(regex, function () {
	var match = Array.prototype.slice.call(arguments, 0, -2);
	match.input = arguments[arguments.length - 1];
	match.index = arguments[arguments.length - 2];
	matches.push(match);
	// example: ['test1', 'e', 'st1', '1'] with properties `index` and `input`
});
matches; /* gives exactly what i want, but abuses `replace`,
	  * mutates the regex's `lastIndex` property,
	  * and requires manual construction of `match` */
```

缺陷：滥用 replace；改变了正则的`lastIndex`属性；需要单独构造 `match`变量中的` input` 和 `index` 属性

因此，`String#matchAll` 可以解决这个以上的缺陷，它既提供了返回捕获组，又不会对正则表达式的`lastIndex`属性进行改变

注：在 js 的已有的正则方法中，除了`String#match`，剩下的  `String#replace()`、 `String#split()`、 `RegExp#exec()` `RegExp#test()` 都会改变 lastIndex 属性

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-string-matchall)
2. [how String#matchAll works](https://2ality.com/2018/02/string-prototype-matchall.html)

#### 额外阅读

1. [讨论 String#replaceAll 与 String#matchAll 是否在未加 /g 时抛出异常](https://github.com/tc39/proposal-string-replaceall/issues/16)

### import()
#### 进一步阅读

1. [how import() works](https://exploringjs.com/impatient-js/ch_modules.html#loading-modules-dynamically-via-import)

### import.meta

### BigInt – arbitrary precision integers

### Promise.allSettled

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-promise-allSettled)
2. https://2ality.com/2019/08/promise-combinators.html

### globalThis 

### Optional chaining 

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-optional-chaining)
2.  [Optional chaining](https://2ality.com/2019/07/optional-chaining.html)

### Nullish coalescing Operator 

#### 进一步阅读

1. [babel plugin 进一步解释](https://babeljs.io/docs/en/babel-plugin-proposal-nullish-coalescing-operator)
2. [Nullish coalescing Operator ](https://2ality.com/2019/08/nullish-coalescing.html)

### export * as ns from "mod"



## 参考

1. 阮一峰 [ECMAScript 6 入门](https://es6.ruanyifeng.com/) 

