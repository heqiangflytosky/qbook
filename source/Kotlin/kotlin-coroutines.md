---
title: Kotlin -- 协程
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 协程的使用
date: 2019-12-22 10:00:00
---

## 概述

学习 Kotlin，就必须提一下协程，协程其实就是 Kotlin 里面的线程框架，它是 Kotlin 中非常重要的一部分，如果你不知道协程，那就说明你还没有学习过 Kotlin。
在 Kotlin 1.1 版本中引入了协程（Coroutines），目前协程还是实验性功能，因此使用前还要在 gradle 里面加上依赖：

```
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.2"
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.2'
```

它是将复杂的异步操作放入底层库中，程序逻辑可以在协程中按照顺序来表达，以此简化异步编程，该底层库将用户代码包装为回调/订阅事件，在不同线程或者不同机器调度执行。

## 协程的特点

协程主要有以下特点：

 - 协程挂起：协程提供了一种使程序避免阻塞且更廉价可控的操作，叫作协程挂起（coroutines suspension），协程挂起是非阻塞式挂起，不阻塞线程。
 - 简化代码：协程让原来使用“异步+回调”方式写出来的复杂代码，简化成可以用看似同步的方式表达。也就是说用看似同步的方式写出异步的代码。
 - 节约资源：线程是由系统调度的，线程切换或线程阻塞的开销都比较大。而协程依赖于线程，但是协程挂起时不需要阻塞线程，几乎是无代价的。协程是由开发者控制的。所以协程也像用户态的线程，非常轻量级，一个线程中可以创建任意多个协程。
 - 支持取消：
 - 协程是结构化的：也就是说在协程作用域内部创建的子协程，它们之间是有父子关系的。而线程是没有这种特性的。这种特性使得协程便于管理。

Kotlin 协程是 Kotlin 的线程框架，是对线程的一种更友好的封装，主要特点是使用方便，方便在哪里？可以使用同步的方式写出异步的代码。这个我感觉是协程带给我们最大的收获。
Kotlin 还是运行在线程中的，因此协程不能代替线程。
协程的挂起其实就是这个协程从正在执行它的线程上脱离了，而不是暂停了。这个协程当前所在的线程从这行代码开始就不再运行这个协程了。当挂起的函数执行完毕后，会自动切回到当前的协程。
协程与线程最大的区别在于，从任务的角度来看，线程一旦开始执行就不会暂停，直到任务结束，这个过程都是连续的。线程之间是抢占式的调度，因此不存在协作问题。
其实，Java 虽然没有实现协程，但开发者可以通过第三方框架提供协程的能力，例如Java的协程框架Quasar。

## 协程作用域

协程作用域定义新协程的范围。每个协程构建器都是CoroutineScope的扩展，并继承其coroutineContext以自动传播上下文元素和取消。
每个协程构建器（如launch，async等）和每个作用域函数（如coroutineScope，withContext等）都会将自己的作用域实例提供给它运行的内部代码块。
我们前面在构建协程时用的 `GlobalScope` 就是继承自 `CoroutineScope` 的作用域，另外 kotlin 还为我们提供了 MainScope 作用域来构建协程。

coroutineScope 是继承外部 Job 的上下文创建作用域，在其内部的取消操作是双向传播的，子协程未捕获的异常也会向上传递给父协程。它更适合一系列对等的协程并发的完成一项工作，任何一个子协程异常退出，那么整体都将退出，简单来说就是”一损俱损“。这也是协程内部再启动子协程的默认作用域。
supervisorScope 同样继承外部作用域的上下文，但其内部的取消操作是单向传播的，父协程向子协程传播，反过来则不然，这意味着子协程出了异常并不会影响父协程以及其他兄弟协程。它更适合一些独立不相干的任务，任何一个任务出问题，并不会影响其他任务的工作，简单来说就是”自作自受“，例如 UI，我点击一个按钮出了异常，其实并不会影响手机状态栏的刷新。需要注意的是，supervisorScope 内部启动的子协程内部再启动子协程，如无明确指出，则遵守默认作用域规则，也即 supervisorScope 只作用域其直接子协程。

## 协程的使用

### suspend

