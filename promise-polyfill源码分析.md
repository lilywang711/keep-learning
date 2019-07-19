## promise-polyfill 源码分析

源码地址：https://cdn.jsdelivr.net/npm/promise-polyfill@8/dist/polyfill.js



在分析源码之前，我们先看看是如何使用 promise 的：

```js
const promise = new Promise((resolve, reject) => {
  try {
    // 可执行一系列操作，例如异步请求数据
    setTimeout(() => {
      const userInfo = { name: 'milly', sex: 1 }
      resolve(userInfo)
    }, 2000)
    
  } catch(e) {
    reject(e)
  }
})

// then 方法
promise.then((info) => {
  // 得到上面异步请求的返回结果
  console.log(info)
}, (err) => {
  throw new Error(err)
})

// catch 方法
promise.catch(err => {
  throw new Error(err)
})

// finally 方法
promise.finally(() => {
  console.log('不管此次 promise 是 resolved 还是 reject 状态都会执行此回调')
})

// resolve 方法
const promise = Promise.resolve(123)

// reject 方法
const promise = Promise.reject(123)

// all 方法
const promise = Promise.all([promise1, promise2, promise3])

// race 方法
const promise = Promise.race([promise1, promise2 promise3])


```



**通过以上代码及日常使用可知：**

1. Promise 是一个构造函数，返回的实例具有 then, catch, finally, resolve, reject, all, race 方法
2. 以上 7 个方法皆返回 promise 对象
3. then, catch, finally 这三个方法定义在 Promise.prototype 上，且因返回的是 promise 对象，因此支持链式调用

### Promise 构造函数

接下来开始分析代码

```js
// Promise 构造函数

function Promise(fn) {
  if (!(this instanceof Promise))
    throw new TypeError('Promises must be constructed via new');
  if (typeof fn !== 'function') throw new TypeError('not a function');
  /** @type {!number} */
  this._state = 0;
  /** @type {!boolean} */
  this._handled = false;
  /** @type {Promise|undefined} */
  this._value = undefined;
  /** @type {!Array<!Function>} */
  this._deferreds = [];

  doResolve(fn, this);
}
```

- 构造函数接受一个 fn 参数
- 第一句 if 判断意思是 Promise 必须通过 `new` 运算符来调用
- 第二句 if 判断意思是传入的 `fn` 参数必须是函数类型
- `_state`: 定义 promise 的状态
  - 0 代表 pending
  - 1 代表 fullfilled
  -  2 代表 rejected
  - 3 是此构造函数自定义的一个状态，代表如果调用 resolve() 时传递的参数为 thenable 类型或者是一个 Promise(后面再详解，先知道就行了)
- `_handled`:  布尔值，记录当前 promise 是否注册了 then 函数来处理 fullfilled 或 rejected 的情况
- `_value`: 保存当前 promise 的值
- `_deferreds`: 数组，其元素形式为 `{onFulfilled, onRejected, promise}`，其作用主要体现在 `then` 方法里，之后再细讲

最后一行`doResolve(fn, this);` 将参数 `fn` 和当前 `this`作为参数，调用 doResolve 函数，接下来看看doResolve 函数的内容。

```js
function doResolve(fn, self) {
  var done = false;
  try {
    fn(
      function (value) {
        if (done) return;
        done = true;
        resolve(self, value);
      },
      function (reason) {
        if (done) return;
        done = true;
        reject(self, reason);
      }
    );
  } catch (ex) {
    if (done) return;
    done = true;
    reject(self, ex);
  }
}
```

首先定义 done 变量，因为 promise 的状态要么 resolved 要么 rejected，只有会这一种值。因此使用此变量来防止 resolve() 和 reject() 都被调用

然后是一个 try catch 结构，catch 结构里 如果 done 为 true，则直接退出。如果 try 结构的代码执行时出现异常，就直接调用这里的 reject()

try 结构里做的事情是传递两个回调函数给 fn，也就是我们使用时的以下代码：

```js
new Promise((resolve, reject) => {
  
})
```

这里的 `(resolve, reject) => {}` 等于这里的 fn 参数，传递的两个回调等同于这里的 resolve, reject，然后执行 fn 函数，这两个回调将会在我们显式的调用 `resolve(userInfo)` 或 `reject(err)` 时调用，因此我们接下来看看 `resolve(self, value)` 和 `reject(self, reason)` 是如何实现的

