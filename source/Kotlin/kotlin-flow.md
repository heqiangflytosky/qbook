---
title: Kotlin -- Flow
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin Flow的使用
date: 2019-12-22 10:00:00
---

## 概述

Flow是google官方提供的一套基于kotlin协程的响应式编程模型，它与RxJava的使用类似，但相比之下Flow使用起来更简单，另外Flow作用在协程内，可以与协程的生命周期绑定，当协程取消时，Flow也会被取消，避免了内存泄漏风险。    

## 使用介绍

先来看一个简单的例子：    

```
    fun testFlow1() {
        runBlocking<Unit> {
            // 启动并发的协程以验证主线程并未阻塞
            launch {
                for (k in 1..3) {
                    Log.e(tag,"I'm not blocked $k")
                    delay(1000)
                }
            }
            simple().flowOn(Dispatchers.IO)
                .onEmpty { Log.e(tag,"onEmpty") }
                .onStart { Log.e(tag,"onStart") }
                .onEach { Log.e(tag,"onEach") }
                .onCompletion { Log.e(tag,"onCompletion") }
                .catch { exception -> exception.message?.let { Log.e(tag,it) } }
                // 收集这个流
                .collect{ value -> Log.e(tag,"value = $value") }
        }
    }

    fun simple(): Flow<Int> = flow { // 流构建器
        for (i in 1..3) {
            delay(100) // 假装我们在这里做了一些有用的事情
            Log.e(tag,"emit $i")

            emit(i) // 发送下一个值
        }
    }
```

 - flow{}为上游数据提供方，并通过emit()发送一个或多个数据，当发送多个数据时，数据流整体是有序的，即先发送先接收；另外发送的数据必须来自同一个协程内，不允许来自多个CoroutineContext，所以默认不能在`flow{}`中创建新协程或通过`withContext()`切换协程。如需切换上游的CoroutineContext，可以通过flowOn()进行切换。    
 - collect{}为下游数据使用方，collect是一个扩展函数，且是一个非阻塞式挂起函数(使用suspend修饰)，所以Flow只能在kotlin协程作用域或者其他挂起函数中使用。    
 - 其他操作符可以认为都是服务于整个数据流的，包括对上游数据处理、异常处理等。    

### collect 挂起问题

这需要注意一点：只要调用了 `collect` 函数之后就相当于进入了一个死循环，在数据提供方提供完数据之前，是不会往下面执行的。如果 `simple()` 是一个死循环，那么 `collect` 函数的下一行代码永远不会执行。因此，如果你的代码中有多个Flow需要collect，下面这种写法就是完全错误的：    

```
lifecycleScope.launch {
    flow1.collect {
        ...
    }
    flow2.collect {
        ...
    }
}
```

这种写法flow2中的数据是无法及时得到更新的。正确的写法应该是借助launch函数再启动子协程去collect，这样不同子协程之间就互不影响了：    

```
lifecycleScope.launch {
    launch {
        flow1.collect {
            ...
        }
    }
    launch {
        flow2.collect {
            ...
        }
    }
}
```

### 流速不均匀问题

由于Flow是一种基于观察者模式的响应式编程模型，水源发出了一个数据，水龙头这边就会收到一个数据。但是水龙头处理数据的速度不一定和水源发出数据的速度是一致的，如果水龙头处理速度过慢，就可能出现管道阻塞的现象。    
响应式编程框架都可能会遇到这种问题，RxJava中还有专门的背压策略来处理这类问题。Flow当中其实也有，但是我们今天不讨论这种过于高端的技巧，今天使用一个特别简单的方案就可以解决这个流速不均匀问题。        
首先我们创建一个这样的场景：数据发送端每隔1秒发送一个数据，接收端每隔3秒处理一个数据。这样就会导致数据阻塞，数据接收端处理的数据有延后，不是发送端发送的最新数据。        

```
    fun simple(): Flow<Int> = flow { 
        for (i in 1..10) {
            delay(1000)
            Log.e(tag,"emit $i")
            emit(i)
        }
    }
```

