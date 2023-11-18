## Promise



### 背景

JavaScript这种单线程事件循环模型，异步行为是为了优化因计算量大而时间长的操作。在JavaScript中我们可以见到很多异步行为，比如计时器、ui渲染、请求数据等等。

Promise的主要功能，是为异步代码提供了清晰的抽象，支持优雅地定义和组织异步逻辑。可以用Promise表示异步执行的代码块，也可以用Promise表示异步计算的值。

Promise现在主流的翻译为“期约”，在英文里，promise还有承诺的意思，既然是承诺，那就是一种约定，这恰好就符合异步情境的需求：异步的代码不在当前的代码块中调用，而是由外部调用。既然如此，为了获取到异步代码执行的状态，或是为了拿到执行结果，就需要制定一定的规范去获取和维护，Promise A+就是对此指定的规范，Promise类型就是对Promise A+规范的实现。

过去在JavaScript中处理异步，通常会使用一层层的回调嵌套，没有一个规范、清晰的处理逻辑，造成的结果就是阅读困难、调试困难，可维护性差。

Promise A+规范设计的一套逻辑，Promise提供统一的API，可以使我们更有条理的去处理异步操作。

首先，将某个异步任务相关的代码包裹在一个代码块里，也就是Promise执行器函数的函数体中；比如下面的代码：

```javascript
let p1 = new Promise((resolve, reject) => { // 执行器函数
    const add = (a, b) => a + b;
    resolve(add(1, 2));
    console.log(add(3, 4));
});
```

同时，针对这段异步代码的执行状态和执行结果，Promise实例内部会进行维护；

此外，Promise类型内部维护一个resolve和reject函数，用于维护状态的更新，以及调用处理程序将异步执行结果传递给用户进行后续处理，这些处理程序由用户自己定义。

Promise类型实现了Thenable接口，用户可以通过Promise的实例方法then来新增处理程序

当用Promise指代异步执行的代码块时，他涉及异步代码执行的三种状态：进行中等待结果的pending、成功执行fulfilled（一般也用resolved）、执行失败或出现异常rejected。当一个Promise实例被初始化时，其对应的异步代码块就进入进行中的状态，也就是说pending是初始状态。

当代码块执行完毕或者出现异常，将得到最终的一个确定状态，resolved或者rejected，和执行结果，并且不能被再次更新。

### Promise的基本使用

```javascript
let p = new Promise((resolve, reject) => {
    const add = (a, b) => a + b;
    resolve(add(1, 2));
    console.log(add(3, 4));
});
p.then(res => {
    console.log(res);
});
```

### 简易版Promise

针对Promise的基本使用，可以实现一个简易版的Promise

**首先是状态常量的维护**，以便于开发和后期维护：

```javascript
const PENDING = 'pending';
const RESOLVED = 'resolved';
const REJECTED = 'rejected';
```

**然后定义我们自己的MyPromise，维护Promise实例对象的属性**

```javascript
function MyPromise(fn) {
  	const that = this;
    that.state = PENDING;
    that.value = null;
    that.resolvedCallbacks = [];
    that.rejectedCallbacks = [];
}
```

* 首先是state，表示异步代码块执行的状态，初始状态为pending
* value变量用于维护异步代码执行的结果
* resolvedCallbacks用于维护部分的处理程序，处理成功执行的结果
* rejectedCallbacks用于维护另一部分的处理程序，处理的是执行失败的结果

内部使用常量`that`是因为，代码可能会异步执行，这用于获取正确的this。

**接下来定义resolve和reject函数**，添加在MyPromise函数体内部

```javascript
function resolve(value) {
    if (that.state === PENDING) {
        that.state = RESOLVED;
        that.value = value;
        that.resolvedCallbacks.forEach(cb => cb(value));
    }
}

function reject(reason) {
    if (that.state === PENDING) {
        that.state = REJECTED;
        that.value = reason;
        that.rejectedCallbacks.forEach(cb => cb(reason));
    }
}
```

* 首先这两个函数都得判断当前状态是否为pending，因为状态落定后不允许再次修改
* 如果判断为pending，就更新为对应状态，并且将异步执行结果维护到Promise实例的value属性上
* 最后遍历处理程序，并传入异步结果挨个执行