```js
function resolve(self, newValue) {
  try {
    if (newValue === self)
      throw new TypeError('A promise cannot be resolved with itself.');
    if (
      newValue &&
      (typeof newValue === 'object' || typeof newValue === 'function')
    ) {
      var then = newValue.then;
      if (newValue instanceof Promise) {
        self._state = 3;
        self._value = newValue;
        finale(self);
        return;
      } else if (typeof then === 'function') {
        doResolve(bind(then, newValue), self);
        return;
      }
    }
    self._state = 1;
    self._value = newValue;
    finale(self);
  } catch (e) {
    reject(self, e);
  }
}
```

参数解释：

- self 为当前 promise 对象
- newValue 为传递给 resolve 函数的参数

resolve 函数内部被一个 try catch 包裹住，如果代码执行中有异常，则直接调用 catch 中的 reject(self, e)，

在 try 结构中，第一个 if 语句意思是传递给 resolve 的参数不能是当前 promise 对象，比如下面的栗子：

```js
// 可在浏览器中执行此代码
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(promise)
  }, 100)
})
```

第二个 if 语句是判断传递的参数 newValue 如果类型为对象或者是函数，那么将其 then 属性赋值给变量 then，

然后判断 newValue 是否是 Promise 的实例（即传递给 resolve 的参数 newValue 是一个 promise 对象），如果是，就将当前 _state 状态改为 3，并将当前 _value 改为这个新的 promise 对象，再将当前 promise 对象 _self 作为参数，执行 finale(self)。finale 方法主要和 then 方法有关系，之后再细讲。

接下来判断 newValue 是否为 thenable 类型，如果是，则以 newValue 为 this 的函数和当前 promise 对象 _self 作为参数再次调用doResolve函数

```js
// Polyfill for Function.prototype.bind
// bind 函数是 Function.prototype.bind 的 polyfill
function bind(fn, thisArg) {
  return function () {
    fn.apply(thisArg, arguments);
  };
}
```

其原因是如果传递给 resolve 的参数为 Promise 或者 thenable 类型数据，那么此次 promise 的最终状态和值取决于这个参数的状态和值，代码如下：

```js
// 参数为 Promise 实例
let promise = new Promise((resolve, reject) => {
  resolve(Promise.resolve(222))
})
console.log(promise) // { [[PromiseStatus]]: "resolved", [[PromiseValue]]: 222 }


// 参数为 thenable 类型
const thenable = {
  then: resolve => resolve(222)
}
let promise = new Promise((resolve, reject) => {
  resolve(thenable)
})
console.log(promise) // { [[PromiseStatus]]: "resolved", [[PromiseValue]]: 222 }
```

这也是为何下面栗子中打印结果为具体值，而不是 promise 对象的原因：

```js
axios.get('https://cnodejs.org/api/v1/topics').then(() => {
  return axios.get('https://cnodejs.org/api/v1/topic/5433d5e4e737cbe96dcef312')
}).then(res => {
  console.log(res)
})
```



分析到这里，可以看到最终都调用 finale(self); 这个函数和之后的 then 函数紧密相关，因此为了方便理解，我们先分析 then 方法：

### then 方法

```js
Promise.prototype.then = function (onFulfilled, onRejected) {
  // @ts-ignore
  var prom = new this.constructor(noop);

  handle(this, new Handler(onFulfilled, onRejected, prom));
  return prom;
};

function noop() {}

function Handler(onFulfilled, onRejected, promise) {
  this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : null;
  this.onRejected = typeof onRejected === 'function' ? onRejected : null;
  this.promise = promise;
}
```

可以看到，then方法接受两个参数，从字面意思可知，第一个代表 promise 成功时的回调，第二个代表失败时的回调，即对应以下：

```js
promise.then(() => {
  // 成功回调
}, () => {
  // 错误回调
})
```

其函数内部很简单，只有三行代码：

第一行：以空函数 noop 为参数，调用当前构造函数 Promise（this.constructor 实际指向了构造函数 Promise），并得到一个 promise 实例，赋值给 prom

第二行：将刚刚得到的 promise 实例以及 then 函数传递进来 onFulfilled, onRejected 作为参数传递给 new Handler，得到的结构形式为 `{ onFulfilled, onRejected, promise}`，再将其和 this 传递给 handle 函数

第三行：返回刚刚得到的 prom对象，也就是说 then 方法反悔了一个 promise 对象，这也是为什么 then 方法支持链式调用。

接下里我们看看第二行中提到的 handle 函数：

```js
function handle(self, deferred) {
  while (self._state === 3) {
    self = self._value;
  }
  if (self._state === 0) {
    self._deferreds.push(deferred);
    return;
  }
  self._handled = true;
  Promise._immediateFn(function () {
    var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
    if (cb === null) {
      (self._state === 1 ? resolve : reject)(deferred.promise, self._value);
      return;
    }
    var ret;
    try {
      ret = cb(self._value);
    } catch (e) {
      reject(deferred.promise, e);
      return;
    }
    resolve(deferred.promise, ret);
  });
}
```