```
        lifecycleScope.launch {
            simple().flowOn(Dispatchers.IO)
                .collect{ value ->
                    Log.e(tag,"value 111111 = $value")
                    delay(3000)
                }
        }
```

如果我们只是想处理最新的数据，而取消掉已经过期的数据，那么只需要借助 `collectLatest` 函数就能做到。    

```
        lifecycleScope.launch {
            simple().flowOn(Dispatchers.IO)
                .collectLatest{ value ->
                    Log.e(tag,"value 111111 = $value")
                    delay(3000)
                    Log.e(tag,"value 222222 = $value")
                }
        }
```

`collectLatest` 函数只接收处理最新的数据。如果有新数据到来了而前一个数据还没有处理完，则会将前一个数据剩余的处理逻辑全部取消。    

```
E  emit 1
E  value 111111 = 1
E  emit 2
E  value 111111 = 2
E  emit 3
E  value 111111 = 3
.....
E  emit 9
E  value 111111 = 9
E  emit 10
E  value 111111 = 10
E  value 222222 = 10
```

可以看到，只有最后一个数据走完了全部流程。处理前面的数据时，因为还没有处理完就来了新数据，因此，完整的处理逻辑没有走完就被取消了。        


## Flow的生命周期管理

 - lifecycleScope.launchWhenCreated
 - lifecycleScope.launchWhenStarted
 - repeatOnLifecycle(Lifecycle.State.CREATED)
 - repeatOnLifecycle(Lifecycle.State.STARTED)
 - repeatOnLifecycle(Lifecycle.State.RESUMED)
 - flow.flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)

它们的区别在《Kotlin-协程》一文中做过详细介绍。    

```
    fun testFlow2() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                var flow: Flow<Int> = flow {
                    var counter = 0

                    while (true) {
                        delay(1000)
                        Log.e(tag, "value 222222 = $counter")
                        counter++
                    }
                }

                flow.onEach {
                    Log.e(tag, "onEach $it")
                }.onStart {
                    Log.e(tag, "onStart")
                }.onCompletion {
                    Log.e(tag, "onCompletion")
                }.collect {
                    Log.e(tag, "collect $it")
                }
            }
        }
    }
```

## 常用操作符

### 创建操作符

 - flow：创建Flow的操作符。
 - flowof：构造一组数据的Flow进行发送。
 - asFlow：将其他数据转换成Flow，一般是集合向Flow的转换，如listOf(1,2,3).asFlow()。
 - callbackFlow：将基于回调的 API 转换为Flow数据流

#### callbackFlow

```
    {
        GlobalScope.launch {
            docallbackFlow().flowOn(Dispatchers.IO).collect{ value -> Log.e(tag,"collect data = $value")}
        }
    }

    fun docallbackFlow() : Flow<Boolean> = callbackFlow {
        var callback = object : Callback{
            override fun onDataChange(data: Boolean?) {
                Log.e(tag,"data =  $data")
                trySend(data!!)
            }

        }
        controller.setCallback(callback)
        trySend(false)
        Log.e(tag,"awaitClose")
        awaitClose { Log.e(tag,"close") }
    }
```

### 回调操作符

 - onStart：上游flow{}开始发送数据之前执行
 - onCompletion：flow数据流取消或者结束时执行
 - onEach：上游向下游发送数据之前调用，每一个上游数据发送后都会经过onEach()
 - onEmpty：当流完成却没有发出任何元素时执行。如emptyFlow().onEmpty {}
 - onSubscription：SharedFlow 专用操作符，建立订阅之后回调。和onStart的区别：因为SharedFlow是热流，因此如果在onStart发送数据，下游可能接收不到，因为提前执行了。

