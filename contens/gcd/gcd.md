# Swift 3 GCD基本概念介绍
维基百科中对于GCD的定义是

> Grand Central Dispatch (GCD) is a technology developed by Apple Inc. to optimize application support for systems with multi-core processors and other symmetric multiprocessing systems.[2] It is an implementation of task parallelism based on the thread pool pattern. The fundamental idea is to move the management of the thread pool out of the hands of the developer, and closer to the operating system. The developer injects "work packages" into the pool oblivious of the pool's architecture. This model improves simplicity, portability and performance.

从中可以提取出以下3点：

1. GCD是Apple开发的一个多核编程的技术
2. GCD是基于线程池实现任务并行的
3. 开发者不用管理线程，而是提交“任务包”到系统中

swift中GCD对应的库是`Dispatch `，其定义是

> Execute code concurrently on multicore hardware by submitting work to dispatch queues managed by the system.

可见，GCD负责管理线程，开发者只需向GCD提供实际任务就行，提供任务的方式主要是代码块（block）的方式。这对开发者带来的思维转变是：将工作考虑为一个队列，而不是一堆线程。这种并行的抽象模型更容易掌握和使用。

## 基本概念

**`串行（Serial）`和`并发（Concurrent）`**

这两个术语用于描述任务相对于其它任务的执行，任务串行执行就是每次只有一个任务被执行，任务并发执行就是在同一时间可以有多个任务被执行。

**`并发（Concurrency）`和`并行(Parallelism)`**

多核设备通过并行来同时执行多个线程；然而，为了使单核设备也能实现这一点，它们必须先运行一个线程，执行一个上下文切换，然后运行另一个线程或进程。即并行要求并发，但并发并不能保证并行。

**`同步（Synchronous）`和`异步（Asynchronous）`**

用于描述一个函数相对于另一个任务完成。一个同步函数只在完成了它预定的任务后才返回，而一个异步函数会立即返回，预定的任务会完成但不会等它完成，即同步函数会阻塞当前线程去执行下一个函数，异步函数则不会阻塞当前线程。

**`串行队列（Serial Queues）`**

每次只执行一个任务，每个任务只在前一个任务完成时才开始。

**`并发队列（Concurrent Queues）`**

同一时间可以有多个任务被执行。并发队列只保证任务会按照被添加的顺序开始执行，而每个任务开始执行的时刻取决于GCD

## 队列类型

GCD提供并管理着一些FIFO的队列，开发者可以向不同的队列提交代码块。队列类型有：

* 主队列（main queue）
* 全局队列（global queue）
* 自定义队列

***`主队列（main queue）`***是系统提供的一个特殊队列，它是一个**串行队列**，并且只有一条线程，即主线程，而主线程是唯一可用于更新 UI 的线程。

***`全局队列（global queue）`***是系统提供的一种**并发队列**，GCD根据优先级（iOS 10根据QoS）将`全局队列`主要分为4大类：background、utility、userInitiated和userInteractive，其优先级依次增加。还有另外两种：default和unspecified作为补充，详见下面的QoS介绍。

***`自定义队列`*** 是指开发者自己创建的队列，可以是并行的，也可以是串行的。

## Quality of Service (QoS)
`Quality of Service`是iOS 8引入的新概念，用于指定任务的重要程度，即执行的优先级，优先级越高的任务会越快执行，并且拥有更多资源。

需要指出QoS不仅用于GCD，而且NSOperation, NSOperationQueue, NSThread objects, dispatch queues, and pthreads (POSIX threads)中都用到QoS去区分不同的任务类型。

> A quality of service (QoS) class allows you to categorize work to be performed by NSOperation, NSOperationQueue, NSThread objects, dispatch queues, and pthreads (POSIX threads).

下面是关于QoS相应的任务类型和执行时间的简单概括:

| QoS | 任务类型 | 执行工作的时间 |
| --- |---|---|
| User-interactive | 一些与用户交互的工作，如刷新用户界面、执行动画 | 瞬间完成 |
| User-initiated | 一些由用户发起的并要求立即有结果的工作，如用户点击按钮之后执行的保存文档工作 | 几秒时间或更少 |
| Utility | 需要一点时间去完成并且不要求立刻出结果的工作，例如下载文件 | 几秒到几分钟时间 |
| Background  | 一些在后台运行，用户不可见的任务。如同步文件，备份等| 需要大量时间，如几分钟甚至几小时 |

另外，QoS的`Default `和`Unspecified `是特殊的类型：

* `Default `介于`User-initiated`和`Utility`之间，默认的优先级类型
* `Unspecified`用于缺乏QoS信息的情况下指定。线程会有`Unspecified`的QoS,如果它们使用历史遗留的没有QoS概念的API。

关于QoS的更详细介绍：[Prioritize Work with Quality of Service Classes](https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html)

iOS 8 用`DispatchQueue.GlobalQueuePriority`（iOS 10已将其标记为废弃）区分不同类型的全局队列

```swift
public enum GlobalQueuePriority {
    case high
    case `default`
    case low
    case background
}
```

iOS 10 用`DispatchQoS.QoSClass`区分不同类型的全局队列

```swift
public enum QoSClass {
    case background
    case utility
    case `default`
    case userInitiated
    case userInteractive
    case unspecified
}
```


*以下API均基于iOS 10.0*