首先是一个 while 循环，判断 self._state 是否为 3，也就是 判断 _value 是否为 promise 对象，循环的结果是直到 _value 属性不为 promise 对象

接下里如果 self._ state 为 0（表示 Promise 的状态为 pending），则将 deferred push 进 self._deferreds 数组，并结束函数调用， 其中 deferred为传入的 Handler 实例对象。 self._state 为 0，表示 Promise 的状态为 pending，也就是 Promise 的状态还未改变(比如在执行一个异步操作)，then 方法不知道要执行哪个回调，所在要先保存。这里保存进一个数组而不是一个变量里是因为 then 方法可以被多次调用：

```js
const promise = Promise.resolve(1)
promise.then(n => console.log(n + 1)) // 2
promise.then(n => console.log(n + 2)) // 3
promise.then(n => console.log(n + 3)) // 4

// 正因为保存到一个数组中，所以以上代码都如期执行并正确返回了
```



继续，将 state 为 0 和 3 的情况处理完，就可以将  self._handled = true; 变成 true 状态，标志此 promise 已处理。

接下来调用 Promise._immediateFn 函数，代码如下：

```js
Promise._immediateFn =
  // @ts-ignore
  (typeof setImmediate === 'function' &&
   function (fn) {
  // @ts-ignore
  setImmediate(fn);
}) ||
  function (fn) {
  setTimeoutFunc(fn, 0);
};

var setTimeoutFunc = setTimeout;
```

由此可知，这段代码其实就在二选其一，then 的回调是通过 setImmediate 或者 setTimeout 实现的 异步执行，setImmediate 属于 Node.js ，setTimeout 属于浏览器 window 对象。

继续上面 Promise._immediateFn的回调参数：

```js
var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
```

_state 为 1 时，将 deferred 对象的 onFulfilled 方法赋值给 cb，否则 onRejected

```js
if (cb === null) {
  (self._state === 1 ? resolve : reject)(deferred.promise, self._value);
  return;
}
```

如果传递给 then 的参数不是函数类型，此时会根据 promise 的状态来执行 resolve 或者是 reject，并结束此次调用。注意传入的参数，deferred.promise 和 self._value，也就是说，用 Promise 的值去改变在 then 方法内创建的promise 对象的状态，总结起来就是，若 then 方法未传入对应的回调，那么 Promise 的值会被传递到下一次 then 方法中：

```js
const promise = Promise.resolve(222)
promise.then().then(res => {
  console.log(res) // 222 被传递了
})
```

最后一段代码：

```js
var ret;
try {
  ret = cb(self._value);
} catch (e) {
  reject(deferred.promise, e);
  return;
}
resolve(deferred.promise, ret);
```

将 cb(self._value) 的值赋值给 ret，并以这个 ret 为参数调用 resolve 函数，意思是将 cb 函数的返回值作为 Promise的值传递给下一个 then 方法：

```js
const promise = Promise.resolve(222)
promise.then((n) => {
  return n + 222
}).then(res => {
  console.log(res) // 444
})
```

若抛出异常，则将错误原因作为 Promise 的值，传递给下一个 then 方法

```js
reject(deferred.promise, e);
return;
```



至此， Promise 核心代码包括 then 方法 已分析完。

接下来是 Promise 实现的 catch, finally, resolve, reject, all, race 这几个方法

### catch 方法：

```js
Promise.prototype['catch'] = function (onRejected) {
  return this.then(null, onRejected);
};
```

接受一个函数回调，并将其传递给 this.then 的失败回调

### finally 方法：

```js
Promise.prototype['finally'] = finallyConstructor;
function finallyConstructor(callback) {
  var constructor = this.constructor; // Promise
  return this.then(
    function (value) {
      // @ts-ignore
      return constructor.resolve(callback()).then(function () {
        return value;
      });
    },
    function (reason) {
      // @ts-ignore
      return constructor.resolve(callback()).then(function () {
        // @ts-ignore
        return constructor.reject(reason);
      });
    }
  );
}
```

不管 promise 最终状态为 fullfilled 还是 rejected ，都会执行这个 callback 参数。

较简单，不解释了。

### resolve 方法：

```js
Promise.resolve = function (value) {
  if (value && typeof value === 'object' && value.constructor === Promise) {
    return value;
  }

  return new Promise(function (resolve) {
    resolve(value);
  });
};
```

较简单，不解释了。

### reject 方法：

```js
Promise.reject = function (value) {
  return new Promise(function (resolve, reject) {
    reject(value);
  });
};
```