先来介绍一下 suspend 关键字，用该关键字修饰的方法可以被协程挂起，如果在协程外部，只有在该关键字修饰的方法中才可以使用 `delay` `Deferred.await` 等方法。
被 suspend 修饰的方法执行运行在协程内部或者另一个 suspend 方法中，suspend 关键字其实不是真正的执行挂起动作，挂起动作是由 suspend 方法内部调用的kotlin自动的挂起函数来执行挂起动作的。suspend 关键字可以理解为只是一个标识，提醒调用者这是一个耗时的函数，需要用挂起的方式放在后台执行。

### 创建协程

可以通过下面的方法创建一个协程

 - GlobalScope.launch
 - GlobalScope.async
 - runBlocking

#### GlobalScope.launch

我们可以通过 `GlobalScope.launch` 启动一个协程。

```
        GlobalScope.launch {
            Log.e("Test","Coroutine start")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue")
```

 `delay` 是一个特殊的挂起函数，它不会造成线程阻塞，但是会挂起协程，并且只能在协程或者 suspend 方法修饰的方法中使用。
 
 ```
 public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {

}
 ```
 
`launch` 方法接受三个参数：

 - context: CoroutineContext：协程的上下文，可以设置协程运行的线程调度器，有 4种模式可以选择：
  - Dispatchers.Default：默认为子线程运行
  - Dispatchers.IO：子线程运行
  - Dispatchers.Main：主线程运行
  - Dispatchers.Unconfined：没指定的情况下就是运行在当前创建协程的线程
 
 ```
        GlobalScope.launch(Dispatchers.Main) {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
 ```
 或者我们可以自己创建线程池，通过 `newSingleThreadContext` 或者 `newFixedThreadPoolContext` 来创建。

 - start: CoroutineStart：启动模式，可以有下面四种模式：
  - DEAFAULT：默认模式，也就是创建就启动
  - ATOMIC：
  - UNDISPATCHED：
  - LAZY：懒加载模式，你需要它的时候再调用启动
 
 ```
        var job:Job = GlobalScope.launch( start = CoroutineStart.LAZY) {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue")
        Thread.sleep(1000);
        job.start()
 ```
 - block：闭包方法体，定义协程内需要执行的操作

launch 方法返回一个 Job 类型的对象，Job 类提供了下面的常用方法：

 - start()：启动协程，除了 lazy 模式之外都不需要手动启动
 - cancel()：取消协程
 - join()：等待协程执行完毕

#### GlobalScope.async

`async` 方法和 `launch` 方法使用类似，唯一的区别就是返回值的类型不同。

```
        GlobalScope.launch {
            var deffer = GlobalScope.async {
                Log.e("Test","Coroutine start ${Thread.currentThread()}")
                delay(1000)
                Log.e("Test","Coroutines end")
                return@async 200;
            }
            Log.e("Test","continue ")
            Log.e("Test","result = ${deffer.await()} ")
        }
```

```
17:25:19.388 16973 17003 E Test    : continue 
17:25:19.388 16973 17004 E Test    : Coroutine start Thread[DefaultDispatcher-worker-4,5,main]
17:25:20.393 16973 17002 E Test    : Coroutines end
17:25:20.398 16973 17008 E Test    : result = 200
```

async 返回的是 Deferred 类型，Deferred 继承自 Job 接口，增加了一个 await 方法，这个方法只能在协程内部使用。async 的特点是不会阻塞当前线程，但会阻塞所在协程，也就是挂起。

#### runBlocking

runBlocking 方法和 launch 方法的区别是 runBlocking 是可以阻塞当前的线程的。

```
        runBlocking {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue ")
```

```
Test    : Coroutine start Thread[main,5,main]
Test    : Coroutines end
Test    : continue 
```

#### withContext

withContext {} 在指定协程上运行挂起代码块，并挂起当前协程直至代码块运行完成再切回来运行。
多个 withContext 任务是串行的， 且 withContext 可直接返回任务执行的结果。

### withContext 和 async 的区别

withContext 与 async 都可以返回耗时任务的执行结果。 
多个 withContext 任务是串行的， 且 withContext 可直接返回任务执行的结果。
多个 async 任务是并行的，async 返回的是一个 `Deferred<T>`，需要调用其 `await()` 方法获取结果。

