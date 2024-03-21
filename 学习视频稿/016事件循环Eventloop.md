## 浏览器事件循环

引言：

事件循环不是浏览器独有的，从字面上看，“循环”可以简单地认为就是重复，比如for循环，就是重复地执行for循环体中的语句，所以事件循环，可以理解为重复地处理事件，那么下一个问题是，处理的是什么事件，事件的相关信息从哪里获取。

因为我没有用nodejs做过什么项目，所以这里我暂且只关注浏览器的事件循环，但我想就“事件循环”本身而言，原理应该是相同的，不过就具体的实现可能存在一些差异。



### 一道面试题

相信应该有部分小伙伴和我一样，在面试中曾遇到过类似于这种问打印结果的题目。

```javascript
(async function main() {
  console.log(1);

  setTimeout(() => {
    console.log(2);
  }, 0);

  setTimeout(() => {
    console.log(3);
  }, 100);

  let p1 = new Promise((resolve, reject) => {
    console.log(4);

    resolve(5);
    console.log(6);
  });

  p1.then((res) => {
    console.log(res);
  });

  let result = await Promise.resolve(7);
  console.log(result);

  console.log(8);
})()
```

这种题目就是变相的在考察事件循环的知识。

我个人感觉事件循环这个点，也是随着Promise的出现，成为了一个常见的考点。



### 什么是事件循环

一提到事件循环，我想很多人会和我一样，立刻想到异步、宏任务、微任务什么的。

#### WIKI

