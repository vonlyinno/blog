# 手写Promise系列

首先需要对Promise有哪些要点有一个掌握，具体的可以查看 [Promises/A+规范中文版](https://www.ituring.com.cn/article/66566)。
<br/>

这里我列出几个手写简略版关注的点：

1. promise 是一个拥有 ```then``` 方法的**对象或函数**

&nbsp;&nbsp;&nbsp;1.1. then 方法接受两个参数：```promise.then(onFulfilled, onRejected)```

&nbsp;&nbsp;&nbsp;1.2 then 方法可以被同一个 promise 调用多次

&nbsp;&nbsp;&nbsp;1.3 then 方法必须返回一个 promise 对象

2. 一个 Promise 的当前状态必须为以下三种状态中的一种：**等待态（Pending）、执行态（Fulfilled）和拒绝态（Rejected）**。并且只能有如下的变化关系，不可逆转：```pending -> rejected | fulfilled```
<br/>

### 大致框架：

```js
const Status = {
  Pending: "pending",
  Fulfilled: "fulfiied",
  Rejected: "rejected"
}

class MyPromise {
  constructor(executor) {
    this.status = Status.Pending;
    this.value = null;
    this.reason = null;
    // 成功的callback队列
    this.fulfilledCb = [];
    // 失败的callback队列
    this.rejectedCb = [];

    exec(this._resolve.bind(this), this._reject.bind(this));

  }

  _resolve(value) {
    if (this.status === Status.Pending) {
      this.value = value;
      this.status = Status.Fulfilled;
      // TODO: 依次执行fulfilledCb
    }
  }

  _reject(reason) {
    if (this.status === Status.Pending) {
      this.reason = reason;
      this.status = Status.Rejected;
      // TODO: rejectedCb
    }
  }

  then(onFulfilled, onRejected) {

  }
}
```
<br/>

### 链式调用

Promise的链式调用，就是可以一直.then().then()的调用，这是通过then方法返回一个新的promise实例来完成的，但是具体怎么和上一个promise连通的呢？我看了很多讲解，但是自己的脑回路是一团麻，理解起来还是有困难。把我的思路整理一下，大概是这样：

![20201216233257](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/20201216233257.png)

> 重点理解这句话：如果要通过链式调用触发下一个then，那就需要上一个then返回一个promise，且执行了这个promise的resolve，resolve的值就是上一个then的结果。

最后代码：

```js
const Status = {
  Pending: "pending",
  Fulfilled: "fulfiied",
  Rejected: "rejected"
};

class MyPromise {
  constructor(exec) {
    this.status = Status.Pending;
    this.value = null;
    this.reason = null;
    this.fulfilledCb = [];
    this.rejectedCb = [];

    try {
      exec(this._resolve.bind(this), this._reject.bind(this));
    } catch (e) {
      this._reject(e);
    }
  }

  _runFulfilled() {
    while (this.fulfilledCb.length) {
      const cb = this.fulfilledCb.shift();
      try {
        var returnValue = cb.handler(this.value);
      } catch (e) {
        cb.nextPromise._reject(e);
      }

      if (returnValue instanceof MyPromise) {
        returnValue
          .then((v) => cb.nextPromise._resolve(v))
          .catch((v) => cb.nextPromise._reject(v));
      } else {
        cb.nextPromise._resolve(returnValue);
      }
    }
  }

  _runRejected() {
    while (this.rejectedCb.length) {
      const cb = this.rejectedCb.shift();
      try {
        var returnValue = cb.handler(this.reason);
      } catch (e) {
        cb.nextPromise._reject(e);
      }

      if (returnValue instanceof MyPromise) {
        returnValue
          .then((v) => cb.nextPromise._resolve(v))
          .catch((v) => cb.nextPromise._reject(v));
      } else {
        cb.nextPromise._resolve(returnValue);
      }
    }
  }

  _resolve(value) {
    if (this.status === Status.Pending) {
      this.value = value;
      this.status = Status.Fulfilled;
      this._runFulfilled();
    }
  }

  _reject(reason) {
    if (this.status === Status.Pending) {
      this.reason = reason;
      this.status = Status.Rejected;
      this._runRejected();

      while (this.fulfilledCb.length) {
        const cb = this.fulfilledCb.shift();
        cb.nextPromise._reject(reason);
      }
    }
  }

  then(onFulfilled, onRejected) {
    const p = new MyPromise(() => {});

    this.fulfilledCb.push({
      handler: onFulfilled,
      nextPromise: p
    });
    if (typeof onRejected === "function") {
      this.rejectedCb.push({
        handler: onRejected,
        nextPromise: p
      });
    }

    if (this.status === Status.Rejected) {
      p._reject(this.reason);
    }

    if (this.status === Status.Fulfilled) {
      this._runFulfilled();
    }
    return p;
  }

  catch(onRejected) {
    const p = new MyPromise(() => {});

    this.rejectedCb.push({
      handler: onRejected,
      nextPromise: p
    });

    if (this.status === Status.Rejected) {
      this._runRejected();
    }

    return p;
  }
}

module.exports = MyPromise;
```

[codesandbox]([https://codesandbox.io/s/promise-402vb?from-embed=&file=/src/promise.js:0-2555](https://codesandbox.io/s/promise-402vb?from-embed=&file=/src/promise.js:0-2555))

> 这里是看了youtube上的一个讲解视频，很清楚，还有完整的测试用例。[How JavaScript Promises Work Under the Hood]([https://www.youtube.com/watch?v=C3kUMPtt4hY&t=6s](https://www.youtube.com/watch?v=C3kUMPtt4hY&t=6s))，以及[原文blog]([https://www.vividbytes.io/how-to-implement-JS-promises/](https://www.vividbytes.io/how-to-implement-JS-promises/))

### Promise.all

```jsx
Promise.myAll = (promiseArr) => {
  const arr = Array.from(promiseArr)
  const result = Array(arr.length)
  const len = arr.length
  let resolved = 0
  
  return new Promise((resolve, reject) => {
    arr.forEach((p, index) => {
      Promise.resolve(p).then((res) => {
        result[index] = res
        if (++resolved === len) {
          resolve(result)
        }
      }).catch(reject)
    })
  })
}
```

### Promise.race

```jsx
Promise.myRace = function (iterators) {
    const promises = Array.from(iterators);

    return new Promise((resolve, reject) => {
        promises.forEach((promise, index) => {
            Promise.resolve(promise).then(resolve).catch(reject);
        });
    });
};
```

### Pomisify

```jsx
const promisify = (fn, content) => {
	return (...args) => {
		return new Promise((resolve, reject) => {
			const res = fn.apply(context, [...args, (err, data) => {
				return err ? reject(err) : resolve(data)
			}])
		})
	}
}
```