### 变换操作符

 - map：对上游发送的数据进行变换，collect最后接收的是变换之后的值
 - mapLatest：类似于 collectLatest，当emit发送新值，会取消掉map上一次转换还未完成的值。
 - mapNotNull：仅发送map之后不为空的值。
 - transform：对发出的值进行变换 。不同于map的是，经过transform之后可以重新发送数据，甚至发送多个数据，因为transform内部又重新构建了flow。
 - transformLatest：类似于mapLatest，当有新值发送时，会取消掉之前还未转换完成的值。
 - transformWhile：返回值是一个Boolean，当为true时会继续往下执行；反之为false，本次发送的流程会中断。
 - asSharedFlow：MutableStateFlow 转换为 StateFlow ，即从可变状态变成不可变状态。
 - asStateFlow：MutableSharedFlow 转换为 SharedFlow ，即从可变状态变成不可变状态。
 - receiveAsFlow：Channel 转换为Flow ，上游与下游是一对一的关系。如果有多个下游观察者，可能会轮流收到值。
 - consumeAsFlow：Channel 转换为Flow ，有多个下游观察者时会crash。
 - withIndex：将数据包装成IndexedValue类型，内部包含了当前数据的Index。
 - scan(initial: R, operation: suspend (accumulator: R, value: T) -> R)：把initial初始值和每一步的操作结果发送出去。
 - produceIn：转换为Channel的 ReceiveChannel
 - runningFold(initial, operation: (accumulator: R, value: T) -> R)：initial值与前面的流共同计算后返回一个新流，将每步的结果发送出去。
 - runningReduce*：返回一个新流，将每步的结果发送出去，默认没有initial值。
 - shareIn：flow 转化为 SharedFlow，后面会详细介绍。
 - stateIn：flow转化为StateFlow，后面会详细介绍。

### 过滤操作符

 - filter：筛选符合条件的值，返回true继续往下执行。
 - filterNot：与filter相反，筛选不符合条件的值，返回false继续往下执行。
 - filterNotNull：筛选不为空的值。
 - filterInstance：筛选对应类型的值，如.filterIsInstance()用来过滤String类型的值
 - drop：drop(count: Int)参数为Int类型，意为丢弃掉前count个值。
 - dropWhile：找到第一个不满足条件的值，返回其和其后所有的值。
 - take：与drop()相反，意为取前n个值。
 - takeWhile：与dropWhile()相反，找到第一个不满足条件的值，返回其前面所有的值。
 - debounce：debounce(timeoutMillis: Long)指定时间内只接收最新的值，其他的过滤掉。可以用来确保处理flow的各项数据之间存在一定的时间间隔。    
想象一下我们正在Edge浏览器的地址栏中输入搜索关键字，浏览器的地址栏下方通常都会给出一些搜索建议。这些搜索建议是根据用户当前输入的内容通过发送网络请求去服务器端实时获取的。    
但如果用户每输入一个字符就立刻发起一条网络请求，这是非常不合理的设计。    
原因很简单，网络请求是不可能做到无延时响应的，而用户的打字速度通常都比较快。如果用户每输入一个字符都立刻发起一条网络请求，那么很有可能用户输完了第3个字符之后，对应第1个字符的网络请求结果才刚刚返回回来，而此时的数据早已是无效数据了。     
正确的做法是，我们不应该在用户每输入一个字符时就立刻发起一条网络请求，而是应该在一定时间的停顿之后再发起网络请求。如果用户停顿了一段时间，很有可能说明用户已经结束了快速输入，开始期待一些搜索建议了。这样就能有效地避免发起无效网络请求的情况。     
而要实现这种功能，使用debounce操作符函数就会非常简单，它就是为了这种场景而设计的。     
 - sample：sample(periodMillis: Long)在指定周期内，获取最新发出的值。和debounce稍微有点类似。它可以从flow的数据流当中按照一定的时间间隔来采样某一条数据。如：
```
  flow {
          repeat(10) {
          emit(it)
          delay(110)
      }
  }.sample(200)
```

执行结果：1, 3, 5, 7, 9
 - distinctUntilChangedBy：判断两个连续值是否重复，可以设置是否丢弃重复值。
 - distinctUntilChanged：若连续两个值相同，则跳过后面的值。

### 重试操作符

 - retry
 - retryWhen