先不着急，我们先看下[Wiki](https://en.wikipedia.org/wiki/Event_loop)上，对事件循环的通用性描述。

> In computer science, the event loop is a programming construct or design pattern that waits for and dispatches events or messages in a program. The event loop works by making a request to some internal or external "event provider" (that generally blocks the request until an event has arrived), then calls the relevant event handler ("dispatches the event"). The event loop is also sometimes referred to as the message dispatcher, message loop, message pump, or run loop.
>
> The event-loop may be used in conjunction with a reactor, if the event provider follows the file interface, which can be selected or 'polled' (the Unix system call, not actual polling). The event loop almost always operates asynchronously with the message originator.
>
> When the event loop forms the central control flow construct of a program, as it often does, it may be termed the main loop or main event loop. This title is appropriate, because such an event loop is at the highest level of control within the program.

简而言之，事件循环是一种编程结构或设计模式，用于在程序中等待和派发事件或消息。

它的工作原理是，向内部或外部的“事件提供者”发出请求（通常会阻止请求，直到事件发生）`这就回答了我们之前的问题：事件的信息从哪里来，是由“事件提供者”提供`，然后调用相关的事件处理程序（“派发事件”）`关于如何处理事件`。

事件循环有时也被称为消息派发器、消息循环、消息泵或者运行循环。

事件循环几乎总是与消息发送者**异步运行**。

```
这里我觉得可以这么理解，“消息发送者”这边将事件的消息交给了“事件提供者”，而事件循环这边会向“事件提供者”发出请求获取事件，然后调用相关的事件处理程序；所以说，事件循环与消息发送者是异步运行。

事件循环必然是在“消息发送者”将事件的消息交出之后，才会去执行事件处理程序；也就是说，事件循环的操作是在当下之后，在”将来“才会发生的。
```

当事件循环构成程序的中心控制流结构时（通常如此），它可以被称为主循环或主事件循环。这个称谓是恰当的，因为这样的事件循环处于程序的最高控制层。

#### MDN

WIKI上提供的是通用性的描述。我们再看一下[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)，MDN上直接搜索事件循环，可以看到是位于JavaScript路径下，针对JavaScript事件循环的描述。

> JavaScript has a runtime model based on an **event loop**, which is responsible for executing the code, collecting and processing events, and executing queued sub-tasks. This model is quite different from models in other languages like C and Java.

第一段很直白的描述：JavaScript的运行时模型，是基于事件循环的，负责执行代码、收集和处理事件以及执行队列中的子任务。

`执行队列中的子任务`：这基本上等于说，是JavaScript的运行时给JavaScript提供了事件循环的能力；可以说JavaScript运行时中**事件循环的部分，提供了JavaScript异步的具体实现方式**。给JavaScript提供了支持异步的能力。

那么**JavaScript为什么要处理异步呢**？这就不得不提JavaScript的单线程运行特性 ，线程是什么？是进行运算调度的最小单位，而JavaScript设计之初，是为了处理网页上的交互事件，如果JavaScript允许多线程，也就是允许多个触发的事件同时进行运算，这可能就会呈现出各种不一样的计算结果，在用户看来就会显得交互很混乱，为了减少不确定性，JavaScript干脆就选择了单线程运行，所有代码都在同一个线程中执行；另外，JavaScript中的交互事件很多，如果每个触发事件都单独开辟线程来处理，也是不小的开销吧。

但是呢，虽然JavaScript是单线程运行的，但也存在需要在将来完成的操作，也就是存在异步代码，比如定时器。如果在Java中，我们也许可以选择new一个线程，sleep多少秒，然后再执行，但是JavaScript中不能这样做，因为它没有多线程，而如果直接在主线程等待，必定会引发阻塞和卡顿。事件循环就是对这种情况的一种解决方案，为了协调浏览器中的各种事件，必须使用事件循环；而事件循环中的消息队列也由JavaScript运行时来管理。

##### 运行时概念

相信不少前端同学都听过“运行时”这个词，那运行时到底是什么呢？我觉得可以这么简单理解，既然运行时的功能是负责执行代码、收集和处理事件以及执行队列中的子任务，那么运行时中必须定义一套规则，关于如何去处理这些事情。所以可以简单地把运行时认为是**定义了一套执行规则的JavaScript执行环境**。

关于运行时，可以看到MDN上有一个直观演示的图，其中包含了函数调用形成的执行栈、分配对象的堆，以及消息队列。

根据WIKI给出的描述，运行时模型中，与事件循环关系最密切的，是消息队列，也就是我们前面提到的“事件提供者”。现在我们来看这个队列。

> A JavaScript runtime uses a message queue, which is a list of messages to be processed. Each message has an associated function that gets called to handle the message.
>
> At some point during the event loop, the runtime starts handling the messages on the queue, starting with the oldest one. To do so, the message is removed from the queue and its corresponding function is called with the message as an input parameter. As always, calling a function creates a new stack frame for that function's use.
>
> The processing of functions continues until the stack is once again empty. Then, the event loop will process the next message in the queue (if there is one).

我们来看翻译的内容：

JavaScript 运行时使用消息队列，这是一个待处理消息列表。每条消息都有一个相关函数被调用来处理该消息。

在事件循环中的某个时刻，运行时开始处理队列中的消息，从最旧的消息开始。`”队“这个数据结构我们知道，是先进先出的，所以先进队的消息会先被处理。`为此，会从队列中移除消息，并将消息作为输入参数调用相应的函数。一如既往，调用函数会创建一个新的堆栈框架供该函数使用。

函数的处理将一直持续到堆栈再次清空为止。然后，事件循环将处理队列中的下一条消息（如果有的话）。`也就是，消息队列中的消息是一条接一条处理的。这里的堆栈指的就是函数调用形成的执行栈和分配对象的堆`



那么队列中的消息是哪里来的呢？从这段内容中我们可以知道，进队的消息已经在等待处理了；所以比如有个定时器setTimeout，定义了有段代码需要等待3秒才执行，那这段代码就不能直接就进队，为了保证动作3秒后才执行，会在3秒后才进队，也就是说，setTimeout的第二个参数代表的是将消息推入队列的延迟时间。

那么肯定需要有什么东西，来管理这段代码，将这段代码在给定的延时后，推入消息队列。既然js没法去开线程管理，所以也是浏览器在管理；Chrome就有一个定时器线程，专门用于处理定时器，在定时器计时结束后，通知事件触发线程将消息推入队列；同样的，在用户触发交互事件时，事件触发线程也会将已在代码中定义的消息推入队列，也就是在事件监听程序addEventListener中监听的操作；还有异步HTTP请求线程，来管理请求回调的消息入队。等等，浏览器的这些线程共同作用来实现事件循环这个机制。



在JS主线程空闲时，就会将这些消息队列中的消息出列，交由主线程来执行。

那么接下来就是事件循环的执行步骤的问题。

### 事件循环执行步骤

首先，关于微任务：我们来看HTML的[文档](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)。

> Each event loop has a microtask queue, which is a queue of microtasks, initially empty. A microtask is a colloquial way of referring to a task that was created via the queue a microtask algorithm.

每个事件循环都有一个微任务队列，这是一个初始为空的微任务队列。微任务是一种通俗的说法，指通过微任务队列算法创建的任务。

也就是说，每个事件循环都会维护一个自己的微任务队列。它和我们之前看的消息队列，不是同一个队列，消息队列指的是这个文档中的任务队列，也就是task queue。

总所周知，常见的产生宏任务的方式有script、setTimeout、setInterval、UI事件等等；常见的产生微任务的方式有Promise.prototype.then、MutationObserver等等。

假设我们在浏览器中加载了一个页面，现在我们来看**事件循环的处理步骤**：

* 初始状态：运行时的**调用栈空**。微任务队列空，消息队列里有且仅有一个script脚本（整体代码）
* 然后消息队列中的**script脚本被推入调用栈**，同步代码开始执行。
* 当碰到微任务时，比如Promise.then，就将微任务推入事件循环的微任务队列中；这里要注意一下，Promise执行器函数中的代码属于同步代码，会被顺序执行；
* 当碰到宏任务时，就将它们丢给相应的浏览器线程；
* 当本次代码中的同步代码都执行完毕后，就将微任务队列中的任务一一处理并出队；
* 这样就完成了一次循环；
* 本次的宏任务script脚本也被出队。
* 此时DOM修改完成，然后浏览器会执行渲染操作，更新界面。
* 如果宏任务在各自的线程中被处理完毕后，就会被推入消息队列。
* 再接着就是当JS主线程空闲后，会去查询队列中是否还有任务，开启新一轮的循环。



现在我们照着最开始的面试题进行举例。

首先，这段代码是一整个script脚本，其中的同步代码会首先被按顺序执行，

可以看到这个script脚本中有一个async异步函数，async函数中的同步代码会首先被执行，所以先会**打印1**，

然后碰到两个产生宏任务的setTimeout，丢给定时器线程，为了后面方便讲述，这里分别把它们叫做宏任务1和宏任务2；

然后执行promise执行器函数中的同步代码，**打印4和6**；

接着碰到Promise.then这个微任务，我们给它记为微任务1，将它推入微任务队列，

然后我们又碰到一个await，await之后的代码相当于是Promise.then中的代码，也就是会被推入微任务队列，我们给它记为微任务2；

到这里，本次循环中的同步代码都执行完毕了；

接着就是开始把微任务队列中的微任务取出执行，首先是执行微任务1，**打印5**；

接着执行微任务2，**打印7和8**；

本次事件循环就结束了。

等到计时结束，宏任务1会先被推入消息队列，在JS主线程空闲，去查询消息队列后，代码就会被执行，会**打印2**；

同理，宏任务2后面也会被执行，并**打印3**。

这样我们就完成了这道面试题的解答。



### 总结

总的来说，事件循环就是JS中异步的具体实现方式，是一种异步方案，它的实现**需要来自宿主环境的支持**，比如浏览器中的各种线程，运行时中的消息队列等等。