当然**传递给Promise的执行器函数fn也得执行**：

```javascript
try {
    fn(resolve, reject);
} catch (e) {
    reject(e);
}
```

执行器函数接收两个函数类型的参数，实际传入的就是前面定义的resolve和reject。另外，执行函数的过程中可能会抛出异常，需要捕获并执行reject函数。

**最后实现较为复杂的then函数**：

```javascript
MyPromise.prototype.then = function (onResolved, onRejected) {
    const that = this;
    onResolved = typeof onResolved === 'function' ? onResolved: v => v;
    onRejected = typeof onRejected === 'function'
        ? onRejected
        : r => {
            throw r;
        };
    if (that.state === PENDING) {
        that.resolvedCallbacks.push(onResolved);
        that.rejectedCallbacks.push(onRejected);
    }
    if (that.state === RESOLVED) {
        onResolved(that.value);
    }
    if (that.state === REJECTED) {
        onRejected(that.value);
    }
}
```

* 首先判断两个参数是否为函数类型，因为这两个参数是可选参数。
* 当参数不是函数类型时，就创建一个函数赋值给对应的参数，实现透传
* 然后是状态的判断，当Promise的状态是等待结果pending时，就会将处理程序维护到Promise实例内部的处理程序的数组中，resolvedCallbacks和rejectedCallbacks，如果不是pending，就去执行对应状态的处理程序。

至此就实现了一个简易版本的MyPromise，可以进行测试：

```javascript
let p = new MyPromise((resolve, reject) => {
    const add = (a, b) => a + b;
    resolve(add(1, 2));
    console.log(add(3, 4));
});
p.then(res => {
    console.log(res);
});
```

### 进阶版Promise

根据promise的使用经验，我们知道promise解析异步结果是一个微任务，并且promise的原型方法then会返回一个promise类型的值，这些简易版中都没有实现，为了使我们的MyPromise更符合Promise A+的规范，我们需要对简易版进行改造。

**首先是`resolve`和`reject`函数**，这两个函数中的代码会被推入微任务的队列中等待执行

```javascript
  function resolve(value) {
        if (value instanceof MyPromise) {
            return value.then(resolve, reject);
        }

        // 调用queueMicrotask，将代码插入微任务的队列
        queueMicrotask(() => {
            if (that.state === PENDING) {
                that.state = RESOLVED;
                that.value = value;
                that.resolvedCallbacks.forEach(cb => cb(value));
            }
        })
    }

    function reject(reason) {
        queueMicrotask(() => {
            if (that.state === PENDING) {
                that.state = REJECTED;
                that.value = reason;
                that.rejectedCallbacks.forEach(cb => cb(reason));
            }
        });
    }
```

* 对于`resolve`函数，我们首先需要判断传入的值是否为Promise类型，如果是，则要得到x最终的异步执行结果再继续执行resolve和reject
* 此处使用`queueMicrotask`方法将代码推入微任务队列

**接下来继续改造`then`函数中的代码**

* 首先新增一个变量`promise2`用于返回，因为每个then函数都需要返回一个新的Promise对象，该变量就用于保存新的返回对象

  ```javascript
  let promise2; // then方法必须返回一个promise
  ```

* 然后先改造pending状态的逻辑

  ```javascript
  if (that.state === PENDING) {
      return promise2 = new MyPromise((resolve, reject) => {
          that.resolvedCallbacks.push(() => {
              try {
                  const x = onResolved(that.value); // 执行原promise的成功处理程序，如果未定义就透传
                  // 如果正常得到一个解决值x，即onResolved的返回值，就解决新的promise2，即调用resolutionProcedure函数，这是对[[Resolve]](promise, x)的实现
                  // 将新创建的promise2，处理程序返回结果x，以及与promise2关联的resolve和reject函数作为参数传递给 这个函数
                  resolutionProcedure(promise2, x, resolve, reject);
              } catch(r) { // 如果onResolved程序执行过程中抛出异常，promise2就被标记为失败，执行reject
                  reject(r);
              }
          });
          that.rejectedCallbacks.push(() => {
              try {
                  const x = onRejected(that.value); // 执行原promise的失败处理程序，如果未定义就抛出异常
                  resolutionProcedure(promise2, x, resolve, reject); // 解决新的promise2
              } catch(r) {
                  reject(r);
              }
          });
      })
  }
  ```

  整体来看下：

  * 首先创建新的Promise实例，传入执行器函数
  * 大致逻辑还是和之前一样，往回调数组中push处理程序，只是除了onResolved函数之外，还做了一些额外操作
  * 首先在onResolved和onRejected函数调用的时候包裹了一层try/catch用于处理异常，如果出现异常，promise2就被标记为失败，执行其关联的reject函数
  * 如果onResolved和onRejected正常执行，就调用resolutionProcedure函数去解决promise2