```
        GlobalScope.launch (Dispatchers.Main){
            var r1 = withContext(Dispatchers.IO) {
                Log.e("Test","enter 1")
                delay(1000)
                Log.e("Test","exit 1")
                100
            }
            var r2 = withContext(Dispatchers.IO) {
                Log.e("Test","enter 2")
                delay(1000)
                Log.e("Test","exit 2")
                200
            }
            Log.e("Test","r1=$r1,r2=$r2")
        }
```

结果：

```
10:42:28.973 : enter 1
10:42:29.977 : exit 1
10:42:29.980 : enter 2
10:42:30.991 : exit 2
10:42:30.992 : r1=100,r2=200
```

```
        GlobalScope.launch (Dispatchers.Main){
            var r1 = async (Dispatchers.IO) {
                Log.e("Test","enter 1")
                delay(1000)
                Log.e("Test","exit 1")
                100
            }
            var r2 = async(Dispatchers.IO) {
                Log.e("Test","enter 2")
                delay(1000)
                Log.e("Test","exit 2")
                200
            }
            Log.e("Test","r1=${r1.await()},r2=${r2.await()}")
        }
```

并行执行：

```
10:40:05.658 : enter 1
10:40:05.662 : enter 2
10:40:06.663 : exit 1
10:40:06.665 : exit 2
10:40:06.671 : r1=100,r2=200
```

如果我们把上面的程序修改一下，两个 async 也可以变为串行执行：

```
        GlobalScope.launch (Dispatchers.Main){
            var r1 = async (Dispatchers.IO) {
                Log.e("Test","enter 1")
                delay(1000)
                Log.e("Test","exit 1")
                100
            }
            Log.e("Test","r1=${r1.await()}")
            var r2 = async(Dispatchers.IO) {
                Log.e("Test","enter 2")
                delay(1000)
                Log.e("Test","exit 2")
                200
            }
            Log.e("Test","r1=${r1.await()},r2=${r2.await()}")
        }
```

```
2020-12-04 : enter 1
2020-12-04 : exit 1
2020-12-04 : r1=100
2020-12-04 : enter 2
2020-12-04 : exit 2
2020-12-04 : r1=100,r2=200
```

因此，因此直接在async 后面使用 await() 就和 withContext 一样，程序运行到这里就会被挂起直到该函数执行完成才会继续执行下一个 async 。

### 取消协程

#### 如何取消协程

`launch` 方法有个 Job 类型的返回值，Job 提供了 cancel 方法来取消协程的执行。

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

```
    fun testCancel() = runBlocking{
        var job = GlobalScope.launch {
            repeat(5) {
                Log.e("Test","hello : $it")
                delay(500)
            }
        }

        delay(1100)
        Log.e("Test","world")

        job.cancel()
        job.join()

        Log.e("Test","welcome")
    }
```

运行结果为：

```
    hello : 0
    hello : 1
    hello : 2
    world
    welcome
```

在delay 1100毫秒之后，由于在runBlocking协程（姑且称之）中调用了job.cancel()之后，launch协程（姑且称之）中原本会repeat 5次的执行，如今只计数到了2，说明的的确确被取消了。cancel()一般会和join()方法一起使用，因为cancel可能不会立马取消对应的协程（下面我们会提到，协程能够被取消，是需要一定条件的），所以会需要join()来协调两个协程。故而有个更加简便的方法：Job.cancelAndJoin()，可以用来替换上面的两行代码。

```
public suspend fun Job.cancelAndJoin() {
    cancel()
    return join()
}
```

#### 能够取消的条件


协程的取消是有条件的，只有当前协程代码是可取消的，cancel()才能起作用。

```
    fun testCancel() = runBlocking{
        val job = launch(context = Dispatchers.Default) {

            Log.e("Test","Current Thread : ${Thread.currentThread()}")
            var nextActionTime = System.currentTimeMillis()
            var i = 0

            while (i < 10) {
                if (System.currentTimeMillis() >= nextActionTime) {
                    Log.e("Test","Job: ${i++}")
                    nextActionTime += 500
                }
            }
        }

        delay(1300)
        Log.e("Test","hello")

        job.cancelAndJoin()

        Log.e("Test","welcome")
    }
```

运行结果：

```
Current Thread : Thread[DefaultDispatcher-worker-1,5,main]
Job: 0
Job: 1
Job: 2
hello
Job: 3
Job: 4
Job: 5
Job: 6
Job: 7
Job: 8
Job: 9
welcome

```

