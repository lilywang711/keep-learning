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

可在 TC39（标准制定委员会）的 [Github](https://github.com/tc39/ecma262) 上查阅所有提案以及提案的状态

## 正题

今年的 ES2020 标准在 4月 2 号正式定稿，接下里一起看看我们又要学习什么东西吧 (#^.^#)

### String.prototype.matchAll

#### 语法

```js
const matchIterator = str.matchAll(regExp);
```

```js
[...'-a-a-a'.matchAll(/-(a)/ug)] // output: [ [ '-a', 'a' ], [ '-a', 'a' ], [ '-a', 'a' ] ]

// 封装成仅返回捕获组的函数
function collectGroup1(regExp, str) {
  let arr = [...str.matchAll(regExp)];
  return arr.map(x => x[1]);
} // output: ['a', 'a', 'a']

// 更精简
function collectGroup1(regExp, str) {
  return Array.from(str.matchAll(regExp), x => x[1]); // output: ['a', 'a', 'a']
}
```

`String#matchAll` 返回的是一个 Iterator(迭代器)，选择 Iterator 的原因是如果正则中包含大量的捕获组或者是超长文本，总是将数据庞大的匹配结果生成数组会产生性能影响

注意：传递给 matchAll 的 regExp 必须含有`/g` 标识符，否则将会报错。这个决定在此 [Issue](https://github.com/tc39/proposal-string-replaceall/issues/16) 中有大量讨论。

#### 为什么会有 matchAll

试想以下场景：

如果有一个字符串和一个带有 [sticky](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky) 或者 global 的正则表达式，其中有多个捕获组(capturing groups)，如果我想迭代所有的匹配，目前，我的方案有以下几种

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

缺陷：为了得到`matches`  使用了 while 循环；循环过程中会改变正则的 [`lastIndex`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/RegExp/lastIndex) 属性；最终的`lastIndexes`在每次循环时都被添加了新属性

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

注：在 js 的已有的正则方法中，除了`String#match`，  `String#replace()`、 `String#split()`、 `RegExp#exec()` `RegExp#test()` 都会改变 lastIndex 属性

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-string-matchall)
2. [How String#matchAll works](https://2ality.com/2018/02/string-prototype-matchall.html)

### dynamic-import: import()

#### 语法

有以下文件

```js
lib/my-math.mjs
main1.mjs
main2.mjs
```

其中`my-math.mjs`：

```js
// Not exported, private to module
function times(a, b) {
  return a * b;
}
export function square(x) {
  return times(x, x);
}
export const LIGHTSPEED = 299792458;
```

在 `main1.mjs` 中这样使用 `import()`

```js
const dir = './lib/';
const moduleSpecifier = dir + 'my-math.mjs';

function loadConstant() {
  return import(moduleSpecifier)
  .then(myMath => {
    const result = myMath.LIGHTSPEED;
    assert.equal(result, 299792458);
    return result;
  });
}
```

以下两点在这个特性未出之前是无法实现的：

- 在函数内部 import 文件
- 模块资源标识符来自于变量

由于 `import()`"返回"的是一个 promise，那么我们可以用 async await 这种更简洁的方式来改写以上代码

```js
const dir = './lib/';
const moduleSpecifier = dir + 'my-math.mjs';

async function loadConstant() {
  const myMath = await import(moduleSpecifier);
  const result = myMath.LIGHTSPEED;
  assert.equal(result, 299792458);
  return result;
}
```

甚至可以通过 Promise.all() Promise.race() 等 api 来实现特定的加载需求

注意：尽管它的工作原理很像一个函数，但 import() 是一个**operator**(操作符)

#### 使用场景

1. 按需加载，一些 function 不再需要在程序启动时就被加载，而是在逻辑需要时才去主动导入

   ```js
   button.addEventListener('click', event => {
     import('./dialogBox.mjs')
       .then(dialogBox => {
         dialogBox.open();
       })
       .catch(error => {
         /* Error handling */
       })
   });
   ```

2. 根据条件判断决定是否加载

   ```js
   if (isLegacyPlatform()) {
     import('./my-polyfill.mjs')
       .then(···);
   }
   ```

3. 动态计算模块资源标识符，国际化获取不同的语言包

   ```js
   import(`messages_${getLocale()}.mjs`)
     .then(···);
   ```

#### 进一步阅读


1. [Proposal and Specs](https://github.com/tc39/proposal-dynamic-import)
2. [How import() works](https://exploringjs.com/impatient-js/ch_modules.html#loading-modules-dynamically-via-import)

### import.meta

以下解释摘自 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/import.meta)

`import.meta` 是一个给 JavaScript 模块暴露特定上下文的元数据属性的对象。它包含了这个模块的信息，比如说这个模块的 URL

`import.meta` 对象由一个关键字 `"import"`, 一个点符号和一个 `meta` 属性名组成。通常情况下 `"import."` 是作为一个属性访问的上下文，但是在这里 `"import"` 不是一个真正的对象

`import.meta` 对象是由 ECMAScript 实现的，它带有一个 [`null`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/null)的原型对象。这个对象可以扩展，并且它的属性都是可写，可配置和可枚举的。

##### 例子

```html
< -- index.html -->
<script type="module" src="./index.js"></script>
```


```js
// index.js
console.log(import.meta) // { url: '/absolute/path/index.js' }
```


#### 进一步阅读

[Proposal and Specs](https://github.com/tc39/proposal-import-meta)

### BigInt – 新的基本数据类型

 JavaScript 中第 8 种基本数据类型 [Boolean, Null, Undefined, Number, BigInt, String, Symbol]

关于 `BigInt` 这里有一篇解释更详细的文章 [传送门](https://juejin.im/post/5d3f8402f265da039e129574)

### Promise.allSettled

#### 语法

```js
const promises = [ fetch('index.html'), fetch('https://does-not-exist/') ];
const results = await Promise.allSettled(promises);
const successfulPromises = results.filter(p => p.status === 'fulfilled');
```

Promise.allSettled 接受一个 [promise1, promise2, ... ]，只有当数组里所有 promise 的状态为 fulfilled 或 rejected 后，`await Promise.allSettled(promises)` 才会返回一个数组，该数组包含原始 promises 集中每个promise的结果

#### 场景

1. 批量修改表格的内容，表格每一行的修改都需单独向服务器发起异步操作，操作完后需要在行末展示此次操作结果为失败还是成功
2. 在埋点时，需要上报第一条中批量处理的结果，以计算成功率/失败率

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-promise-allSettled)
2. https://2ality.com/2019/08/promise-combinators.html

### globalThis 

以下解释摘自 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/globalThis)

在以前，从不同的 JavaScript 环境中获取全局对象需要不同的语句。在 Web 中，可以通过 `window`、`self` 或者 `frames` 取到全局对象，但是在 Web Workers 中，只有 `self` 可以。在 Node.js 中，它们都无法获取，必须使用 `global`。

`globalThis` 提供了一个标准的方式来获取不同环境下的全局 `this` 对象（也就是全局对象自身）。不像 `window` 或者 `self` 这些属性，它确保能在有/无窗口的各种环境下正常工作。所以，你可以安心的使用 `globalThis`，不必担心它的运行环境。为便于记忆，你只需要记住，全局作用域中的 `this` 就是 `globalThis`。

### Optional chaining 

先暂且翻译为”可选链“

#### 语法

```js
obj?.prop       // optional static property access
obj?.[expr]     // optional dynamic property access
func?.(...args) // optional function or method call

const street = user.address?.street
iterator.return?.()
```

如果`?.`前面的值是 undefined 或者 null，那该表达式的结果为 undefined

#### 例子

```js
a?.b                          // undefined if `a` is null/undefined, `a.b` otherwise.
a == null ? undefined : a.b

a?.[x]                        // undefined if `a` is null/undefined, `a[x]` otherwise.
a == null ? undefined : a[x]

a?.b()                        // undefined if `a` is null/undefined
a == null ? undefined : a.b() // throws a TypeError if `a.b` is not a function
                              // otherwise, evaluates to `a.b()`

a?.()                        // undefined if `a` is null/undefined
a == null ? undefined : a()  // throws a TypeError if `a` is neither null/undefined, nor a function
                             // invokes the function `a` otherwise

a?.[++x]         // `x` is incremented if and only if `a` is not null/undefined
a == null ? undefined : a[++x]


a?.b.c(++x).d  // if `a` is null/undefined, evaluates to undefined. Variable `x` is not incremented.
               // otherwise, evaluates to `a.b.c(++x).d`.
a == null ? undefined : a.b.c(++x).d

a?.b[3].c?.(x).d
a == null ? undefined : a.b[3].c == null ? undefined : a.b[3].c(x).d
  // (as always, except that `a` and `a.b[3].c` are evaluated only once)

(a?.b).c
(a == null ? undefined : a.b).c // edge case, there is no practical reason to use parentheses in this position anyway
```

#### 注意

为了使`foo?.3:0`向后兼容解析成`foo ? .3 : 0`，此提案要求`?.`后不能紧跟数字

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-optional-chaining)
2.  [Optional chaining](https://2ality.com/2019/07/optional-chaining.html)
3. [@babel/plugin-proposal-optional-chaining](https://babeljs.io/docs/en/babel-plugin-proposal-optional-chaining)

### Nullish coalescing Operator ：??

本文先暂且将它翻译为”双问号操作符“

双问号操作符的目的是在给变量提供默认值时替代以前的 `||`操作符，例如

```js
function getName(userInfo) {
  return userInfo.name || 'david'
}

```

既当 `||` 左边的值为 falsy 时，返回右边的默认值，对于值为 `undefined` 或 `null` 这样 falsy 值是没问题的，但在某些场景下， `0` 、`''` 、 `false` 是程序所期望的 falsy 值，不应该拿到默认值，所以就有了 双问号操作符 的提案，解决有意义的 falsy 值被忽略的问题

#### 语法

```js
falsyValue ?? 'default value'

function getName(userInfo) {
  return userInfo.name ?? 'david'
}
```

#### 例子

```js
const response = {
  settings: {
    nullValue: null,
    height: 400,
    animationDuration: 0,
    headerText: '',
    showSplashScreen: false
  }
};

const undefinedValue = response.settings.undefinedValue ?? 'some other default'; // result: 'some other default'
const nullValue = response.settings.nullValue ?? 'some other default'; // result: 'some other default'
const headerText = response.settings.headerText ?? 'Hello, world!'; // result: ''
const animationDuration = response.settings.animationDuration ?? 300; // result: 0
const showSplashScreen = response.settings.showSplashScreen ?? true; // result: false
```

#### 进一步阅读

1. [Proposal and Specs](https://github.com/tc39/proposal-nullish-coalescing)
2. [Babel plugin 进一步解释](https://babeljs.io/docs/en/babel-plugin-proposal-nullish-coalescing-operator)
3. [Nullish coalescing Operator ](https://2ality.com/2019/08/nullish-coalescing.html)
4. [@babel/plugin-proposal-nullish-coalescing-operator](https://babeljs.io/docs/en/babel-plugin-proposal-nullish-coalescing-operator)

### export * as ns from "mod"

有这个提议后，以下代码将可以被改写

```js
import * as errors from "./errors";
export {errors};
```

改写为：

```js
export * as errors from "./errors";
```

这里有一个真实场景的代码也使用这个提案后就能被简化 [a real world example](https://github.com/open-flash/swf-tree/blob/894f609d1191db5222a68b37ccba8ba57077c58a/swf-tree.ts/src/lib/index.ts#L1-L9) 

#### 进一步阅读

1. [@babel/plugin-proposal-export-namespace-from](https://babeljs.io/docs/en/babel-plugin-proposal-export-namespace-from)

## 参考

1. 阮一峰 [ECMAScript 6 入门](https://es6.ruanyifeng.com/) 
2. [ECMAScript 2020: the final feature set](https://2ality.com/2019/12/ecmascript-2020.html)