* 继续改造resolved状态的逻辑

  ```javascript
  if (that.state === RESOLVED) {
      return promise2 = new MyPromise((resolve, reject) => {
          queueMicrotask(() => {
              try {
                  const x = onResolved(that.value);
                  resolutionProcedure(promise2, x, resolve, reject);
              } catch (r) {
                  reject(r);
              }
          });
      })
  }
  ```

  * 这段代码和pending的逻辑基本一致，不同之处在于，这里直接将处理程序插入微任务队列，而不是push进回调数组
  * rejected状态的逻辑基本也类似

**最后就是实现上述代码中所调用的`resolutionProcedure`函数**，用于解决promise2

```javascript
function resolutionProcedure(promise2, x, resolve, reject) {}
```

* 首先规范规定了x不能与promise2相等，否则会发生循环引用的问题

  ```javascript
  if (promise2 === x) { // 如果x和promise2相等，以 TypeError 为拒因 拒绝执行 promise2
      return reject(new TypeError('Error'));
  }
  ```

* 接着判断x的类型是否为promise

  ```javascript
  if (x instanceof MyPromise) { // 如果x为Promise类型，则使 promise2 接受 x 的状态
      x.then(function (value) {
          // 等到x状态落定后，再去解决promise2，也就是递归调用resolutionProcedure这个函数
          resolutionProcedure(promise2, value, resolve, reject);
      }, reject/*如果x落定为拒绝状态，就用同样的拒因拒绝promise2*/);
  }
  ```

* 处理x的类型不是promise的情况

  首先创建一个变量called用于标识是否调用过函数

  ```javascript
  let called = false;
  if (x !== null && (typeof x === 'object' || typeof x === 'function')) { // 如果x为对象或函数类型
      try {
          let then = x.then; // 取出x上的then属性
          if (typeof then === 'function') { // 判断then的类型是否为函数，进行调用
            	// 根据规范可知，在then调用时，要将this指向x，所以这里使用call对then函数进行调用
              // then接收两个函数类型的参数，第一个参数叫做resolvePromise，第二个参数叫做rejectPromise
              // 如果resolvePromise被执行，则去解决promise2，如果rejectPromise被调用，则promise2被认为失败，会调用其关联的reject函数
              then.call(
                  x, // 将this指向x
                  y => { // 第一个参数叫做resolvePromise
                      if (called) return;
                      called = true;
                      resolutionProcedure(promise2, y, resolve, reject);
                  },
                  r => { // 第二个参数叫做rejectPromise
                      if (called) return;
                      called = true;
                      reject(r);
                  }
              )
          } else { // 如果then不是函数，就将x传递给resolve，执行promise2的resolve函数
              resolve(x);
          }
      } catch (e) { // 如果上述代码抛出异常，则认为promise2失败，执行其关联的reject函数
          if (called) return;
          called = true;
          reject(e);
      }
  } else { // 如果x不是对象或函数，就将x传递给promise2关联的resolve并执行
      resolve(x);
  }
  ```

至此`resolutionProcedure`函数就完成了，最终会执行promise2关联的resolve或者reject函数。之所以说关联，是因为这两个函数中有对实例的引用。

到这为止，进阶版的promise就基本完成了，可以来试用一下：

```javascript
let p = new MyPromise((resolve, reject) => {
    const add = (a, b) => a + b;
    resolve(add(1, 2));
    console.log(add(3, 4));
});
p.then(res => {
    console.log(res);
    return res;
}).then(res => {
    console.log(res);
});
p.then(res => {
    return {
        name: 'x',
        then: function (resolvePromise, rejectPromise) {
            resolvePromise(this.name + res);
        }
    }
}).then(res => {
    console.log(res);
})
```
