# iOS底层原理总结 -多线程详解
目录：<br>
一. 多线程基础<br>
1.进程<br>
2.线程<br>
3.进程和线程的比较<br>
4.线程的串行<br>
5.多线程<br>
6.多线程原理<br>
7.多线程优缺点<br>
8.多线程的应用<br>
二. 多线程实现方案<br>
1.pthreads<br>
2.NSThread<br>
3.GCD<br>
4.NSOperation<br>
三.三种多线程技术比较<br>
四.多线程的锁<br>
1.什么是锁<br>
2.锁的分类<br>
3.性能对比<br>
4.常见的死锁<br>

[GCD-Demo](https://github.com/xiaoyaoknight/GCD-Demo)


## 一. 多线程基础
#####1.进程
>进程是指在系统中正在运行的一个应用程序
每个进程之间是独立的，每个进程均运行在其专用且受保护的内存空间内

#####2.线程
>一个进程要想执行任务，必须得有线程（每1个进程至少要有1条线程，称为主线程）
一个进程（程序）的所有任务都在线程中执行

##### 3.进程和线程的比较
>1.线程是CPU调用(执行任务)的最小单位。
2.进程是CPU分配资源的最小单位。
3.一个进程中至少要有一个线程。
4.同一个进程内的线程共享进程的资源。

#####4.线程的串行
>一个线程中任务的执行是串行的
如果要在一个线程中执行多个任务，那么只能一个一个地按顺序执行这些任务
也就是说，在同一时间内，一个线程只能执行一个任务

#####5.多线程
>一个进程中可以开启多条线程，每条线程可以并行（同时）执行不同的任务
多线程技术可以提高程序的执行效率

#####6.多线程原理
>同一时间，CPU只能处理1条线程，只有1条线程在工作（执行），多线程并发（同时）执行，其实是CPU快速地在多条线程之间调度（切换），如果CPU调度线程的时间足够快，就造成了多线程并发执行的假象。
那么如果线程非常非常多，会发生什么情况？
CPU会在N多线程之间调度，CPU会累死，消耗大量的CPU资源，同时每条线程被调度执行的频次也会会降低（线程的执行效率降低）。
因此我们一般只开3-5条线程。

#####7.多线程优缺点
>多线程的优点:
能适当提高程序的执行效率
能适当提高资源利用率（CPU、内存利用率）

>多线程的缺点:
创建线程是有开销的，iOS下主要成本包括：内核数据结构（大约1KB）、栈空间（子线程512KB、主线程1MB，也可以使用-setStackSize:设置，但必须是4K的倍数，而且最小是16K），创建线程大约需要90毫秒的创建时间
如果开启大量的线程，会降低程序的性能，线程越多，CPU在调度线程上的开销就越大。
程序设计更加复杂：比如线程之间的通信、多线程的数据共享等问题。

#####8.多线程的应用
> 主线程的主要作用
显示\刷新UI界面
处理UI事件（比如点击事件、滚动事件、拖拽事件等）
主线程的使用注意
别将比较耗时的操作放到主线程中
耗时操作会卡住主线程，严重影响UI的流畅度，给用户一种“卡”的坏体验
将耗时操作放在子线程中执行，提高程序的执行效率

一些其他的概念可以阅读文章[iOS多线程详解：概念篇](https://juejin.im/post/5ab4a3b0f265da237f1e3b37#heading-4)，在此不再赘述。

## 二. 多线程实现方案
![image.png](https://upload-images.jianshu.io/upload_images/1401554-46636983a344b632.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)
### [pthreads](https://www.jianshu.com/p/4434f18c5a95)
pthread: 跨平台，适用于多种操作系统，可移植性强，是一套纯C语言的通用API，且线程的生命周期需要程序员自己管理，使用难度较大，所以在实际开发中通常不使用。

### [NSThread](https://www.jianshu.com/p/9c5ccb9e6191)

NSThread: 基于OC语言的API，使得其简单易用，面向对象操作。线程的声明周期由程序员管理，在实际开发中偶尔使用。

### [GCD](https://www.jianshu.com/p/8d671c0c36cc)
GCD: 基于C语言的API，充分利用设备的多核，旨在替换NSThread等线程技术。线程的生命周期由系统自动管理，在实际开发中经常使用。

### [NSOperation](https://www.jianshu.com/p/5b69f8e4860a)

NSOperation: 基于OC语言API，底层是GCD，增加了一些更加简单易用的功能，使用更加面向对象。线程生命周期由系统自动管理，在实际开发中经常使用。

## 三.三种多线程技术比较

1、NSThread
优点：NSThread 比其他两个轻量级，使用简单
缺点：需要自己管理线程的生命周期、线程同步、加锁、睡眠以及唤醒等。线程同步对数据的加锁会有一定的系统开销

2、GCD
GCD 是iOS 4.0以后才出现的并发技术
使用方式：将任务添加到队列（串行/并行（全局）），指定执行任务的方法，（同步（阻塞）/异步 ）
拿到主队列：dispatch_get_main_queu()
NSOperation无法做到的：1.一次性执行，2.延迟执行，3.调度组（op实现要复杂的多 ）

3、NSOperation
NSOperation iOS2.0的时候就出现了（当时不好用，后来苹果对其进行改造）
使用方式：将操作（异步执行）添加到队列（并发/全局）
拿到主队列：[NSOperationQueue mainQueue] 主队列，任务添加到主队列就会在主线程执行
提供了GCD不好实现的：1.最大并发数，2.暂停和继续，3.取消所有任务，4.依赖关系
GCD是比较底层的封装，我们知道较低层的代码一般性能都是比较高的，相对于NSOperationQueue。所以追求性能，而功能够用的话就可以考虑使用GCD。如果异步操作的过程需要更多的用户交互和被UI显示出来，NSOperationQueue会是一个好选择。如果任务之间没有什么依赖关系，而是需要更高的并发能力，GCD则更有优势。


## [四.多线程的锁](https://www.jianshu.com/p/27e29d86613a)

参考文章
[iOS多线程详解：概念篇](https://juejin.im/post/5ab4a3b0f265da237f1e3b37#heading-4)
https://bujige.net/blog/iOS-Complete-learning-GCD.html
https://juejin.im/post/5ab4a4466fb9a028d14107ff#heading-31
[https://juejin.im/post/5a9e57af6fb9a028df222555](https://juejin.im/post/5a9e57af6fb9a028df222555)
[https://juejin.im/post/5a0a92996fb9a0451f307479](https://juejin.im/post/5a0a92996fb9a0451f307479)