较简单，不解释了。

### all 方法：

```js
Promise.all = function (arr) {
  return new Promise(function (resolve, reject) {
    if (!isArray(arr)) {
      return reject(new TypeError('Promise.all accepts an array'));
    }

    var args = Array.prototype.slice.call(arr);
    if (args.length === 0) return resolve([]);
    var remaining = args.length;

    function res(i, val) {
      try {
        if (val && (typeof val === 'object' || typeof val === 'function')) {
          var then = val.then;
          if (typeof then === 'function') {
            then.call(
              val,
              function (val) {
                res(i, val);
              },
              reject
            );
            return;
          }
        }
        args[i] = val;
        if (--remaining === 0) {
          resolve(args);
        }
      } catch (ex) {
        reject(ex);
      }
    }

    for (var i = 0; i < args.length; i++) {
      res(i, args[i]);
    }
  });
};
```

这段代码不难理解，首先函数主体返回一个 promise 对象，然后判断传入参数是否为数组类型

```js
if (!isArray(arr)) {
  return reject(new TypeError('Promise.all accepts an array'));
}

// 个人认为这个判断是否为数组的函数有点草率
function isArray(x) {
  return Boolean(x && typeof x.length !== 'undefined');
}
```

通过 Array.prototype.slice.call 拷贝 arr 给 args，判断如果 args 长度为 0，则直接返回空数组。并将 args 的长度赋值给 remaining

```js
var args = Array.prototype.slice.call(arr);
if (args.length === 0) return resolve([]);
var remaining = args.length;
```

接下来是核心代码，先看最下面的循环体：

```js
for (var i = 0; i < args.length; i++) {
	res(i, args[i]);
}
```

遍历 args ，分别将 i 和 arg[i] 值作为参数调用 上面定义好的 res 函数

```js
if (val && (typeof val === 'object' || typeof val === 'function')) {
  var then = val.then;
  if (typeof then === 'function') {
    then.call(
      val,
      function (val) {
        res(i, val);
      },
      reject
    );
    return;
  }
}
```

判断如果参数 val 是对象或者是函数类型，就将 val.then 赋值给 then，判断 then 是否是函数，如果是，则传递当前 val 以及两个 onFullfiled 和 onReject 回调函数，并执行 then 函数。

需要注意的是这里 onReject 回调函数就是最开始 Promise.all 返回的 promise 对象的 reject 回调，这也是为什么在 Promise.all 中只要其中一个 promise 是 reject 的状态，那么这个 Promise.all 将直接进入 reject。

另外需要注意的是传递的这个 onFullfiled 回调：

```js
function (val) {
  // 此时的 val 为 arg[i] 这个 promise 的 resolve 值，然后在再传递给 res(i, val);
  res(i, val);
},
```

```js
// 如果 val 值不是对象或者函数类型，则直接赋值给 args 对应索引，通过 --remaining === 0 判断当前数组是否全部处理完成，处理完成之后执行 resolve(args);
args[i] = val;
if (--remaining === 0) {
  resolve(args);
}
```



举个栗子：

```js
const promise1 = Promise.resolve(111)
const promise2 = Promise.resolve(222)

Promise.all([promise1, promise2]).then(list => {
  console.log(list) // [111, 222]
})

```

代码执行顺序详解：

1. 将 [promise1, promise2] 赋值给 args，并进入循环体，执行 res 函数
2. res 函数里判断 promise1 是对象类型，调用 Promise.resolve(1) 的 then 方法，得到 1 赋值给 val，再执行 res(0, 111)
3. 重新进入 res 方法后，直接走到 args[0] = 111 这一步，接下来 promise2 的逻辑同上



### race 方法：

```js

Promise.race = function (arr) {
  return new Promise(function (resolve, reject) {
    if (!isArray(arr)) {
      return reject(new TypeError('Promise.race accepts an array'));
    }

    for (var i = 0, len = arr.length; i < len; i++) {
      Promise.resolve(arr[i]).then(resolve, reject);
    }
  });
};
```

 Promise.race 的意思是「谁先走到 resolved，就返回谁」，有点竞赛的意思，理解了这个再看上面的代码就简单了很多，不管数组里是否是异步 or 同步，都直接带入 Promise.resolve 中执行，then 的两个回调直接使用最顶层的 resolve, reject

### 其它方法：

### _unhandledRejectionFn

```js
Promise._unhandledRejectionFn = function _unhandledRejectionFn(err) {
  if (typeof console !== 'undefined' && console) {
    console.warn('Possible Unhandled Promise Rejection:', err); // eslint-disable-line no-console
  }
};
```