这两个运算符在大多数情况下都可以互换使用。    

```
fun <T> Flow<T>.retry(
    retries: Long = Long.MAX_VALUE,
    predicate: suspend (cause: Throwable) -> Boolean = { true }
): Flow<T>
```

retry函数具有默认参数。使用 `.retry()` 它将继续重试，直到任务成功完成。    
`.retry(3)` 它只会重试3。    
它的返回值标识是否需要重试，如果返回了false，就不再继续重试。   

我们还可以使用 `retryWhen`  

```
.retryWhen { cause, attempt ->

}
```

cause 标识所抛出的异常，attempt 表示重试的次数。返回值表示是否需要继续重试。    

```
    fun testFlow1() {
        runBlocking<Unit> {
            // 启动并发的协程以验证主线程并未阻塞
            launch {
                for (k in 1..3) {
                    Log.e(tag,"I'm not blocked $k")
                    delay(1000)
                }
            }
            simple().flowOn(Dispatchers.IO)
                .onEmpty { Log.e(tag,"onEmpty") }
                .onStart { Log.e(tag,"onStart") }
                .onEach {
                    Log.e(tag,"onEach $it")
                    if(it == 2) {
                        throw Exception("error")
                    }
                }
                .onCompletion { Log.e(tag,"onCompletion") }

                .retry(2) {
                    Log.e(tag,"retry... $it")
                    true
                }
//                .retryWhen { cause, attempt ->
//                    Log.e(tag,"retryWhen... $cause $attempt")
//                    attempt < 2
//                }
                .catch { Log.e(tag,"",it)  }
                // 收集这个流
                .collect{ value -> Log.e(tag,"value = $value") }
        }
    }

    fun simple(): Flow<Int> = flow { // 流构建器
        for (i in 1..3) {
            delay(100) // 假装我们在这里做了一些有用的事情
            Log.e(tag,"emit $i")

            emit(i) // 发送下一个值
        }
    }
```

上述代码中使用了`retry`和`retryWhen`，他们必须放在 `catch` 操作符的前面，不满足重试条件时异常会被 `catch` 捕获。   

### 终端操作符

上面的操作符最终都还是要通过collect函数来收集结果的。接下来我们学习两个不需要借助collect函数，自己就能终结整个flow流程的操作符函数，这种操作符函数也被称为终端操作符函数。

 - collect 和 collectLatest
 - reduce：reduce函数会通过参数给我们一个Flow的累积值和一个Flow的当前值，我们可以在函数体中对它们进行一定的运算，运算的结果会作为下一个累积值继续传递到reduce函数当中。
 - fold：和reduce类似，主要区别在于fold函数需要传入一个初始值，这个初始值会作为首个累积值被传递到fold的函数体当中


### 结合操作符

前面介绍的操作符都是在一个 flow 上进行的操作，接下来介绍几个对两个flow进行操作的操作符。    
 - flatMapConcat：将几个flow中的数据按顺序合并
 - flatMapMerge：将几个flow中的数据按合并，并不一定按顺序
 - flatMapLatest：flow1中的数据传递到flow2中会立刻进行处理，但如果flow1中的下一个数据要发送了，而flow2中上一个数据还没处理完，则会直接将剩余逻辑取消掉，开始处理最新的数据。    
 - zip：使用zip连接的两个flow，它们之间是并行的运行关系。这点和flatMap差别很大，因为flatMap的运行方式是一个flow中的数据流向另外一个flow，是串行的关系。    

flatMap的核心，就是将两个flow中的数据进行映射、合并、压平成一个flow，最后再进行输出。        

#### flatMapConcat

下面看一个例子：    

```
    fun testFlatMap() {
        lifecycleScope.launch {
            flowOf(1, 2, 3)
                .flatMapConcat {
                    flowOf("a$it", "b$it")
                }
                .collect {
                    Log.d("Test",it)
                }
        }
    }
```

