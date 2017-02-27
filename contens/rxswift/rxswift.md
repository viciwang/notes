#RxSwift基本用法

`RxSwift`的一个重要概念是`Observable`，即类`Obserable<Element>`。`Obserable<Element>`是一个事件序列，Element是其事件中携带的数据类型，并且`Obserable<Element>`能异步接收数据。

`Obserable<Element>`有三种事件类型，next，error，completed：

```swift
enum Event<Element>  {
    case next(Element)        // 下一个数据元素
    case error(Swift.Error) 	// 序列以一个错误结束
    case completed				// 序列成功结束
}
```
一个事件序列可以用正则表达式表达为

```
next*(error|completed)?
```
其含义为:

* 事件序列有0或多个next事件
* 如果error或completed事件被接收，序列将不会再产生任何事件。


##创建`Observable`对象
* 通过create函数创建，如

```swift
func aFunction<E>(element: E) -> Observable<E> {
    return Observable.create { observer in
        observer.on(.next(element))	//	发送next事件
        observer.on(.completed)		//	发送completed时间
        return Disposables.create()	//	返回Disposables对象，用于资源释放
    }
}

aFunction(0).subscribe(onNext: { n in
	print(n)
})
```
最终输出

```
0
```

##资源释放

* **通过调用订阅者的`dispose`函数，结束一个事件序列，释放资源**

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
    .subscribe { event in
        print(event)
    }

Thread.sleep(forTimeInterval: 1.0)

subscription.dispose()	// 一秒后，序列结束
```
最终输出
```
0
1
2
3
```

**但手动调用订阅者的dispose并不是一个好的实践（？），更好的方式是通过DisposeBag、takeUntil等实现。**

* **通过DisposeBag释放资源**
此方法的实现原理是当`DisposeBag`对象的`deinit()`函数调用的时候，循环调用加到`DisposeBag`对象中的订阅者的dispose函数。用法如下：

```swift
var disposeBag = DisposeBag()		// 创建DisposeBag对象

var obserable = aFunction()		// 创建obserable

obserable
	.subscribe(onNext: { n in
		print(n)
	})
	.addDisposeableTo(disposeBag)	// 订阅者加到DisposeBag对象
```

* **通过takeUntail函数自动释放资源**

```swift
sequence
    .takeUntil(self.rx.deallocated)	// 到self被释放的时候，自动释放
    .subscribe {
        print($0)
    }
```


##热信号和冷信号
冷热信号的概念源于.NET框架[Reactive Extensions(RX)](https://msdn.microsoft.com/en-us/library/hh242985.aspx)中的Hot Observable和Cold Observable，两者的区别是：
>  Hot Observable是主动的，尽管你并没有订阅事件，但是它会时刻推送，就像鼠标移动；而Cold Observable是被动的，只有当你订阅的时候，它才会发布消息。
> 
>  Hot Observable可以有多个订阅者，是一对多，集合可以与订阅者共享信息；而Cold Observable只能一对一，当有不同的订阅者，消息是重新完整发送。

详细可参考：

* [细说ReactiveCocoa的冷信号与热信号（一）](http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-1.html)
* [细说ReactiveCocoa的冷信号与热信号（二）：为什么要区分冷热信号](http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-2.html)

##监听变量值的变化
RxSwift提供`Variable`类用于监听变量值的变化。

```swift
class AClass {
	var numbers = [Int]()
}

//	如果要监听numbers的变化，可以用`Variable`对numbers进行包装，将number改写成：
class AClass {
	let numbers: Variable<[Int]> = Variable([])
}

var object = AClass()
print("\(object.numbers.value)")	// `Variable`对象有一个value属性，用于操作存储的值

var disposeBag = DisposeBag()

object
	.asObservable()				   // asObservable函数将Variable转化成Observable对象
	.subscribe(onNext: { array in
		print("\(array)")
	})
	.addDisposableTo(disposeBag)
```

##参考
* [RxSwift Getting Started](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/GettingStarted.md)
* [Getting Started With RxSwift and RxCocoa](https://www.raywenderlich.com/138547/getting-started-with-rxswift-and-rxcocoa)
* [ReactiveCocoa vs RxSwift](https://www.raywenderlich.com/126522/reactivecocoa-vs-rxswift)