## 获取及创建GCD队列
获取主队列：

```swift
var queue = DispatchQueue.main
```
获取全局队列：

```swift
var queue0 = DispatchQueue.global(qos: .userInitiated)  // QoS 为.userInitiated的全局队列
var queue1 = DispatchQueue.global()                     // QoS 为.default
var queue2 = DispatchQueue.global(priority: .high)      // 指定GlobalQueuePriority获取，已被废弃
```
创建自定义队列：

```swift
var queue0 = DispatchQueue(label: "com.example.queue")      // 创建一个串行队列
var queue1 = DispatchQueue(label: "com.example.queue", attributes: .concurrent) // 创建一个并行队列
```
DispatchQueue的构造函数为

```swift
public convenience init(label: String, qos: DispatchQoS = default, attributes: DispatchQueue.Attributes = default, autoreleaseFrequency: DispatchQueue.AutoreleaseFrequency = default, target: DispatchQueue? = default)
```
可以指定更多参数，定制队列

## 执行任务
获取到一个队列之后，可以通过其`async`和`sync`函数来异步和同步任务，最简单的方式如下

```swift
// 异步执行
DispatchQueue.global().async {
    // 具体的任务代码
}
// 对应的async函数声明如下：
public func async(group: DispatchGroup? = default, qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, execute work: @escaping @convention(block) () -> Swift.Void)

// 同步执行
DispatchQueue.global().sync {
    // 具体的任务代码
}
// 对应的sync函数声明如下：
public func sync(execute block: () -> Swift.Void)
```
可见async拥有更多的配置参数

除了通过block提交任务的方式外，还可以提交一个`DispatchWorkItem`实例来同步或异步执行任务，对应的API如下：

```swift
public func sync(execute workItem: DispatchWorkItem)
public func async(execute workItem: DispatchWorkItem)
public func async(group: DispatchGroup, execute workItem: DispatchWorkItem)
```
`DispatchWorkItem`是对提交给DispatchQueue的任务的封装，从DispatchWorkItem的init函数：

```swift
public init(qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, block: @escaping @convention(block) () -> ())`
```
看出，`DispatchWorkItem`可以指定`QoS`和`DispatchWorkItemFlags`,一个block用于执行业务代码。使用`DispatchWorkItem`可以**取消**已经提交的任务（通过调用其`DispatchWorkItem`实例的cancle行数），但前提是DispatchWorkItem还没有开始执行。

## 延迟执行
swift 3 的GCD延迟执行有以下几个API:

```swift
public func asyncAfter(deadline: DispatchTime, qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, execute work: @escaping @convention(block) () -> Swift.Void)
public func asyncAfter(wallDeadline: DispatchWallTime, qos: DispatchQoS = default, flags: DispatchWorkItemFlags = default, execute work: @escaping @convention(block) () -> Swift.Void)
public func asyncAfter(deadline: DispatchTime, execute: DispatchWorkItem)
public func asyncAfter(wallDeadline: DispatchWallTime, execute: DispatchWorkItem)
```
一般用DispatchTime.now()或DispatchWallTime.now()加上延迟的时间得到一个时间点，如DispatchTime.now() + 2即延迟两秒。用法：

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 2) {  // 延迟两秒执行
    // work
}

DispatchQueue.main.asyncAfter(wallDeadline: .now() + 2) {	// 延迟两秒执行
    // work
}
```
这些API中主要的不同点是一种使用`DispatchTime `，另一种使用`DispatchWallTime `来描述时间

官方文档对`DispatchTime`描述是
> DispatchTime represents a point in time relative to the default clock with nanosecond precision. On Apple platforms, the default clock is based on the Mach absolute time unit.

对`DispatchWallTime`的描述是
> DispatchTime represents an absolute point in time according to the wall clock with microsecond precision. On Apple platforms, the default clock is based on the result of gettimeofday(2).

结合这两个回答

[https://forums.developer.apple.com/thread/49361](https://forums.developer.apple.com/thread/49361)

[http://stackoverflow.com/questions/26062702/what-is-the-difference-between-dispatch-time-and-dispatch-walltime-and-in-what-s](http://stackoverflow.com/questions/26062702/what-is-the-difference-between-dispatch-time-and-dispatch-walltime-and-in-what-s)

可以看出两者的不同在于：

`DispatchTime`是系统的启动时间（以纳秒为单位），当系统睡眠的时候时间停止，即`DispatchTime `不包含系统睡眠时间。`DispatchWallTime`是绝对时间（以毫秒为单位），即挂钟的时间（也就是你在系统栏里看到的当前时间）。

举个例子：如果在16:00时刻指定一个任务要在1个小时后执行，但5分钟后系统睡眠50分钟。如果用`DispatchWallTime`则任务会在17:00执行，而如果用`DispatchTime `则任务会在系统唤醒之后再运行55分钟，即在17:50执行。

## 参考
* [NSOperation](http://nshipster.com/nsoperation/)
* [Quality of Service](https://medium.com/the-traveled-ios-developers-guide/quality-of-service-849cd6dee1e#.752ocoszg)
* [GCD 深入理解：第一部分](https://github.com/nixzhu/dev-blog/blob/master/2014-04-19-grand-central-dispatch-in-depth-part-1.md)
* [https://developer.apple.com/reference/dispatch](https://developer.apple.com/reference/dispatch)