这里的第一个flow会依次发送1、2、3这几个数据。然后在flatMapConcat函数中，我们传入了第二个flow。    
第二个flow会依次发送a、b这两个数据，但是在a、b的后面又各自拼接了一个it。    
这个it就是来自第一个flow中的数据。所以，flow1中的1、2、3会依次与flow2中的a、b进行组合，这样就能组合出a1、b1、a2、b2、a3、b3这样几条数据。    
而collect函数最终收集到的就是这些组合后的数据。    
最终输出：

```
a1
b1
a2
b2
a3
b3
```

现实中我们通常有这样的场景：请求一个网络资源时需要依赖于先去请求另外一个网络资源。    
比如说我们想要获取用户的数据，但是获取用户数据必须要有token授权信息才行，因此我们得先发起一个请求去获取token信息，然后再发起另一个请求去获取用户数据。    
如果我们通过回调的方式来实现，通常会嵌套比较深，容易陷入回调地狱。     
而这个问题我们就可以借助flatMapConcat函数来解决。    
将sendGetTokenRequest()函数和sendGetUserInfoRequest()函数都使用flow的写法进行实现：    

```
fun sendGetTokenRequest(): Flow<String> = flow {
    // send request to get token
    emit(token)
}

fun sendGetUserInfoRequest(token: String): Flow<String> = flow {
    // send request with token to get user info
    emit(userInfo)
}

fun main() {
    runBlocking {
        sendGetTokenRequest()
            .flatMapConcat { token ->
                sendGetUserInfoRequest(token)
            }
            .flowOn(Dispatchers.IO)
            .collect { userInfo ->
                println(userInfo)
            }
    }
}
```

当然，这个用法并不仅限于只能将两个flow串连成一条链式任务，如果你有更多的任务需要串到这同一条链上，只需要不断连缀flatMapConcat即可：

```
fun main() {
    runBlocking {
        flow1.flatMapConcat { flow2 }
             .flatMapConcat { flow3 }
             .flatMapConcat { flow4 }
             .collect { userInfo ->
                 println(userInfo)
             }
    }
}
```

#### flatMapMerge

flatMapMerge 和 flatMapConcat 类似，但是区别其实就在字面上。concat是连接的意思，merge是合并的意思。连接一定会保证数据是按照原有的顺序连接起来的，而合并则只保证将数据合并到一起，并不会保证顺序。    
flatMapMerge函数的内部是启用并发来进行数据处理的，它不会保证最终结果的顺序。    

先用 flatMapConcat 来实现这样一个例子：    

```
    fun testFlatMap() {
        lifecycleScope.launch {
            flowOf(300, 200, 100)
                .flatMapConcat {
                    flow {
                        delay(it.toLong())
                        emit("a$it")
                        emit("b$it")
                    }
                }
                .collect {
                    println(it)
                }
        }
    }
```

输出结果：

```
a300
b300
a200
b200
a100
b100
```

最终的结果仍然是按照flow1中数据发送的顺序输出的，即使第一个数据被delay了300毫秒，后面的数据也没有优先执行权。这就是flatMapConcat函数所代表的涵义。    

接下来用 flatMapMerge 来实现：    

```
    fun testFlatMap() {
        lifecycleScope.launch {
            flowOf(300, 200, 100)
                .flatMapMerge {
                    flow {
                        delay(it.toLong())
                        emit("a$it")
                        emit("b$it")
                    }
                }
                .collect {
                    println(it)
                }
        }
    }
```

输出结果：

```
a100
b100
a200
b200
a300
b300
```

它是可以并发着去处理数据的，而并不保证顺序。那么哪条数据被delay的时间更短，它就可以更优先地得到处理。    


#### flatMapLatest

flatMapLatest 和前面介绍的 collectLatest 操作符意思比价接近，flow1中的数据传递到flow2中会立刻进行处理，但如果flow1中的下一个数据要发送了，而flow2中上一个数据还没处理完，则会直接将剩余逻辑取消掉，开始处理最新的数据。    

