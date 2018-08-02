#### 0.怎么理解javascript单线程
可以通过下面的等式来表示:

**One thread == One call stack == One thing at a time**

也就是说javascript只有一个线程，只有一个调用栈，栈中的代码是队列式顺序执行的。但是同时要记住下面这句话:

**Browser is more than runtime!**

浏览器提供了很多机制(Stack和Heap属于Javascript的runtime部分):比如事件队列，事件循环，WebAPI等(请继续阅读下面的部分)。但是,调用栈中的函数在执行的时候浏览器不能做任何事情，包括`UI渲染或者响应用户输入`。这样对于一些计算密集的函数来说，要尽量将它转化为异步的，浏览器可以在执行函数的间隙给UI渲染特定的时间，而不会出现界面假死状况。因为UI渲染要获得时间与调用栈的状态有关，如果是同步的函数，那么代码一直在执行，进而调用栈一直不为空，那么浏览器就无法给UI渲染队列(Render Queue)回调函数分配时间，但是异步可以，因为浏览器会自己做处理，在异步函数调用间隙给UI线程特定的渲染时间,而且UI队列本身的权重要比回调队列的权重要高。下面给出具体的实例:
```js
console.log('started');
//放在webapi里面等待执行，如果你点击了很多次，那么其实这些回调函数都会放在Callback Queue中等待执行。WebAPI和setTimeout不一样的地方在于只会添加一次$.on('button','click')
$.on('button','click',function onClick(){
  console.log('clicked!');
})
setTimeout(function onTimeout(){
   console.log('Timeout finished!');
},5000);
console.log('Done');
```
所以多次点击某一个有事件监听的元素的时候，如果该回调非常耗时，一定要注意。因为多次回调函数都会被推到Callback Queue中等待被放入调用栈中执行。而不会因为前一个回调还没有执行结束就取消后面插入的回调函数(注意[setInterval](https://github.com/liangklfangl/react-article-bucket/blob/master/js-native/foundamental-QA.md#3深入理解settimeout与setinterval)的特殊情况和下面的Mutation的特殊情况)。也正因为所有的回调是放在回调队列中等待执行的，所以setTimeout往往指定的是最小间隔时间，除非执行栈一直处于空闲状态:
```js
//callback是需要等待的，所以1000是最小时间间隔
setTimeout(function onTimeout(){
   console.log('Timeout finished!');
},1000);
setTimeout(function onTimeout(){
   console.log('Timeout finished!');
},1000);
setTimeout(function onTimeout(){
   console.log('Timeout finished!');
},1000);
setTimeout(function onTimeout(){
   console.log('Timeout finished!');
},1000);
```
上面讲了事件监听的回调函数不会因为前面一个没有执行就不推入Callback Queue中了，所以对于Scroll事件监听的时候一直就要关注debounce，函数节流等:
```js
//因为滚动的时候会有很多回调函数被添加到callback  queue,所以需要debouncing
function animateSomething(){
  delay();
}
$.on("document","scroll",animateSomething)
```

#### 1.生产者与消费者理解事件循环
工作线程是生产者，主线程是消费者(只有一个消费者)。工作线程执行`异步任务(产生事件)`，执行完成后把对应的回调函数封装成为一条消息放在消息队列中。主线程源源不断的从消息队列中取消息并执行。当消息队列为空的时候主线程会阻塞，直到消息队列再次非空。
```js
while(true){
  var message = queue.get();
  execute(message);
}
```
#### 2.浏览器的事件循环深入详解（Event Loop）？
##### 2.1 浏览器事件循环详解
一个事件循环有一个或者多个任务队列（task queues）。任务队列是task的有序列表，这些task是以下工作的对应算法：`Events(比如下面的例子中的click事件的触发就是Task)`，Parsing，Callbacks，Using a resource，Reacting to DOM manipulation,setImmediate(setImmediate.js), MessageChannel,window.postMessage，setTimeout,setInterval[等等](https://segmentfault.com/a/1190000008589736)。

每一个任务都来自一个特定的`任务源`（task source）。所有来自一个特定任务源并且属于特定事件循环的任务，通常必须被加入到同一个任务队列中，但是来自不同任务源的任务可能会放在不同的任务队列中。

举个例子，用户代理有一个处理鼠标和键盘事件的任务队列。用户代理可以给这个队列比其他队列多3/4的执行时间，以确保交互的响应而不让其他任务队列饿死（starving），并且不会乱序处理任何一个任务队列的事件。

`每个事件循环`都有一个进入[microtask](https://www.w3.org/TR/html5/webappapis.html#microtask)检查点（performing a microtask checkpoint）的flag标志，这个标志初始为false。它被用来组织反复调用‘进入microtask检查点’的算法。总结一下，一个事件循环里有很多个任务队列（task queues）来自不同任务源，每一个任务队列里的任务是严格按照先进先出的顺序执行的，但是不同任务队列的任务的执行顺序是不确定的。按我的理解就是，浏览器会自己调度不同任务队列。网上很多文章会提到macrotask这个概念，其实就是指代了标准里阐述的task。

标准同时还提到了microtask的概念，也就是微任务。看一下标准阐述的事件循环的进程模型：
<pre>
1.选择当前要执行的任务队列，选择一个最先进入任务队列的任务，如果`没有任务可以选择(如果任务已经执行完毕，直接跳转到microtasks步骤，否则执行第2步)`，则会跳转至microtask的执行步骤。
2.将事件循环的当前运行任务设置为已选择的任务。
3.运行任务。
4.将事件循环的当前运行任务设置为null。
5.将运行完的任务从任务队列中移除。
6.microtasks步骤：进入microtask检查点（performing a microtask checkpoint ）。
7.更新界面渲染。
8.返回第一步。   
</pre>

执行进入microtask检查点时，用户代理会执行以下步骤：
<pre>
1.设置进入microtask检查点的标志为true。
2.当事件循环的微任务队列不为空时：选择一个最先进入microtask队列的microtask；设置事件循环的当前运行任务为已选择的microtask；运行microtask；设置事件循环的当前运行任务为null；将运行结束的microtask从microtask队列中移除。
3.对于相应事件循环的每个环境设置对象（environment settings object）,通知它们哪些promise为rejected。
4.清理indexedDB的事务。
5.设置进入microtask检查点的标志为false。    
</pre>


现在我们知道了。在事件循环中，用户代理会不断从`task队列`中按顺序取task执行，每执行完一个task都会检查microtask队列是否为空（执行完一个task的具体标志是`函数执行栈`为空），如果不为空则会一次性执行完所有microtask。然后再进入下一个循环去task队列中取下一个task执行...

那么哪些行为属于task或者microtask呢？标准没有阐述，但各种技术文章总结都如下：
<pre>
macrotasks: script(整体代码), setTimeout, setInterval, setImmediate, I/O, UI rendering
microtasks: process.nextTick, Promises, Object.observe(废弃), MutationObserver
</pre>

关于事件循环还要注意一点,你可以查看[为什么Vue使用MutationObserver做批量处理？](https://github.com/liangklfangl/react-article-bucket/blob/master/vue/inner-core-concept.md#2%E4%B8%BA%E4%BB%80%E4%B9%88vue%E4%BD%BF%E7%94%A8mutationobserver%E5%81%9A%E6%89%B9%E9%87%8F%E5%A4%84%E7%90%86),即:每次macrotask执行完成后会进行UI更新，所以**为了使得界面更快更新，我们应该使用microtask而不是macrotask**。

##### 2.2 浏览器事件循环详细例子
###### 2.2.1 Promise+setTimeout例子
比如下面例子:
```js
console.log('script start');
setTimeout(function() {
  console.log('setTimeout');
}, 0);
Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});
console.log('script end');
```
运行结果是：
<pre>
script start
script end
promise1
promise2
setTimeout  
</pre>

解释如下:

1.一开始task队列中只有script，则script中所有函数放入函数执行栈执行，代码按顺序执行。
接着遇到了`setTimeout`,它的作用是0ms后将回调函数放入task队列中，也就是说这个函数将在`下一个事件循环`中执行（注意这时候setTimeout执行完毕就返回了）。

2.接着遇到了Promise，按照前面所述Promise属于microtask，所以第一个.then()会放入microtask队列。

3.当所有script代码执行完毕后，此时`函数执行栈(下面的例子的onclick调用函数执行栈不为空)`为空。开始检查microtask队列，此时队列不为空,执行.then()的回调函数输出'promise1'，由于.then()返回的依然是promise,所以第二个.then()会放入microtask队列继续执行,输出'promise2'。

4.此时microtask队列为空了，进入下一个事件循环，检查task队列发现了setTimeout的回调函数，立即执行回调函数输出'setTimeout'，代码执行完毕。

###### 2.2.2 上面的例子只是保证Promise.then里面的代码提前执行而已
知乎上有一个很热的帖子[Promise的队列与setTimeout的队列有何关联？](https://www.zhihu.com/question/36972010),它给出了和上文类似的代码:
```js
setTimeout(function() {
    console.log(4)
},0);
// macrotask
new Promise(function(resolve) {
    console.log(1) 
    for (var i = 0; i < 10000; i++) {
        //这里即使是没有i==9999也是一样的结果
        //因为promise只会care第1次的resolve
        //但是for循环会支持多次
        i == 9999 && resolve()
    }
    console.log(2)
}).then(function() {
    console.log(5)
});
console.log(3);
// 此时当前的macrotask被执行完毕，才会去检查microtask队列
```
此时的打印结果是"1;2;3;5;4"。它和上面的例子唯一不同的地方在于Promise的构造函数，**这里不是直接调用Promise.resolve方法**，因此**new Promise里面传入的函数会立即执行，而不会被放到macrotask或者microtask里面，其就相当于普通的代码而已**。这一点一定要分清楚。

###### 2.2.3 MutationObserver与事件回调
下面再给出一个例子:
```html
<div class="outer">
  <div class="inner"></div>
</div>
```
下面是js代码:
```js
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');
// 监听outer元素的属性改变
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});
//DOM的click监听器
function onClick() {
  console.log('click');
  setTimeout(function() {
    console.log('timeout');
  }, 0);
  Promise.resolve().then(function() {
    console.log('promise');
  });
  outer.setAttribute('data-random', Math.random());
}
//outer和inner都监听click事件
//处理函数会被放到事件队列里面，但是事件本身并非macrotask/microtask
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```
此时打印结果为:
<pre>
click
promise
mutate
click
promise
mutate
timeout
timeout    
</pre>
解释一下过程：

点击inner输出'click'，Promise和设置outer属性会依次把Promise和MutationObserver推入microtask队列，setTimeout则会推入task队列。此时`执行栈为空`，**虽然后面还有冒泡触发，但是此时microtask队列会先执行**，所以依次输入'promise'和'mutate'。接下来事件冒泡再次触发事件，过程和开始一样。接着代码执行完毕，此时进入下一次事件循环，执行task队列中的任务，输出两个'timeout'。

好了，如果你理解了这个，那么现在换一下事件触发的方式。在上面的代码后面加上
```js
inner.click()
```
运行结果为:
<pre>
click
click
promise
mutate
promise
timeout
timeout
</pre>
造成这个差异的结果是什么呢？在前面的例子中，**微任务是在事件处理函数之间来执行的!!!**。但是，如果你直接调用了`.click()`函数，那么事件是`同步触发`的，因此，此时调用`click()`方法的代码在多次回调的时候一直是在执行栈中的。这样，我们能够保证在执行js代码的时候不会被微任务打断。这也意味着，我们在多次回调函数的过程中也就不会触发微任务的执行，微任务的执行被推迟到多个事件处理函数执行之后。上面对于执行过程的分析，可能会有如下的图表:

![](./images/stack.png)

其中有一点需要注意:那就是"JS stack"部分，当Promise回调执行的时候其中就是"Promise callback"，而当Mutation observers被执行的时候就会是Mutation callback。也就是说JS stack中存放的是当前执行栈中执行的代码。同时，还有一点要注意:比如上面的inner/outer点击的事件例子，因为click会冒泡，`第一次click执行完毕以后会继续执行outer元素的click事件`,此时的事件执行过程会变成下面的状态:

![](./images/stack2.png)

而对于上面的`.onclick()`例子来说，初始的js stack状态为:

![](./images/stack3.png)

如果继续往下执行，你会看到如下的状态:

![](./images/stack4.png)

如果再继续执行，onClick事件处理函数又会继续放入"js Stack"中，而和上面通过手动点击触发click的不同之处在于:

![](./images/stack5.png)

即:"因为JS Stack没有处于空的状态，所以无法执行微任务，而本质原因在于此时**事件的触发是同步的**!"。还有一点不知道你有没有注意到:**上面的MutationEvent只执行了一次**，这是因为**第一次MutationEvent还处于pending状态，所以不会再次添加**!

![](./images/pending.png)


#### 3.代码角度理解浏览器的事件循环？
##### 3.1 Callback Stack调用流程
例如，当你的javascript代码发起了一个Ajax请求去获取服务端的数据，你在回调函数中设置了response对象，此时相当于JS Engine告诉宿主环境:"Hi,我现在要暂停执行了，当你完成了这个网络请求同时获取到了数据，记得调用回调函数"。此时，我们的浏览器会监听是否有网络请求返回，当有数据返回了，那么它就会将我们的回调函数插入到事件循环中(Event Loop)。比如下图:

![](./images/web.png)

其实这里面最重要的就是:`Web API`。本质上，它们代表了特定的线程，我们无法直接访问它，但是我们可以调用它。他们是浏览器中那部分与并发相关的内容，如果你是Nodejs开发者，它们代表的是C++相关的API内容。那么事件循环到底指的的是什么?看下图:

![](./images/callStack.png)

其实Event Loop只干一件事:监听`Call Stack`和`Callback队列`，如果Call Stack已经空了，那么他就会从Callback队列中拿出第一个事件，然后将它推入到Call Stack中执行！每一个过程就是Event Loop中一次Tick，每一个事件就是一个函数回调。我们给出下面的代码执行过程:
```js
console.log('Hi');
setTimeout(function cb1() { 
    console.log('cb1');
}, 5000);
console.log('Bye');
```
首先第一个状态是浏览器的console控制台是空的，同时Call Stack的状态是空的:

![](./images/1.png)

接着`console.log('Hi')`被添加到Call Stack中:

![](./images/2.png)

然后`console.log('Hi')`被执行了，浏览器控制台打印出"Hi":

![](./images/3.png)

`console.log('Hi')`被执行完成后，将会从Call Stack中被移除:

![](./images/4.png)

接着`setTimeout(function cb1() { ... })`被添加到Call Stack中:

![](./images/5.png)

接着`setTimeout(function cb1() { ... })`被执行了，此时浏览器创建了一个Web APIs中的一个`Timer`,这个Timer将会为你处理倒计时操作:

![](./images/6.png)

接着`setTimeout(function cb1() { ... })`被执行完毕，同时从Call Stack中被移除:

![](./images/7.png)

接着`console.log('Bye')`被添加到Call Stack中:

![](./images/8.png)

然后`console.log('Bye')`被执行了，打印出'Bye':

![](./images/9.png)

接着`console.log('Bye')`从Call Stack中被移除了:

![](./images/10.png)

在5000ms以后，`Timer完成了`，然后将cb1这个回调函数从`Web APIs`推入到回调函数队列(Callback Queue)准备执行。

![](./images/11.png)

然后，`Event Loop(只干一件事:监听`Call Stack`和`Callback队列`，如果Call Stack已经空了，那么他就会从Callback队列中拿出第一个事件，然后将它推入到Call Stack中执行)`从回调队列中拿出cb1函数，并将它推入到调用栈Call Stack。如下图:

![](./images/12.png)

此时cb1被执行了，同时添加`console.log('cb1')`到Call Stack中:

![](./images/13.png)

此时，我们的`console.log('cb1')`被执行了，并打印`cb1`:

![](./images/14.png)

然后，我们的`console.log('cb1')`从Call Stack中被移除:

![](./images/15.png)

最后，我们的`cb1`回调函数也会从Call Stack中被移除:

![](./images/16.png)

而最终将所有流程串联起来的gif将会得到下面的图片:

![](./images/17.gif)

##### 3.2 setTimeout如何工作
`setTimeout()`不会自动将回调放到`事件循环队列`中，而是设置一个Timer。当Timer到期，JS执行环境将会将回调放到事件循环队列中，事件循环的后续Tick将会自动执行该回调。比如下面的代码:
```js
setTimeout(myCallback, 1000);
```
并不表示`myCallback`将会在1000ms后执行，**而是在1000ms后添加到事件循环队列**。而事件循环队列可能有更早的回调已经被添加进去，因此此时的`myCallback`可能需要等待一段时间执行。而常见的`setTimeout(callback,0)`表示**将回调函数推迟到Call Stack为空的时候执行**。比如下面的例子:
```js
console.log('Hi');
setTimeout(function() {
    console.log('callback');
}, 0);
console.log('Bye');
```
此时`setTimeout`的第二个参数设置为0，此时浏览器的console将会打印如下的信息:
<pre>
Hi
Bye
callback
</pre>


#### 4.JS引擎 vs 执行环境 vs Call Stack
##### 4.1 内存堆和调用栈
我们的JS引擎包含两个部分:
<pre>
* Memory Heap — 和内存分配有关
* Call Stack — 代码执行时候的栈帧(数据栈中的数据帧)
</pre>

具体见下面:

![](./images/memory.png)

##### 4.2 Runtime执行环境
在日常的开发中我们可能会遇到如`setTimeout`的方法，这些方法并不是JS引擎提供的。那么这些方法从哪里来的呢?请看下面的图:

![](./images/lop.png)

因此，我们不仅含有JS引擎，真实的JS运行环境可能包含更多的内容。比如这里的`Web APIS`,这部分的内容`是浏览器`本身提供的，比如DOM(`上面的onload,onclick,onDown都与事件有关`)，AJAX,setTimeout等等。同时，我们还有广为人知的事件循环Event Loop和回调队列。

##### 4.3 [Callback Stack](https://blog.risingstack.com/node-js-at-scale-understanding-node-js-event-loop/)调用栈(异步回调解决耗时调用)
JS是单线程的编程语言，这也意味着它只有一个Call Stack调用栈。因此，在同一时间只能做一件事情。调用栈是一个数据结构，其真实的记录`当前运行的代码`，如果进入一个函数，那么将该函数放在调用栈的顶部，如果函数返回了，那么该函数将会从调用栈顶部被移除，这些都是Call Stack完成的事情。比如下面的代码:
```js
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```
此时调用栈将会经历下面的过程:

![](./images/callSt.png)

上面的每一个状态都被称为栈帧(数据栈中的数据帧)。这也表明了当异常发生以后，stack track是如何构建的。比如下面的例子:
```js
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```
这个代码如果在chrome中运行，将会打印如下的错误栈(比如文件是foo.js):

![](./images/err.png)

因为JS只有一个堆栈，因此可能会看到如下的错误:

(1)“Blowing the stack(堆栈溢出)” — 当超过Call Stack最大的值，一般如死循环。比如下面的例子:
```js
function foo() {
    foo();
}
foo();
```
此时的调用栈如下:

![](./images/recur.png)

(2)并发和事件循环(Concurrency & the Event Loop)

如果调用栈中某一个函数的执行消耗了大量的时间，此时浏览器无法去处理其他的事情，就会出现阻塞。比如无法render，也无法执行其他的js代码。此时浏览器可能出现下面的界面要求你终止网页:

![](./images/terminate.png)

那么，我们有什么方法解决在我们执行耗时的js的时候不会阻塞UI渲染？此时就需要我们的异步回调了(请参考上面的`第3点事件循环`内容)。


#### 5.V8中的js代码是如何执行的？
###### 5.1 V8引擎作用
V8引擎一开始是为了提升在浏览中js的执行速度而设计的。为了提升速度，V8将js代码翻译为`机器码(是电脑的CPU可直接解读的数据,计算机可以直接执行，并且执行速度最快的代码)`，而`不是使用一个解析器`。和其他浏览器，如SpiderMonkey，Rhino (Mozilla)一样，通过实现JIT (Just-In-Time)编译器，它在js代码执行的过程中将它转化为机器码。而和其他浏览器不一样的地方是，V8不会产生`字节码(字节码是一种中间码，它比机器码更抽象，需要直译器转译后才能成为机器码的中间代码。字节码的典型应用为Java bytecode)`，或者其他的`中间码`。

在V5.9之前V8使用了两个Compiler:

<pre>
  full-codegen:一个很简单但是很快的解析器，用于产生简单但是运行相对较慢的机器码
  Crankshaft:一个复杂的(Just-In-Time)优化的解析器，用于产生高度优化后的机器码
</pre>

V8引擎在内部也使用了多个线程:
<pre>
主线程:获取你的代码，编译并执行
其他线程:编译代码，这样当其他线程在优化代码的时候主线程能够保持执行
Profiler线程:告诉执行环境那个方法消耗了大量的时间，这样Crankshaft编译器能够优化它
垃圾回收线程:用于垃圾回收
</pre>

当首次执行JS代码的时候，V8使用full-codegen这个解析器将解析后的JS代码直接转化为机器码。这样，V8能够快速执行机器码。注意:V8不会使用中间码，因此不需要一个编译器(类似于java虚拟机)。

当你的代码执行了一段时间，我们的`Profile线程`已经收集了足够多的数据,可以用于判断哪些方法应该被优化。此时，Crankshaft在一个新的线程中开始优化代码。它将JS的AST抽象语法树转化为更高级别的static single-assignment (SSA)，也被称为`Hydrogen`。同时对Hydrogen进行优化，很多类型的优化都是在此时完成的。

###### 5.2 如何根据V8引擎来优化代码
(1)内联代码

第一个优化的点就是提前内联尽可能多的js代码。内联代码就是将调用函数的那一段代码使用函数本身内容来替换。这个简单的步骤将会使得下面的优化变得有意义:

![](./images/inline.png)

(2)理解Hidden Class原理

如下图:

![](./images/hidden.jpg)

这是Java，它可以共享`Class info` ，很好理解因为Java 不是动态脚本，`运行时`不能为类对象添加属性。它的 Class info可以在指定内存地址保存固化。成员对象属性先访问到info 表，获取得到`属性对应值地址偏移后`，通过指针偏移得到属性值。而对于JS对象来说，它的对象压根就是通过哈希表存储，存上key和value完事，key被hash，value是值地址。比如下面的js代码:
```js
function Point(x, y) {
 this.x = x;
 this.y = y;
}
var p1 = new Point(10, 11);
var p2 = new Point(12, 13);
```
在内存中存储后得到如下的内容:

![](./images/js.jpg)

Point的成员属性都是相同的，但是被分散存储了。每个对象取值都要`hash key`然后再找到其value。而[hash](https://lz5z.com/JavaScript-Object-Hash/)本身的速度较慢，因此才有了通过Hidden Class来提速的情况，因为Hidden Class本身是可以提速的，这样就不需要在访问每一个JS对象的对应的key的时候都做一次hash key!

在 JavaScript中对象是以[Hash结构](https://baike.baidu.com/item/%E5%93%88%E5%B8%8C%E8%A1%A8/5981869?fr=aladdin&fromid=10027933&fromtitle=%E6%95%A3%E5%88%97%E8%A1%A8)存储的，用`<Key, Value>`键值对表示对象的属性，Key 的数据类型为字符串，Value 的数据类型是`结构体`，即对象是以 <String, Object> 类型的`HashMap` 结构存储的。


#### 6.什么是JavaScript Binding？
在html页面中，我们可以通过JavaScript语句来访问DOM节点，例如document.createElement(“canvas”); 可是document所指向的对象`HTMLDocument存在于WebKit中`，通过C++实现的，`并不存在于JavaScript的引擎中`，所以如果想要在web页面中也能通过JavaScript来访问webkit定义的对象，这就需要把WebKit中的对象*注入*到JavaScript引擎中，而JavaScript引擎中对象的表示方式与WebKit中对象的表示方式存在差别，就需要存在一种方式，`把WebKit中的对象转换成JavaScript引擎能识别的对象，这一过程就称为JavaScript Binding`。例如以V8引擎为例，HTMLDocument --> V8HTMLDocument --> v8::Object.
在Google开发Blink（来自于WebKit）以前，WebKit存在两种JavaScript引擎，即v8和JSC(JavaScriptCore)。v8是由Google公司开发的，应用在Chrome和Android原生浏览器中。JSC是有Apple公司开发，应用Safari中。那么WebKit中也同样存在两种相对应的JavaScript Binding，即JSC Binding和v8 binding。除了JavaScript Binding之外，WebKit中还存在一些其他的语言binding，例如Objective-C binding, GObject binding和CPP binding。Objective-Cbinding让JavaScript可以访问Objective-C的对象和方法，GObject binding存在于GTK中。
JavaScript Binding如何工作？

如果需要把WebKit中实现的`DOM对象或HTML5对象`暴露给JavaScript，让web开发者在JavaScript中能够访问，就需要在WebKit中为每个对象实现相应的JavaScript Binding文件，以v8引擎为例，我们需要为HTMLDocument对象实现一个V8HTMLDocument的对象，目的是转化HTMLDocument对象为v8::Object。只是WebKit中的DOM对象如此庞大，再加上后续添加的HTML5对象，如果需要为每个对象都亲自完成一个cpp文件实现一个相应的转换类，工作量是可想而知的。WebKit为了解决上述问题，编写了一套工具，可以让WebKit在编译时自动生成相应的binding文件，省去了开发者重复劳动的麻烦。
在介绍binding工具之前，首先介绍一个Web IDL。Web IDL是W3C Web工作组为了定义Web接口定义的一套标准接口，全称为Web Interface Description Language. 如果浏览器需要实现某项新功能，一般的流程是W3C委员会先讨论该功能，并给出该功能的接口定义，也就是Web IDL，然后各个浏览器厂商来实现该接口，这样就能最大程度的保证各个浏览器的兼容性，才不至于Web开发者采用标准接口开发的Web网页或应用程序只能在某个特定浏览器上运行。

#### 7.什么是浏览器的Web API?
##### 7.1 Web API以及相关概念?
客户端的Web API是为了扩展Web浏览器的功能或者其他的HTTP客户端设计的编程接口。常见的Web API一开始都是通过浏览器内置扩展程序实现的，而现在更加倾向于使用JS binding来完成。[Mozilla组织](https://en.wikipedia.org/wiki/Mozilla_Foundation)定义了他们自己的WebAPI标准，目的是使用HTML5形式的应用来替换原生的手机应用。Google也开发了他们自己的Native Client去替换不安全的这种浏览器内置的插件形式，转而采用安全的原生的sandboxed的扩展程序或者应用。这些WebAPI包括常见的DOM, AJAX, setTimeout[等等](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf),这些WebAPI并不是JS引擎提供的，而是宿主环境，例如`浏览器`提供的，这一点一定要注意!那么我们下面给出一个比较[简短的定义](https://medium.com/@gaurav.pandvia/understanding-javascript-function-executions-tasks-event-loop-call-stack-more-part-1-5683dea1f5ec):

<pre>
Browser Web APIs:浏览器创建的C++实现的线程，其专门用于处理异步的事件，如DOM事件,http Request,setTimeout等。WebAPIs本身
无法将执行代码放置到栈中进行执行，每一个WebAPI在执行完成以后将回调放到我们的事件队列中。而Event Loop就是检查执行栈和事件队列，
如果执行栈已经为空，那么将事件队列中的第一个回调函数放到栈中执行。
</pre>

比如下面的例子:
```js
<button id="doStuff">Do Stuff</button>
<script>
    document.getElementById('doStuff')
        .addEventListener('click', function() {
                console.log('Do Stuff');
            }
        );
</script>
```
我们再给出网上常见的关于WebAPI的图表:

![](./images/webapi.png)

下面给出上图中的概念的定义:

- Web APIs
  比如上面的click事件通过DOM的Web API进行传播，然后在冒泡和捕获阶段触发相应的父级以及子级DOM相应的click回调函数。Web APIs是浏览器中的`多线程`部分内容，它允许在同一时间触发多个事件。它们可以通过在页面加载完成的时候我们熟悉的`window`对象来访问。比如在window上的用于ajax访问的`XMLHttpRequest`对象以及定时器的`setTimeout`函数。
- [事件队列](https://www.w3.org/TR/html5/webappapis.html#task-queues)
  每一个事件的回调函数都会被推入到一个或者多个事件队列。就像浏览器有很多Web APIs一样，浏览器也会有多个事件队列，比如:网络请求队列,DOM事件队列,UI渲染队列等。它本身是任务的有序列表，它负责的内容包括:Events,Parsing,Using a resource,Reacting to DOM manipulation
- [事件循环](https://www.w3.org/TR/html5/webappapis.html#event-loops)
  事件循环会选择将那个JS的回调函数推入到执行栈中执行。下面是火狐中C++伪代码实现的事件循环。
```js
while(queue.waitForMessage()){
    queue.processNextMessage();
}
```
此时，事件回调函数将会进入浏览器的`JS执行环境`中执行。事件循环是为了协调:事件回调执行,用户交互，脚本执行，页面渲染,网络请求等等。

##### 7.2 JS中的调用栈及特点
JS执行环境:JS引擎包括很多部分:比如加载的js的解析，用于对象内存分配的`堆`，垃圾回收系统，解析器，以及用于执行事件处理函数的`栈`等。下面具体讲下栈的内容。
每一个函数,包括事件回调在执行的时候会创建一个`栈帧`(也叫做执行对象)。这些栈帧会被从调用栈的顶部推入或者推出，在顶部的栈帧表示当前执行的代码。当一个执行的函数返回后，栈帧将会从栈中被推出。下面是Chrome的V8中单个栈帧的代码:
```js
    /**
   *  v8.h line 1372 -- A single JavaScript stack frame.
   */
  class V8_EXPORT StackFrame {
   public:
      int GetLineNumber() const;
      int GetColumn() const;
      int GetScriptId() const;
      Local<String> GetScriptName() const;
      Local<String> GetScriptNameOrSourceURL() const;
      Local<String> GetFunctionName() const;
      bool IsEval() const;
      bool IsConstructor() const;
  };
```
其中栈帧有三个显著的特点:
- 单线程
  线程是CPU使用的基本单元。作为OS的底层实现，它包括线程ID，程序计数器(program counter),寄存器组(register set)以及调用栈。虽然，`JS引擎本身是多线程的？(待确认)`，但是调用栈却是单线程的，也就是说在同一时间只有一份代码在执行。
- 同步的
  JS通过调用栈来完成一个个的任务，而不是采用任务切换(Task switching)，这对于事件也是一样的。这不是ECMAScript或者WC3强制规定的。但是，这也有例外，比如`window.alert`就会`打断当前的执行任务`。
- 非阻塞的
  阻塞发生在当一个线程在执行的时候当前应用状态被挂起。`浏览器的调用栈是非阻塞的`，它能够继续接受事件，比如鼠标点击事件，虽然他们并没有立即执行。比如下面CPU密集型的例子，当调用栈代码在执行的时候依然可以向里面添加事件处理函数。

##### 7.3 浏览器处理CPU密集型任务
CPU密集型的任务有点稍微复杂，因为单线程+同步的执行环境使得我们必须排队执行所有的回调函数，此时我们的线程进入等待状态，比如`UI线程`。比如下面的例子:
```js
<button id="bigLoop">Big Loop</button>
<button id="doStuff">Do Stuff</button>
<script>
document.getElementById('bigLoop')
    .addEventListener('click', function() {
        //  big loop
        for (var array = [], i = 0; i < 10000000; i++) {
            array.push(i);
        }
    });

document.getElementById('doStuff')
    .addEventListener('click', function() {
        //  message
        console.log('do stuff');
    });
<\/script>
```
在点击了`Big Loop` 后再点击`Do Stuff`。当`Big Loop` 回调函数执行的时候浏览器仿佛冻结了一样。我们知道JS的调用栈是同步的，因此必须要等待`Big Loop`执行完毕。而且，调用栈是非阻塞的，因为:`Do Stuff`的点击事件依然能够被接收到，虽然它们并没有立即执行。这也就是告诉我们:`'事件触发是异步的，但是在调用栈中的回调执行却是同步的'`。CPU密集型的操作可以修改为如下:
```js
 document.getElementById('bigLoop')
  .addEventListener('click', function() {
      var array = []
      // smaller loop
      setTimeout(function() {
           for (i = 0; i < 5000000; i++) {
               array.push(i);
           }
      }, 0);
      // smaller loop
      setTimeout(function() {
           for (i = 0; i < 5000000; i++) {
               array.push(i);
           }
      }, 0);
  });
```
`setTimeout()`在WebAPI中执行，然后将回调函数推送到事件队列中，进而可以使得事件循环在将回调函数推送到js执行栈之前进行重新渲染页面([repaint](https://www.w3.org/TR/html5/webappapis.html#event-loops)，因为`事件循环会从多个任务队列中选择具体应该执行的回调，包括DOM queue队列的操作。所以两个setTimeout执行间隙可能插入其他的回调执行`)。当然，对于CPU密集型也可以使用web worker。请关注[Events, Concurrency and JavaScript](https://danmartensen.svbtle.com/events-concurrency-and-javascript)原文与参考文献。

#### 8.JS语言层面实现了异步?
答案是:"No"!事实上，ECMAScript`并没有`从语言上约定其异步的特性，我们所探讨的“异步”都是由`执行引擎`所赋予的。于Firefox，这个引擎是SpiderMonkey，于Node.js这个引擎是V8。而提供这个异步能力的`机制`，则是我们所谓的Event Loop——事件轮询。所以像setTimeout，setInterval这样的函数，实际上并不是由语言本身所约定的，而是`浏览器/执行引擎`来实现，向JavaScript暴露的、提供的异步入口。

因此，`异步与单线程`并没有出现矛盾。而具体到浏览器端，每个跃然于我们屏幕之前的Tab页，都拥有一个JS执行线程，即：
<pre>
There is only one JavaScript thread per window. 
</pre>

页面上虽然只提供了一个`JavaScript Call Stack`用于执行代码，不过浏览器在内部还实现了一个或多个队列，借由`事件轮询的机制`来调度全部事件的处理，而且在一定程度上，Programmer有权access到这个内部的轮询中。其一，可以是Timer函数，其二，则可以是通过DOM事件。而在浏览器中，`UI Rendering与JS Call Stack共用了线程`，轮询机制由浏览器内建；而Node.js中，轮询则由libuv提供的，并且libuv建立了针对不同kernel的抽象，封装了更多IO有关的具体的处理场景以及woker线程，这也解释了为什么Node.js单节点拥有高负载的原因。

#### 9.什么是JS执行环境(Runtimes)与浏览器宿主环境
##### 9.1 基本概念
JS执行环境，比如V8，有一个堆(用于内存分配)和一个栈(执行环境)。但是执行环境没有`setTimeout`,`DOM`等等,这些API都是在浏览器中提供的。

在浏览器中的JS有以下部分:
- JS执行环境
  比如V8(包括堆和栈)
- Web API
  浏览器提供的Web API,比如DOM，ajax,setTimeout等
- 回调队列
  包括一个回调队列用于执行事件的回调，比如onClick,onLoad,onDone等等
- 事件循环 

比如下图:

![](./images/chrome.png)

JS的执行环境在同一个时间只能执行一份代码，在执行其他代码的时候不能发出ajax请求，也不能执行setTimeout。但是，在浏览器中我们可以并发操作，因为浏览器比执行环境Runtime包含的部分要多得多(包括webApi等独立的线程)!

##### 9.2 JS执行环境与UI渲染
浏览器会受到执行的JS的限制，它会每隔16.6ms进行一次重绘(repaint,60帧每秒)。但是，如果在执行栈中有代码正在执行，那么它就无法正常的进行页面渲染。

<pre>
  When people say “don’t block the event loop”, this is exactly what they’re talking about. Don’t put slow code 
  on the stack because, when you do that, the browser can’t do what it needs to do, like create a nice fluid UI.
</pre>。

当我们讨论不要`阻塞事件循环`的时候，其实是在讨论不要在执行栈中放置耗时代码，这种情况下浏览器不能做它本来应该做的事情，比如创建一个流畅的UI界面。

比如页面频繁滚动的时候，事件处理函数执行会出现卡顿。而一个好的方法就是使用防抖，比如间隔多少秒才执行等等。原文[JavaScript's Call Stack, Callback Queue, and Event Loop](http://cek.io/blog/2015/12/03/event-loop/) 阅读。

#### 10.JavaScript的内存管理
##### 10.1 栈内存与堆内存
JavaScript中的变量分为基本类型和引用类型。基本类型就是保存在栈内存中的简单数据段，而引用类型指的是那些保存在堆内存中的对象。              
- 1、基本类型 
    基本类型有Undefined、Null、Boolean、Number 和String。这些类型在内存中分别`占有固定大小的空间`，他们的值保存在栈空间，我们通过按值来访问的。              
- 2、引用类型
    引用类型，值大小不固定，`栈内存中存放地址指向堆内存中的对象`。是`按引用访问`的。如下图所示：栈内存中存放的只是该对象的访问地址，在堆内存中为这个值分配空间。由于这种值的大小不固定，因此不能把它们保存到栈内存中。但`内存地址大小`是固定的，因此可以将内存地址保存在栈内存中。这样，当查询引用类型的变量时， `先`从栈中读取内存地址， `然后`再通过地址找到堆中的值。对于这种，我们把它叫做`按引用访问`。

![](./images/stc.png)

那么为什么会有栈内存和堆内存之分？

通常与垃圾回收机制有关。为了使程序`运行时占用的内存最小`。当一个方法执行时，[每个方法都会建立自己的内存栈](https://glebbahmutov.com/blog/javascript-stack-size/)，在这个方法内定义的变量将会逐个放入这块栈内存里，随着方法的执行结束，这个方法的内存栈也将自然销毁了。因此，所有在方法中定义的变量都是放在栈内存中的(如果是对象，那么保存在堆中，然后`栈中中保存的是一个引用`)；当我们在程序中创建一个对象时，这个对象将被保存到运行时数据区中，以便`反复利用`（`因为对象的创建成本通常较大`），这个运行时数据区就是堆内存。堆内存中的对象不会随方法的结束而销毁，即使方法结束后，这个对象还可能被另一个引用变量所引用（方法的参数传递时很常见），则这个对象依然不会被销毁，只有当一个对象没有任何引用变量引用它时，系统的垃圾回收机制才会在合适的时候回收它。文字转载[JavaScript变量——栈内存or堆内存](http://blog.csdn.net/xdd19910505/article/details/41900693)

同时，堆栈的区分能够让代码执行更加安全(stack is more protected)同时也更加快(不需要动态栈帧的垃圾回收，而只是创建新的栈帧)。比如下面的例子:
```js
function foo() { var a = 1; }
function bar() { var b = 2; foo(); }
bar();
```
即使我们的方法是递归调用，那么每一个栈帧也会有自己的一份独立的本地变量。当函数执行完毕以后，该栈帧就会从栈中被移除，释放本地变量的内存分配。这也是为什么类似于C++，C的语言不需要去考虑释放本地变量的原因。

##### 10.2 内存的回收策略
- 引用计数的循环引用
  引用计数的机制为:(1)当我们创建了一个对象，然后将它存储到一个变量中，那么该对象的引用数量就是1，而当它的引用又被用于另外一个变量或者函数中的时候，它的引用数量就是2。(2)如果我们将使用变量的引用值的变量设置为一个新的值，那么原来的变量和使用引用变量的引用数就是1。(3)如果最后对象被置空了，那么引用数量就是0。下面是示例代码:
```js
var a = {obj:{name:"my_name"}};
var b = a;
a = 1;
//Here we created an object, which is used by variable a and b
// we have 1 to a so one of objects reference is reduced to 1
// still we have one reference which is b
b = null;
// now object has not any reference left, garbage collector will take this object.
```
而引用计数算法可能出现循环引用，最终两者都无法经过垃圾回收器进行回收。
```js
function f() {
  var o1 = {};
  var o2 = {};
  o1.p = o2; // o1 references o2
  o2.p = o1; // o2 references o1. This creates a cycle.
}
f();
```
- 标记清除

#####  10.3 内存泄露的方式
- 意外的全局变量
  即使我们讨论了不可预测的全局变量，但是仍有一些明确的全局变量产生的垃圾。这些是根据定义不可回收的（除非被取消或重新分配）。特别地，用于临时存储和处理大量信息的全局变量是令人关注的。 如果必须使用全局变量来存储大量数据，请确保将其置空或在完成后重新分配它。与全局变量有关的增加的内存消耗的一个常见原因是高速缓存）。缓存存储重复使用的数据。 为了有效率，高速缓存必须具有其大小的上限。 无限增长的缓存可能会导致高内存消耗，因为缓存内容无法被回收。
- 被遗忘的计时器或回调函数
  比如下面的例子:
```js
var someResource = getData();
  setInterval(function() {
    var node = document.getElementById('Node');
    if(node) {
      // Do stuff with node and someResource.
      node.innerHTML = JSON.stringify(someResource));
    }
  }, 1000);
```
此示例说明了挂起计时器可能发生的情况：引用不再需要的节点或数据的计时器。 由节点表示的对象可以在`将来被移除`，使得区间处理器内部的整个块不需要了。但是，处理程序（因为时间间隔仍处于活动状态）无法回收（需要停止时间间隔才能发生）。 如果无法回收间隔处理程序，则也无法回收其依赖项。这意味着someResource，它可能存储大小的数据，也不能被回收。解决方法就是在DOM移除的时候清除定时器。

对于观察者的情况，重要的是进行显式调用，以便在不再需要它们时删除它们（或者相关对象即将无法访问）。 在过去，以前特别重要，因为某些浏览器（Internet Explorer 6）不能管理循环引用（参见下面的更多信息）。 现在，一旦观察到的对象变得不可达，即使没有明确删除监听器，大多数浏览器也可以回收观察者处理程序。 然而，在对象被处理之前显式地删除这些观察者仍然是良好的做法。 例如：
```js
 var element = document.getElementById('button');
  function onClick(event) {
    element.innerHtml = 'text';
  }
  element.addEventListener('click', onClick);
  // Do stuff
  element.removeEventListener('click', onClick);
  element.parentNode.removeChild(element);
  // Now when element goes out of scope,
  // both element and onClick will be collected even in old browsers that don't
  // handle cycles well.
```
- 脱离DOM的引用要置空
有时，将DOM节点存储在数据结构中可能很有用。 假设要快速更新表中多行的内容。 在字典或数组中存储对每个DOM行的引用可能是有意义的。当发生这种情况时，会保留对同一个DOM元素的`两个引用`：一个在DOM树中，另一个在字典中。 如果在将来的某个时候，您决定删除这些行，则需要使这两个引用不可访问。
```js
  var elements = {
    button: document.getElementById('button'),
    image: document.getElementById('image'),
    text: document.getElementById('text')
  };
  function doStuff() {
    image.src = 'http://some.url/image';
    button.click();
    console.log(text.innerHTML);
    // Much more logic
  }
  function removeButton() {
    // The button is a direct child of body.
    document.body.removeChild(document.getElementById('button'));
    // At this point, we still have a reference to #button in the global
    // elements dictionary. In other words, the button element is still in
    // memory and cannot be collected by the GC.
  }
```
对此的另外考虑与对DOM树内的内部或叶节点的引用有关。假设您在JavaScript代码中保留对表的特定单元格（标记）的引用。 在将来的某个时候，您决定从DOM中删除表，但保留对该单元格的引用。直观地，可以假设GC将回收除了该单元之外的所有东西。在实践中，这`不会发生`:单元格是该表的子节点，并且`子级保持对其父级的引用`。 换句话说，从JavaScript代码对表单元格的引用导致整个表保留在内存中。 在保持对DOM元素的引用时仔细考虑这一点。下面再给出一个类似的内存泄露的例子:
```html
<html>
    <body>
        <div id="refA">
            <ul>
                <li><a></a></li>
                <li><a></a></li>
                <li><a id="#refB"></a></li>
            </ul>
        </div>
        <div></div>
        <div></div>
    </body>
</html>

<script>
    var refA = document.getElementById('refA');
    var refB = document.getElementById('refB');//refB引用了refA。它们之间是dom树父节点和子节点的关系。
</script>
```
现在，问题来了，如果我现在在dom中移除div#refA会怎么样呢？答案是dom内存依然存在，因为它被js引用。那么我把refA变量置为null呢？答案是内存依然存在了。因为refB对refA存在引用，所以除非在把refB释放，否则dom节点内存会一直存在浏览器中无法被回收掉。

![](./images/clu.png)

在上图,红色的虽然已经不再DOM树中了，但是其依然占据着内存，所以称为detached Dom Nodes。
下面是闭包导致的内存泄漏问题：
```js
function a() {
    var obj = [1,2,3,4,5,6];
    return function Test() {
        //js作用域的原因，在此闭包运行的上下文中可以访问到obj这个对象
        console.log(obj);
    }
}
//正常情况下，a函数执行完毕obj占用的内存会被回收，但是此处a函数返回了一个函数表达式（见Tom大叔的博客函数表达式和函数声明），其中obj因为js的作用域的特殊性一直存在，所以我们可以说b引用了obj。
var b = a();
//每次执行b函数的时候都可以访问到obj，说明内存未被回收 所以对于obj来说直接占用内存[1,2,....n], 而b依赖obj，所obj是b的最大内存。
b()
```
在chrome调试工具中可以查看所有自定义的函数等，也可以通过这个视图查找我们写的闭包，如下面的函数的context属性里面有一个closure:

![](./images/clu1.png)

而且我们知道：在每一次`snapshot`的时候都是会提前GC的，所以如果某个元素已经被GC掉那么不会出现在上面的列表中，然而上面的我们的a.b变量都是存在的，所以他们根本没有被GC掉，a函数我们就不讲了，因为他是全局函数，然后我们压根不希望b长久存在，因此我们必须手动解除引用b=null!,这时候我们的b变量就变成了：

![](./images/clu2.png)

这就是告诉我们`全局变量`(只是一个变量)只要能够清除掉那么我们都应该清除掉，而`全局函数`只有在页面卸载的时候被清除，当然是否可以注册onunload事件呢!


- 闭包
JavaScript开发的一个关键方面是闭包：从父作用域捕获变量的匿名函数。 Meteor开发人员发现了一个特定的情况，由于JavaScript运行时的实现细节，可能以一种微妙的方式泄漏内存：
```js
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  var unused = function () {
    if (originalThing) // a reference to 'originalThing'
      console.log("hi");
  };
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log("message");
    }
  };
};
setInterval(replaceThing, 1000);
```
这个片段做了一件事：`每次`replaceThing被调用，theThing获取一个新的对象，其中包含一个大数组和一个新的闭包（someMethod）。同时，unused变量保持一个闭包，该闭包具有对originalThing的引用（来自之前对replaceThing的调用的Thing）。已经有点混乱了，是吗？重要的是，一旦为同一父作用域中的闭包创建了作用域，则该作用域是共享的。在这种情况下，为闭包someMethod创建的作用域由unused共享。unused的引用了originalThing。即使unused未使用，可以通过theThing使用someMethod。由于someMethod与unused共享闭包范围，即使未使用，它`对originalThing的引用强制它保持活动（防止其收集）`。当此代码段重复运行时，可以观察到内存使用量的稳定增加。这在GC运行时不会变小。实质上，创建一个闭包的链接列表（其根以theThing变量的形式），并且这些闭包的范围中的每一个都包含对大数组的间接引用，导致相当大的泄漏。原文[点击这里阅读](How JavaScript works: memory management + how to handle 4 common memory leaks)。文中还提到了两种在`堆中分配内存与在栈中分配内存`的区别,如那些内存是从堆中获取的，而那些是从栈中直接获取的,即所谓的动态分配内存与静态分配内存:

![](./images/allocate.png)

在栈中直接分配内存的特点有:分配的大小在编译的时候就能够确定;在编译的时候内存就能够分配完成;内存为栈所拥有;采用先进后出的方式

在堆中直接分配内存的特点有:分配的大小在编译的时候是未知的;在代码执行的时候进行内存分配;内存为堆所持有;分配的顺序无固定式。

##### 10.4 垃圾回收机制(GC)是否会清除栈中数据
答案是:\`不会!\` 当一个函数执行的时候，它会在栈中添加很多自有的状态数据，而当`函数执行结束`，这些自有的数据将会从栈中被移除掉。而且,栈中的数据采用的是先进后出的原则，因此分配起来很简单，同时也比基于堆的内存分配(动态分配)更加快速。但是，在一些CPU中，线程分配的栈的大小可能会非常小，此时就会出现我们常见的栈内存溢出!一个极端的例子就是死循环的时候，循环的每一次都会有函数被压入栈中，而每一个函数都会消耗掉栈中的一部分内存，最后导致整个程序异常退出。关于栈内存回收，你可以[点击这里](https://en.wikipedia.org/wiki/Stack-based_memory_allocation)。

##### 10.5 栈中存储基本数据也能共享内存
比如下面的例子:
```js
var a=3;
var b=3;
a=5;
b
```
执行a=3,b=3的时候，a=3执行时为3在栈中分配了内存，那么b=3的时候不会在栈中分配内存存储3这个值，而是让b去指向已有的3，当a=5的时候，程序去寻找栈中有没有5这个值，如果有则让a去指向5，如果没有则重新分配内存存储5。显示在上面的例子中，a=5重新分配了内存，a此时指向了5，而b指向的值是3，并不会因为a的值的改变而改变。下面的例子也是同样的道理:
```js
var a = "apple";
var b = a;
//此时a,b指向栈中的同一个地址
a = "banana";
//此时栈中为a单独创建了一个值，为"banana"，而b还是指向栈中原来的地址
b
```
和下面的代码结果一致:
```js
var a = new String("apple");
var b = a;
//此时a,b指向栈中的同一个地址
a = new String("banana");
//此时栈中为a单独创建了一个值，为"banana"，而b还是指向栈中原来的地址
b
```
同时下面给出基本类型的生命周期过程:
```js
let str = 'Jack';
let oStr = str.substring(2);
// String对象被销毁了，返回的是一个全新的string，原来的值并没有发生改变
```
第二行代码，访问 str时，访问过程处于读取模式，也就是会从栈内存中读取这个字符串的值，在读取过程中，会进行以下几步：
<pre>
1.创建一个String类型的一个实例；
2.在实例上调用相应的方法。
3.销毁这个实例。
</pre>

基本包装函数，与引用类型主要区别就是对象的生存期，使用new操作符创建的引用类型的实例，在执行流离开当前作用域之前一直都保存在内存中，而自动创建的`基本包装类型的对象`，则只存在与一行代码的执行`瞬间`，然后被[立即销毁](https://www.zhangshengrong.com/p/boNwr5jkaw/)。这也就是不能给基本类型添加属性和方法的原因了。但是通过显示的包装基本类型却可以添加属性和方法:
```js
var s1="some text"
s1.color="red";
// 执行后添加了color属性的这个对象已经被销毁了
alert(s1.color);
//undefined
```
在此，第二行代码试图为字符串s1添加一个color属性。但是，当第三行代码在此访问s1时，其color属性不见了。问题的原因就是第二行创建的String对象在执行第三行代码时已经被销毁了。第三行代码又创建自己的String对象，而该对象没有color属性。但是下面的代码就可以:
```js
var s1=new String("some text")
s1.color="red";
console.log(s1.color);
//red
```
其实下面[这个例子](http://laichuanfeng.com/study/javascript-immutable-primitive-values-and-mutable-object-references/)也是同样的道理:
```js
var s = "hello"; 
//定义一个由小写字母组成字符串
s.toUpperCase(); 
//=>“HELLO”，但并没有改变s的值。首先构造一个new String('hello')然后调用它的toUpperCase方法
//得到一个新的对象后，因为没有栈中的值对它引用，所以垃圾回收机制能够立即清除它
//但是原始的s值并没有改变，因为操作的是new String而不是s本身
s;
//=>“hello”：原始字符串的值并未改变。
```
而且，在我看来，不管是隐式的产生基本类型包装对象还是显式的产生，他们都会在`堆`中被分配内存空间，而不是在`栈`中!只是隐式的这种方式在调用后会将产生的基本类型包装对象立即设置为null(比如toUpperCase的例子没有栈中的变量对它进行引用)，从而可以立即`通过垃圾回收机制回收内存`!而显式的这种方式因为存在对于基本类型包装对象的引用，所以无法立即通过`垃圾回收机制回收内容`!


#### 11.异步的回调函数通过闭包保存变量或者函数
本章节我们主要围绕了浏览器的事件循环，比如时间处理函数，setTimeout，ajax等，那么这些回调函数被推入到执行栈中被执行的时候它能获取到那些变量呢？我们给出如下的一个例子:
```js
'use strict'
const express = require('express')
const superagent = require('superagent')
const app = express()
app.get('/', sendWeatherOfRandomCity)
function sendWeatherOfRandomCity (request, response) {
  getWeatherOfRandomCity(request, response)
  sayHi()
}
const CITIES = [
  'london',
  'newyork',
  'paris',
  'budapest',
  'warsaw',
  'rome',
  'madrid',
  'moscow',
  'beijing',
  'capetown',
]
function getWeatherOfRandomCity (request, response) {
  const city = CITIES[Math.floor(Math.random() * CITIES.length)];
  superagent.get(`wttr.in/${city}`)
    .end((err, res) => {
      // 该回调函数中的私有变量都会被随着执行上下文的销毁而回收!
      // 执行上下文销毁后外部变量的引用city等也会销毁
      if (err) {
        console.log('O snap')
        return response.status(500).send('There was an error getting the weather, try looking out the window')
      }
      const responseText = res.text
      response.send(responseText)
      console.log('Got the weather')
    })
  console.log('Fetching the weather, please be patient')
}
function sayHi () {
  console.log('Hi')
}
app.listen(3000)
```
当访问`/`这个URL的时候会调用superagent.get(`wttr.in/${city}`)方法，此时end回调函数会被添加到回调队列中，当回调真正完成的时候会将该函数推入到执行栈中被执行。因为**闭包**的存在，该回调函数可以访问到:express, superagent, app, CITIES, request, response, city变量以及我们定义的函数。这也就是告诉我们:虽然sendWeatherOfRandomCity和getWeatherOfRandomCity函数调用已经完毕并return,但是当end回调函数被推入到栈中被执行的时候，它依然可以访问外部的变量，从另一方面来说sendWeatherOfRandomCity和getWeatherOfRandomCity函数调用完毕并return没法销毁如city,superagent等这一类的变量，即内存无法回收,这是**闭包**的作用!而当该函数在调用栈中被执行以后，其会从栈中被pop up出来，进而可以**销毁其执行上下文，从而回收该栈帧占用的所有的内存空间，包括对外部变量的引用/局部变量**，而该过程是由栈的特性自动完成内存回收的!



参考资料:

[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

[什么是浏览器的事件循环（Event Loop）？](https://segmentfault.com/a/1190000010622146)

[JS 单线程和事件循环](https://www.cnblogs.com/SamWeb/p/6434889.html)

[精读《Javascript 事件循环与异步》](https://zhuanlan.zhihu.com/p/30744300)

[How JavaScript works: Event loop and the rise of Async programming + 5 ways to better coding with async/await](https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5)

[How JavaScript works: an overview of the engine, the runtime, and the call stack](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

[How JavaScript works: inside the V8 engine + 5 tips on how to write optimized code](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)

[How JavaScript works: memory management + how to handle 4 common memory leaks](https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec)

[机器码和字节码](https://www.cnblogs.com/qiumingcheng/p/5400265.html)

[V8 Hidden Class](https://www.w3ctech.com/topic/660)

[什么是JavaScript Binding？](http://blog.csdn.net/yl02520/article/details/20720167)

[Web API](https://en.wikipedia.org/wiki/Web_API)

[Events, Concurrency and JavaScript](https://danmartensen.svbtle.com/events-concurrency-and-javascript)

[event-loops事件循环](https://www.w3.org/TR/html5/webappapis.html#event-loops)


[w3c的浏览器草案](https://www.w3.org/TR/html5/webappapis.html#microtask)

[关于浏览器处理事件的问题?](https://www.zhihu.com/question/30970837)

[Understanding Javascript Function Executions — Call Stack, Event Loop , Tasks & more — Part 1](https://medium.com/@gaurav.pandvia/understanding-javascript-function-executions-tasks-event-loop-call-stack-more-part-1-5683dea1f5ec)

[JavaScript's Call Stack, Callback Queue, and Event Loop](http://cek.io/blog/2015/12/03/event-loop/)

[4种JavaScript内存泄漏浅析及如何用谷歌工具查内存泄露](https://github.com/wengjq/Blog/issues/1)

[Reference-counting garbage collection in Javascript](http://findnerd.com/list/view/Reference-counting-garbage-collection-in-Javascript/6196/)

[JavaScript stack size](https://glebbahmutov.com/blog/javascript-stack-size/)

[Stack-based memory allocation](https://en.wikipedia.org/wiki/Stack-based_memory_allocation)

[How does a “stack overflow” occur and how do you prevent it?](https://stackoverflow.com/questions/26158/how-does-a-stack-overflow-occur-and-how-do-you-prevent-it)

[Stack-based_memory_allocation](https://en.wikipedia.org/wiki/Stack-based_memory_allocation)

[javascript中变量重新赋值和引用重新赋值问题](http://www.cnblogs.com/songxiaochen/p/7738167.html)

[浅谈javascript中基本包装类型](https://www.zhangshengrong.com/p/boNwr5jkaw/)

[Philip Roberts: What the heck is the event loop anyway?](https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html)

[作用域链与执行上下文](https://github.com/mqyqingfeng/Blog/issues/8)