在delay 1300毫秒之后，我们调用了cancelAndJoin方法，但是协程没有取消成功，协程一直在运行，知道达到设置的循环次数。
为什么会这样呢？在空等500毫秒中，实际上可以看做是死循环了500毫秒，并且一直占用着cpu。而我们也没有检测协程的状态，因此它就没有自动取消。这个特点和Java线程的取消有点类似。

这个示例中的launch协程的代码是不可取消的。那么什么样的代码才可以视为可取消的呢？
kotlinx.coroutines包下的所有挂起函数都是可取消的。这些挂起函数会检查协程的取消状态，当取消时就会抛出CancellationException异常。如果协程正在处于某个计算过程当中，并且没有检查取消状态，那么它就是无法被取消的。

下面我们改进一下代码：

```
    fun testCancel() = runBlocking{
        val job = launch(context = Dispatchers.Default) {

            Log.e("Test","Current Thread : ${Thread.currentThread()}")
            var nextActionTime = System.currentTimeMillis()
            var i = 0

            while (isActive) {
                if (System.currentTimeMillis() >= nextActionTime) {
                    Log.e("Test","Job: ${i++}")
                    nextActionTime += 500
                }
            }
        }

        delay(1300)
        Log.e("Test","hello")

        job.cancelAndJoin()

        Log.e("Test","welcome")
    }
```

运行结果：

```
Current Thread : Thread[DefaultDispatcher-worker-1,5,main]
Job: 0
Job: 1
Job: 2
hello
welcome
```

可见，协程取消成功。

最后，我们对协程取消条件做一下总结：从某种角度上讲，是否能够取消是主动的；外部调用了cancel方法后，相当于是发起了一条取消信号；被取消协程内部如果自身检测到自身状态的变化，比如isActive的判断以及所有的kotlinx.coroutines包下挂起函数，都会检测协程自身的状态变化，如果检测到通知被取消，就会抛出一个CancellationException的异常。
另外还可以使用 `ensureActive()` 方法来判断当前协程是否被取消。

#### 取消异常

```
    fun testCancel() = runBlocking{
        var job = GlobalScope.launch {
            try{
                repeat(5) {
                    Log.e("Test","hello : $it")
                    delay(500)
                }
            } catch (e : CancellationException){
                Log.e("Test","CancellationException")
            } finally {
                Log.e("Test","finally")
            }
        }

        delay(1100)
        Log.e("Test","world")

        job.cancel()
        job.join()

        Log.e("Test","welcome")
    }
```

运行结果：

```
hello : 0
hello : 1
hello : 2
world
CancellationException
finally
welcome
```

我们使用 catch 块捕获了因取消协程而抛出的 CancellationException 异常，另外，可以再 finally 中做一些资源的关闭和回收工作。
这里有个问题，加入程序抛出了 CancellationException，那么为什么没有在输出终端见到它的身影呢？因为kotlin的协程是这样规定的：

```
That is because inside a cancelled coroutine CancellationException is considered to be a normal reason for coroutine completion.
```

也就是说，CancellationException这个异常是被视为正常现象的取消。

#### 父子协程的取消

如果父协程取消，对子协程有什么影响呢？同样地，子协程的取消，会对父协程有什么影响呢？

```
/* Jobs can be arranged into parent-child hierarchies where cancellation
* of a parent leads to immediate cancellation of all its [children]. Failure or cancellation of a child
* with an exception other than [CancellationException] immediately cancels its parent. This way, a parent
* can [cancel] its own children (including all their children recursively) without cancelling itself.
*
*/
```

这一段是Job这个接口的文档注释，我截取了一部分出来。我们一起来看下这段文档说明：

```
Job可以被组织在父子层次结构下，当父协程被取消后，会导致它的子协程立即被取消。一个子协程失败或取消的异常（除了CancellationException），它也会立即导致父协程的取消。
```

下面我们就通过代码来测试一下：
先看一下父协程取消对于子协程的影响：

```
    fun testCancel() = runBlocking{
        val parentJob = launch {
            launch {
                Log.e("Test","child Job: before delay")
                delay(2000)
                Log.e("Test","child Job: after delay")
            }

            Log.e("Test","parent Job: before delay")
            delay(1000)
            Log.e("Test","parent Job: after delay")
        }

        delay(500)
        parentJob.cancelAndJoin()
        Log.e("Test","hello")
    }
```