```
    fun testFlatMap() {
        lifecycleScope.launch {
            flow {
                for (i in 1..5) {
                    delay(100)
                    Log.e(tag,"emit $i")
                    emit(i)
                }
            }.flatMapLatest {
                flow {
                    Log.e(tag,"before do $it")
                    delay(200)
                    emit("$it")
                    Log.e(tag,"after do $it")
                }
            }.collect {
                Log.e(tag,"collect $it")
            }
        }
    }
```

输出结果：

```
emit 1
before do 1
emit 2
before do 2
emit 3
before do 3
emit 4
before do 4
emit 5
before do 5
collect 5
after do 5
```

只有最后一个数走完了处理逻辑，其他都被取消了。    

#### zip

和flatMap函数有点类似，zip函数也是作用在两个flow上的。不过，它们的适用场景完全不同。    
使用zip连接的两个flow，它们之间是并行的运行关系。这点和flatMap差别很大，因为flatMap的运行方式是一个flow中的数据流向另外一个flow，是串行的关系。    

```
    fun testZip() {
        lifecycleScope.launch {
            val flow1 = flow {
                for (i in 1..5) {
                    delay(100)
                    Log.e(tag,"emit $i")
                    emit(i.toString())
                }
            }
            val flow2 = flow {
                for (i in 10..20) {
                    delay(200)
                    Log.e(tag,"emit $i")
                    emit(i.toString())
                }
            }
            flow1.zip(flow2) { a, b ->
                a + b
            }.collect {
                println(it)
            }
        }
    }
```

两个flow发送的数据个数不同，而且间隔也不一样。    

输出：

```
emit 1
emit 10
110
emit 2
emit 11
211
emit 3
emit 12
312
emit 4
emit 13
413
emit 5
emit 14
514
```

zip函数的规则是，只要其中一个 flow 中的数据全部处理结束就会终止运行，剩余未处理的数据将不会得到处理。因此，flow2 中的15到20这些数据会被舍弃掉。    
而且，每次emit的数据都是并行关系，每次执行的时间取决于耗时更久的那个。    
它的使用场景：比如我们有两个接口来请求不用的数据，需要这两个接口都返回数据后一并展示出来。那么 zip 就和贴合这样的场景，避免了串行请求耗时叠加的问题。    
zip 请求并不局限于两个flow这样的场景，多个flow也可以用zip实现并行处理。    

```
            flow1.zip(flow2) { a, b ->
                a + b
            }.zip(flow3) { a, b ->
                a + b
            }.collect {
                println(it)
            }
```

### 背压处理操作符

 - collectLatest
 - buffer
 - conflate

这三个操作符可以用来处理 Flow 流速不均匀的问题，其中 collectLatest 前面已经介绍过。它的处理逻辑时下一个数据发送时如果前面的数据逻辑没有处理完就直接取消掉。    
#### buffer

我们来看看 buffer 是如何处理的：    

```
    fun testBuffer() {
        lifecycleScope.launch {
            flow {
                for (i in 1..5) {
                    emit(i.toString())
                    Log.e(tag,"emit $i")
                    delay(100)
                }
            }.buffer().collect{ value ->
                    Log.e(tag,"value 111111 = $value")
                    delay(300)
                    Log.e(tag,"value 222222 = $value")
                }
        }
    }
```

```
emit 1
value 111111 = 1
emit 2
emit 3
value 222222 = 1
value 111111 = 2
emit 4
emit 5
value 222222 = 2
value 111111 = 3
value 222222 = 3
value 111111 = 4
value 222222 = 4
value 111111 = 5
value 222222 = 5
```

buffer函数会让flow函数和collect函数运行在不同的协程当中，这样flow中的数据发送就不会受collect函数的影响了。    
buffer函数其实就是一种背压的处理策略，它提供了一份缓存区，当Flow数据流速不均匀的时候，使用这份缓存区来保证程序运行效率。    
flow函数只管发送自己的数据，它不需要关心数据有没有被处理，反正都缓存在buffer当中。    
而collect函数只需要一直从buffer中获取数据来进行处理就可以了。    
但是，如果流速不均匀问题持续放大，缓存区的内容越来越多时又该怎么办呢？    
这个时候，我们又需要引入一种新的策略了，来适当地丢弃一些数据。    
那么进入到本篇文章的最后一个操作符函数：conflate。    

