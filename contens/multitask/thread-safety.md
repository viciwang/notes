# iOS线程安全
维基百科对线程安全的描述是：

> Thread safety is a computer programming concept applicable to multi-threaded code. Thread-safe code only manipulates shared data structures in a manner that guarantees safe execution by multiple threads.

这里大概意思是线程安全的代码能保证**安全**操作多线程之间的共享数据。这里的安全可以理解为“正确”，线程安全的代码能够正确地处理多个线程之间的共享变量，使程序功能正确完成。

如果代码是线程不安全的，则代码没有正确处理多个线程之间的共享变量，数据在执行代码之后并不等于预期的值。例如：

```swift
int count = 0;

// 线程1
NSInteger index = 100000;
while (index > 0) {
    count++;
    index--;
}

// 线程2
NSInteger index = 100000;
while (index > 0) {
    count++;
    index--;
}
```
线程1和线程2执行完之后，`count`的值如果不等于200000，即线程不安全。

发生线程不安全的根本原因是**无法保证原子性**，即一个线程中的代码执行到一半的时候会有另一个线程介入，存取这两个线程的共享用数据。

原子性的粒度可大可小，从一个元变量类型（如int、bool类型等）数据的访问到一段代码的执行。

## iOS中线程不安全的例子

考虑以下代码

```objective-c
@interface ViewController ()

@property (atomic, assign) NSInteger count;	// 1. 设置atomic属性

@end

@implementation ViewController

- (void)viewDidLoad {
    
    self.count = 0;
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_enter(group);
    dispatch_group_enter(group);
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;		// 2. 循环100000次，对self.count自增1
        while (index > 0) {
            self.count = self.count + 1;
            index--;
        }
        dispatch_group_leave(group);
    });
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;		// 3. 循环100000次，对self.count自增1
        while (index > 0) {
            self.count = self.count + 1;
            index--;
        }
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"%@", @(self.count));		// 4. 两个循环结束后，打印self.count的值
    });
}

@end
```
这段代码中分别对self.count循环加100000，按照预期两个循环结束之后self.count的值应该是200000，但实际self.count的值却小于200000，下面是某一次执行的结果：

<div align="center"> <image src="thread-safety.png" /></div>

分析以上代码，有以下三点需要注意：

* count已经设置atomic属性
* 两个循环是并发执行的
* self.count是两个循环中的共享数据（group在这里只是用于在两个循环完成之后，执行回调进行输出）

可见出现这种结果的原因就在于以下代码

```
self.count = self.count + 1;
```
并不具备原子性。

`self.count = self.count + 1;`至少包含三条指令：

```
(读取self.count)  load
(加1)             add 1
(保存self.count)  store 
```
如果在线程1取完self.count之后，保存self.count之前，线程2又取了self.count的值，这就导致了线程1的加一不起作用，所以最后self.count比预期的要小。

可见，**对property加atomic属性，并不能确保线程安全。**那么atomic的作用是什么呢？

### atomic的作用
引用博客 *[iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)* 的描述：

**第一、生成原子操作的getter和setter**

设置atomic之后，默认生成的getter和setter方法执行是原子的。也就是说，当我们在线程1执行getter方法的时候（创建调用栈，返回地址，出栈），线程B如果想执行setter方法，必须先等getter方法完成才能执行。

对于值类型，在32位系统里，如果通过getter返回64位的double，地址总线宽度为32位，从内存当中读取double的时候无法通过原子操作完成，如果不通过atomic加锁，有可能会在读取的中途在其他线程发生setter操作，从而出现异常值。如果出现这种异常值，就发生了多线程不安全。

对于对象类型，到了ARC时代，系统自动帮我们添加retain和release，最终setter类似如下：

```objective-c
- (void)setUserName:(NSString *)userName {
    if(_uesrName != userName) {
        [userName retain];
        [_userName release];
        _userName = userName;
    }
}
```
可见setter并不能通过一条指令完成，添加atomic可使setter是一个原子操作。

**第二、设置Memory Barrier**

由于编译器会对我们的代码做优化，使得最终的机器指令的执行顺序不一定严格按照代码的顺序执行，添加Memory Barrier可保证代码按照书写的顺序执行。
详见*[iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)* 的相关内容。

可见，**atomic只保证property的getter和setter是原子操作**，在getter和setter之外的原子性需要我们自己保证。

### 其他常见的线程不安全的例子

```objective-c
@interface ViewController ()

@property (atomic, copy) NSString *string;

@end

@implementation ViewController

- (void)viewDidLoad {
    self.string = @"123";
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;
        while (index > 0) {
            if (index%2 == 0) {
                self.string = @"1234567";
            }
            else {
                self.string = @"123";
            }
            index--;
        }
    });
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;
        while (index > 0) {
            if (self.string.length > 5) {           // 这里虽然有判断string的长度，
                [self.string substringToIndex:4];   // 但由于整个if语句不具备原子性，
            }	                                    // 很容易发生崩溃：'NSRangeException'
            index--;
        }
    });
}

@end
```

类似的还有集合类的操作，例如NSArray等的长度判断。

## 如何保证线程安全

从上面的分析，保证操作的原子性可以保证线程安全。而保证原子性的主要方法是**加锁**，例如上面的例子可改为：

```objective-c
@interface ViewController ()

@property (atomic, assign) NSInteger count;
@property (nonatomic, copy) NSString *lockId;

@end

@implementation ViewController

- (void)viewDidLoad {
    
    self.count = 0;
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_enter(group);
    dispatch_group_enter(group);
    self.lockId = @"lockId";
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;
        while (index > 0) {
            @synchronized (self.lockId) {
                self.count = self.count + 1;
            }
            index--;
        }
        dispatch_group_leave(group);
    });
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger index = 100000;
        while (index > 0) {
            @synchronized (self.lockId) {
                self.count = self.count + 1;
            }
            index--;
        }
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"%@", @(self.count));
    });
}

@end

```

也可直接锁住整个while循环


```objective-c
@synchronized (self.lockId) {
    NSInteger index = 100000;
    while (index > 0) {
        self.count = self.count + 1;
        index--;
    }
}
```

对于各种锁的介绍，以及其性能比较，可参见：[深入理解 iOS 开发中的锁](https://bestswifter.com/ios-lock/)

### 其他保证线程安全的点：
* 使用不可修改的数据类型

苹果的一份文档（[Thread Safety Summary](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html#//apple_ref/doc/uid/10000057i-CH12-SW1)）列出了SDK中线程安全的类，其中不可修改的类都是线程安全的，而其对应的可修改的类都是线程不安全的。

* 线程自行存储变量

线程保存一份私有的变量拷贝，这样就不会受其他线程的影响。


## 参考
* [Thread-Safe Class Design](https://www.objc.io/issues/2-concurrency/thread-safe-class-design/)
* [iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)
* [正确使用多线程同步锁@synchronized()](http://mrpeak.cn/blog/synchronized/)
* [深入理解 iOS 开发中的锁](https://bestswifter.com/ios-lock/)
* [Thread Safety Summary](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html#//apple_ref/doc/uid/10000057i-CH12-SW1)