运行结果如下：

```
parent Job: before delay
child Job: before delay
hello
```

可以看到，我们一旦取消父协程对应的Job之后，子协程的执行也被取消了，那么也就验证父协程的取消对于子协程的影响。

子协程正常的CancellationException取消：

```
        val parentJob = launch {
            val childJob = launch {
                Log.e("Test","child Job: before delay")
                delay(2000)
                Log.e("Test","child Job: after delay")

            }

            Log.e("Test","parent Job: before delay")
            delay(1000)
            childJob.cancelAndJoin()
            Log.e("Test","parent Job: after delay")
        }

        delay(500)
        Log.e("Test","hello")
```

运行结果：

```
parent Job: before delay
child Job: before delay
hello
parent Job: after delay
button1 click end
```

可见，如果子协程是正常的取消（即CancellationException），那么对于父协程是没有影响的。

子协程的非CancellationException取消：

```
    fun testCancel() = runBlocking{
        val parentJob = launch {
            val childJob = launch {
                Log.e("Test","child Job: before delay")
                delay(800)
                throw RuntimeException("cause to cancel child job")
            }

            Log.e("Test","parent Job: before delay")
            delay(1000)
            childJob.cancelAndJoin()
            Log.e("Test","parent Job: after delay")
        }

        delay(500)
        Log.e("Test","hello")
    }
```

RuntimeException 会使程序发生crash。

#### 协程的超时取消

可以使用 withTimeout，withTimeoutOrNull。
withTimeoutOrNull与withTimeout的区别在于，当发生超时取消后，withTimeoutOrNull的返回为null，而withTimeout会抛出一个TimeoutCancellationException。

## 协程异常

前面介绍协程取消的时候也介绍了一些关于协程异常的知识，这里再进行一些深入和扩展。
首先，Kotlin 是没有像 Java 中的那样的 Checked Exception 机制的，具体请参考[浅谈Kotlin的Checked Exception机制](https://guolin.blog.csdn.net/article/details/108817286)一文。但是在 Kotlin 中也是可以对一些异常进行捕获的。

### 异常捕获

先看下面的代码：

```
        GlobalScope.launch {
            try {
                throw Exception("Test")
            }catch (e:Exception) {
                Log.e("Test","", e)
            }
        }
```

这种捕获异常的方式毫无疑义是正确的。
那么，如果把 try catch 放到协程的外面呢？

```
        try {
            GlobalScope.launch {
                throw Exception("Test")
            }
        }catch (e:Exception) {
            Log.e("Test","", e)
        }
```

这种方式明显是不行的，因为 Kotlin 是异步的，lunch代码块中的方式并不是在 try catch 代码执行期间执行的，它是异步的，执行期间会超出 try catch 的作用域，这时在抛出异常的话肯定就捕获不到了。和以前我们介绍的 Java Thread 中的捕获机制类似的。

现在再换一种方式：

```
    suspend fun testException() {
        try {
            val job = GlobalScope.async {
                throw Exception("Test")
            }
            job.await()
        }catch (e:Exception) {
            Log.e("Test","", e)
        }

    }
```

这种情况会捕获到的，因为await会消费携程中抛出的异常，从而被catch到。
在协程嵌套的情况下，协程会采取冒泡的行为来传递异常。如果是 supervisorScope，那么子协程的异常不会向上传递。


### 全局异常处理

我们可以为 Thread 设置全局异常处理回调，那么在 Kotlin 中，我们也可以为协程设置全局异常处理。

```
        val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
            Log.e("Test","Throws an exception with message: ${throwable.message}")
        }
        GlobalScope.launch(exceptionHandler) {
            throw Exception("Test")
        }
```

上面的代码其实并不算是一个全局的异常捕获，因为它只能捕获对应协程内未捕获的异常，如果要做到真正的全局捕获，我们可以自己定义一个捕获类的实现，然后修改 META-INF/services/kotlinx.coroutines.CoroutineExceptionHandler 文件来实现。

## 推荐文章

https://kaixue.io/tag/kotlin-coroutines/
[浅谈Kotlin的Checked Exception机制](https://guolin.blog.csdn.net/article/details/108817286)

## 推荐书籍

《深入理解Kotlin协程》