#### conflate

buffer 处理背压的逻辑是不会丢弃数据，`collectLatest` 的处理方式时新数据来时原来的逻辑没有处理完直接取消掉，就会导致处理流程是不完整的。    
而 conflate 的处理逻辑是可以保证新数据来时让当前正在处理的逻辑处理完，正在处理当前逻辑时如果有新数据发送过来，那么没有处理的数据就当做过期数据丢弃掉。    

```
    fun tesConflate() {
        lifecycleScope.launch {
            flow {
                for (i in 1..3) {
                    emit(i.toString())
                    Log.e(tag,"emit $i")
                    delay(100)
                }
            }.conflate().collect{ value ->
                Log.e(tag,"value 111111 = $value")
                delay(300)
                Log.e(tag,"value 222222 = $value")
            }
        }
    }
```

```
emit 1
value 111111 = 1
emit 2
emit 3
value 222222 = 1
value 111111 = 3
value 222222 = 3
```

可以看到数据2被丢弃，数据1和3完整的处理。   

## 冷流和热流

在Kotlin中，Flow 是一种冷流，不过有一种特殊的Flow（ StateFlow/SharedFlow） 是热流。什么是冷流，他和热流又有什么关系呢？    
冷流：只有订阅者订阅时，才开始执行发射数据流的代码。并且冷流和订阅者只能是一对一的关系，当有多个不同的订阅者时，消息是重新完整发送的。也就是说对冷流而言，有多个订阅者的时候，他们各自的事件是独立的。       
热流（ StateFlow/SharedFlow）：无论有没有订阅者订阅，事件始终都会发生。当 热流有多个订阅者时，热流与订阅者们的关系是一对多的关系，可以与多个订阅者共享信息。      

下面通过例子来看一下他们的区别。    

```
    var flow1 = flow {
        for (i in 1..50) {
            emit(i.toString())
            Log.e(tag,"emit $i")
            delay(1000)
        }
    }

    fun testFlow() {
        lifecycleScope.launch {
            flow1.collect{ value ->
                    Log.e(tag,"value 111111 = $value")
                }
        }
    }

    fun test2(view: View) {
        lifecycleScope.launch {
            flow1.collect{ value ->
                Log.e(tag,"value 3333 = $value")
            }
        }
    }
```

构造一个Flow，然后首先执行 testFlow()，再执行 test2，通过打印发现，test2 是重新发送数据。    
接下来我们通过 stateIn 把 Flow 转换成 StateFlow。    

```
    var flow1 = flow {
        for (i in 1..50) {
            emit(i.toString())
            Log.e(tag,"emit $i")
            delay(1000)
        }
    }.stateIn(lifecycleScope, SharingStarted.WhileSubscribed(5000), "")
```

按照前面的步骤执行发现，testFlow 和 test2 订阅了通过一个流，他们收到了相同的数据。   

### StateFlow

stateIn 可以把 Flow 转换成 StateFlow，它接收3个参数，其中第1个参数是作用域，传入viewModelScope即可。第3个参数是初始值。第二个参数传入的是超时的事件，也就是说，到达指定的生命周期状态时，Flow并不会立即停止工作，而是会等待设定的超时事件。在此期间flow会继续工作来发送数据，但是这个事件订阅者是不会收到数据的。如果在超时时间内，重新返回，那么订阅者就会继续收到数据。如果到达了超时的时间，那么Flow就停止工作。重新返回时会重新执行flow。    

```
    var flow1 = flow {
        for (i in 1..50) {
            emit(i.toString())
            Log.e(tag,"emit $i")
            delay(1000)
        }
    }.stateIn(lifecycleScope, SharingStarted.WhileSubscribed(5000), "")
    
    fun testFlow3() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                flow1.onStart {
                    Log.e(tag, "onStart")
                }.onCompletion {
                    Log.e(tag, "onCompletion")
                }.collect{ value ->
                        Log.e(tag,"value 111111 = $value")
                    }
            }
        }
    }
```

再来介绍一下 MutableStateFlow 的用法：    

```
    var stateFlow = MutableStateFlow(0)
    
    fun test2(view: View) {
        stateFlow.value +=1
    }
    
    fun testFlow4() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                stateFlow.collect { value ->
                    Log.e(tag, "value 111111 = $value")
                }
            }
        }
    }
```

当调用 testFlow4() 订阅flow时，只会发送 stateFlow 当前的 value 值，当 stateFlow.value 变化时，订阅者会实时收到数据。    
StateFlow 是粘性的，以上面代码为例，当我们把Activity放到后台，然后再返回到前台时，订阅者会收到之前的值。如果我们不想使用粘性的特性，那么就要使用 SharedFlow 了。    

### SharedFlow

`shareIn` 可以把 Flow 转换成 SharedFlow，它接受三个参数：    
 - scope：作用域
 - started：启动策略，SharingStarted.Eagerly:立即启动；SharingStarted.Lazily：在第一个订阅者出现后开始共享数据，并使数据流永远保持活跃状态；SharingStarted.WhileSubscribed()：存在订阅者时，将使上游提供方保持活跃状态。    
 - replay：订阅时从流中回放的元素数量
同样还可以通过 MutableSharedFlow 来创建。    

```
    var counter = 0
    var sharedFlow = MutableSharedFlow<Int>()
    fun test2(view: View) {
        lifecycleScope.launch {
            sharedFlow.emit(counter++)
        }
    }
    
    fun testFlow4() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                sharedFlow.collect { value ->
                    Log.e(tag, "value 111111 = $value")
                }
            }
        }
    }
```

上面的例子演示了 SharedFlow 的非粘性特性。    

```
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
)
```

 - replay： 表示在订阅时从流中回放的元素数量。默认值为 0，表示不回放任何元素。如果设置为正整数 n，则在订阅时将向新订阅者回放最近的 n 个元素，此时就有粘性特性了。    
 - extraBufferCapacity： 表示额外的缓冲容量，用于存储订阅者尚未消耗的元素。默认值为 0，表示不使用额外的缓冲容量。设置为正整数 m 时，会在内部使用一个带有额外 m 容量的缓冲区。    
 - onBufferOverflow： 表示在缓冲区溢出时的处理策略。默认值为 BufferOverflow.SUSPEND，表示当缓冲区溢出时暂停发射，等待订阅者消费。其他选项还包括 BufferOverflow.DROP_OLDEST 和 BufferOverflow.DROP_LATEST，它们分别表示在缓冲区溢出时丢弃最老的元素或最新的元素。

## 文章

[Kotlin Flow响应式编程，基础知识入门 ](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650269681&idx=1&sn=dbc3ea08e3eecb324dfa46b2e6bd46d3&chksm=8863169ebf149f8869cdb5b07ddbad691c8ec09e741892b7f2010cd8a931e8f17a3dbcba8cd6&scene=21#wechat_redirect)
[Kotlin Flow响应式编程，操作符函数进阶](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650270522&idx=1&sn=b05ff0909b3454cfb5e17bbfd4a37f60&chksm=88631a55bf149343b7bcfefa54b563df66a3b5fb7848c296dc50b964a3e06770241c6aa42a15&scene=21#wechat_redirect)
[Kotlin Flow响应式编程，StateFlow和SharedFlow ](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650271833&idx=1&sn=aa56e9319033163672300b6c46edbc7a&chksm=88630136bf148820b1df8f9fea45f3fab196394425fb580c1a1acccb629d305b497cc9340bf4&scene=27&poc_token=HGkPqmajbN_KEMMZmmIGt1b9yIRCiwijoKcEaNqW)
[Kotlin | 协程Flow数据流](https://zhuanlan.zhihu.com/p/613172764?utm_id=0)
