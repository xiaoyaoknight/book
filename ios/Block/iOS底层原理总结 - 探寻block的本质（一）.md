# iOS底层原理总结 - 探寻block的本质（一）

## 面试题

1.  block的原理是怎样的？本质是什么？
2.  __block的作用是什么？有什么使用注意点？
3.  block的属性修饰词为什么是copy？使用block有哪些使用注意？
4.  block在修改NSMutableArray，需不需要添加__block？

首先对block有一个基本的认识

**`block本质上也是一个oc对象，他内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。`**

## 探寻block的本质
[Block-Demo](https://github.com/xiaoyaoknight/Block-Demo)
[快速写block](http://fuckingblocksyntax.com/)

首先写一个简单的block

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 10;
        void(^block)(int ,int) = ^(int a, int b){
            NSLog(@"this is block,a = %d,b = %d",a,b);
            NSLog(@"this is block,age = %d",age);
        };
        block(3,5);
    }
    return 0;
}
```

使用命令行将代码转化为c++查看其内部结构，与OC代码进行比较

**`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`**

![image.png](https://upload-images.jianshu.io/upload_images/1401554-533a970e897a24a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/680)


上图中将c++中block的声明和定义分别与oc代码中相对应显示。将c++中block的声明和调用分别取出来查看其内部实现。

### 定义block变量

```
// 定义block变量代码
void(*block)(int ,int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
```

上述定义代码中，可以发现，`block`定义中调用了`__main_block_impl_0`函数，并且将`__main_block_impl_0`函数的地址赋值给了`block`。那么我们来看一下`__main_block_impl_0`函数内部结构。

#### __main_block_imp_0结构体

![image.png](https://upload-images.jianshu.io/upload_images/1401554-aa86a3228f724b83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


`__main_block_imp_0`结构体内有一个同名构造函数`__main_block_imp_0`，构造函数中对一些变量进行了赋值最终会返回一个结构体。

那么也就是说最终将一个`__main_block_imp_0`结构体的地址赋值给了block变量

`__main_block_impl_0`结构体内可以发现`__main_block_impl_0`构造函数中传入了四个参数。
`(void *)__main_block_func_0`、
`&__main_block_desc_0_DATA`、
`age`、
`flags`
其中flage有默认值，也就说flage参数在调用的时候可以省略不传。而最后的 age(_age)则表示传入的_age参数会自动赋值给age成员，相当于`age = _age`。

接下来着重看一下前面三个参数分别代表什么。

#### (void *)__main_block_func_0

![image.png](https://upload-images.jianshu.io/upload_images/1401554-81455b322d830cd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


在`__main_block_func_0`函数中首先取出block中age的值，紧接着可以看到两个熟悉的`NSLog`，可以发现这两段代码恰恰是我们在block块中写下的代码。 那么`__main_block_func_0`函数中其实存储着我们block中写下的代码。而`__main_block_impl_0`函数中传入的是`(void *)__main_block_func_0`，也就说将我们写在block块中的代码封装成`__main_block_func_0`函数，并将`__main_block_func_0`函数的地址传入了`__main_block_impl_0`的构造函数中保存在结构体内。

#### &__main_block_desc_0_DATA

![image.png](https://upload-images.jianshu.io/upload_images/1401554-68716f6f4673f843.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


我们可以看到`__main_block_desc_0`中存储着两个参数，
`reserved`
`Block_size`，
并且`reserved`赋值为`0`而`Block_size`则存储着`__main_block_impl_0`的占用空间大小。最终将`__main_block_desc_0`结构体的地址传入`__main_block_func_0`中赋值给`Desc`。

#### age

age也就是我们定义的局部变量。因为在block块中使用到age局部变量，所以在block声明的时候这里才会将age作为参数传入，也就说block会捕获age，如果没有在block中使用age，这里将只会传入`(void *)__main_block_func_0`，`&__main_block_desc_0_DATA`两个参数。

这里可以根据源码思考一下为什么当我们在定义block之后修改局部变量age的值，在block调用的时候无法生效。

```
int age = 10;
void(^block)(int ,int) = ^(int a, int b){
     NSLog(@"this is block,a = %d,b = %d",a,b);
     NSLog(@"this is block,age = %d",age);
};
     age = 20;
     block(3,5); 
     // log: this is block,a = 3,b = 5
     //      this is block,age = 10
```

因为block在定义的之后已经将age的值传入存储在`__main_block_imp_0`结构体中
并在调用的时候将age从block中取出来使用，
因此在block定义`之后`对`局部变量`进行改变是`无法被block捕获的`。

#### 此时回过头来查看__main_block_impl_0结构体

![image.png](https://upload-images.jianshu.io/upload_images/1401554-71195c124133188e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


首先我们看一下`__block_impl`第一个变量就是`__block_impl`结构体。 来到`__block_impl`结构体内部

![image.png](https://upload-images.jianshu.io/upload_images/1401554-bc64b3d93827357e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


我们可以发现`__block_impl`结构体内部就有一个`isa`指针。因此可以证明`block本质上就是一个oc对象`。而在构造函数中将函数中传入的值分别存储在`__main_block_impl_0`结构体实例中，最终将结构体的地址赋值给block。

接着通过上面对`__main_block_impl_0`结构体构造函数三个参数的分析我们可以得出结论：

**1\. __block_impl结构体中isa指针存储着&_NSConcreteStackBlock地址，可以暂时理解为其类对象地址，block就是_NSConcreteStackBlock类型的。**

**2\. block代码块中的代码被封装成__main_block_func_0函数，FuncPtr则存储着__main_block_func_0函数的地址。**

**3\. Desc指向__main_block_desc_0结构体对象，其中存储__main_block_impl_0结构体所占用的内存。**

### 调用block执行内部代码

```
// 执行block内部的代码
((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 3, 5);
```

通过上述代码可以发现调用block是通过`block`找到`FunPtr`直接调用，通过上面分析我们知道block指向的是`__main_block_impl_0`类型结构体，但是我们发现`__main_block_impl_0`结构体中并不直接就可以找到`FunPtr`，而`FunPtr`是存储在`__block_impl`中的，为什么`block`可以直接调用`__block_impl`中的`FunPtr`呢？

重新查看上述源代码可以发现，`(__block_impl *)block`将`block`强制转化为`__block_impl`类型的，因为`__block_impl`是`__main_block_impl_0`结构体的第一个成员，相当于将`__block_impl`结构体的成员直接拿出来放在`__main_block_impl_0`中，那么也就说明`__block_impl`的内存地址就是`__main_block_impl_0`结构体的内存地址开头。所以可以转化成功。并找到`FunPtr`成员。

上面我们知道，`FunPtr`中存储着通过代码块封装的函数地址，那么调用此函数，也就是会执行代码块中的代码。并且回头查看`__main_block_func_0`函数，可以发现第一个参数就是`__main_block_impl_0`类型的指针。也就是说将`block`传入`__main_block_func_0`函数中，便于重中取出`block`捕获的值。

### 如何验证block的本质确实是`__main_block_impl_0`结构体类型。

通过代码证明一下上述内容： 同样使用之前的方法，我们按照上面分析的block内部结构自定义结构体，并将block内部的结构体强制转化为自定义的结构体，转化成功说明底层结构体确实如我们之前分析的一样。

```
struct __main_block_desc_0 { 
    size_t reserved;
    size_t Block_size;
};
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
// 模仿系统__main_block_impl_0结构体
struct __main_block_impl_0 { 
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int age;
};
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 10;
        void(^block)(int ,int) = ^(int a, int b){
            NSLog(@"this is block,a = %d,b = %d",a,b);
            NSLog(@"this is block,age = %d",age);
        };
// 将底层的结构体强制转化为我们自己写的结构体，通过我们自定义的结构体探寻block底层结构体
        struct __main_block_impl_0 *blockStruct = (__bridge struct __main_block_impl_0 *)block;
        block(3,5);
    }
    return 0;
}
```

通过打断点可以看出我们自定义的结构体可以被赋值成功，以及里面的值。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-ecaf686835f184fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来断点来到block代码块中，看一下堆栈信息中的函数调用地址。`Debuf workflow -> always show Disassembly`

![image.png](https://upload-images.jianshu.io/upload_images/1401554-1ed651df6662fd57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过上图可以看到地址确实和FuncPtr中的代码块地址一样。

## 总结

此时已经基本对block的底层结构有了基本的认识，上述代码可以通过一张图展示其中各个结构体之间的关系。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-41357a8216dadfe8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


block底层的数据结构也可以通过一张图来展示

![image.png](https://upload-images.jianshu.io/upload_images/1401554-5bbe96cd31a7d570.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## block的变量捕获

为了保证block内部能够正常访问外部的变量，block有一个变量捕获机制。

### 局部变量

#### auto变量

上述代码中我们已经了解过block对age变量的捕获。 auto自动变量，离开作用域就销毁，通常局部变量前面自动添加auto关键字。自动变量会捕获到block内部，也就是说block内部会专门新增加一个参数来存储变量的值。 auto只存在于局部变量中，访问方式为值传递，通过上述对age参数的解释我们也可以确定确实是值传递。

#### static变量

static 修饰的变量为指针传递，同样会被block捕获。

接下来分别添加aotu修饰的局部变量和static修饰的局部变量，重看源码来看一下他们之间的差别。

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        auto int a = 10;
        static int b = 11;
        void(^block)(void) = ^{
            NSLog(@"hello, a = %d, b = %d", a,b);
        };
        a = 1;
        b = 2;
        block();
    }
    return 0;
}
// log : block本质[57465:18555229] hello, a = 10, b = 2
// block中a的值没有被改变而b的值随外部变化而变化。
```

重新生成c++代码看一下内部结构中两个参数的区别。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-2735214b7ff6bf08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


从上述源码中可以看出，a,b两个变量都有捕获到block内部。但是a传入的是值，而b传入的则是地址。

为什么两种变量会有这种差异呢?
`因为自动变量可能会销毁`，block在执行的时候有可能自动变量已经被销毁了，那么此时如果再去访问被销毁的地址肯定会发生坏内存访问，因此对于`自动变量一定是值传递`而不可能是指针传递了。
而`静态变量`不会被销毁，所以完全可以`传递地址`。而因为传递的是值得地址，所以在block调用之前修改地址中保存的值，block中的地址是不会变得。所以值会随之改变。

### 全局变量

我们同样以代码的方式看一下block是否捕获全局变量

```
int a = 10;
static int b = 11;
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void(^block)(void) = ^{
            NSLog(@"hello, a = %d, b = %d", a,b);
        };
        a = 1;
        b = 2;
        block();
    }
    return 0;
}
// log hello, a = 1, b = 2
```

同样生成c++代码查看全局变量调用方式

![image.png](https://upload-images.jianshu.io/upload_images/1401554-4d5fc146eafdbc34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


通过上述代码可以发现，`__main_block_imp_0`并没有添加任何变量.
因此block不需要捕获全局变量，因为全局变量无论在哪里都可以访问。
**`局部变量因为跨函数访问所以需要捕获，全局变量在哪里都可以访问 ，所以不用捕获。`**

最后以一张图做一个总结

![image.png](https://upload-images.jianshu.io/upload_images/1401554-0f6ba5593075d64c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#####**总结：局部变量都会被block捕获，自动变量是值捕获，静态变量为地址捕获。全局变量则不会被block捕获**

#### 疑问：以下代码中block是否会捕获变量呢？

```
#import "Person.h"
@implementation Person
- (void)test
{
    void(^block)(void) = ^{
        NSLog(@"%@",self);
    };
    block();
}
- (instancetype)initWithName:(NSString *)name
{
    if (self = [super init]) {
        self.name = name;
    }
    return self;
}
+ (void) test2
{
    NSLog(@"类方法test2");
}
@end
```

同样转化为c++代码查看其内部结构

![image.png](https://upload-images.jianshu.io/upload_images/1401554-fe8ae6d366802344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


上图中可以发现，self同样被block捕获，接着我们找到test方法可以发现，test方法默认传递了两个参数self和_cmd。而类方法test2也同样默认传递了类对象self和方法选择器_cmd。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-9fe987bb6241fc60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


不论对象方法还是类方法都会默认将self作为参数传递给方法内部，既然是作为参数传入，那么self肯定是局部变量。上面讲到局部变量肯定会被block捕获。

接着我们来看一下如果在block中使用成员变量或者调用实例的属性会有什么不同的结果。

```
- (void)test
{
    void(^block)(void) = ^{
        NSLog(@"%@",self.name);
        NSLog(@"%@",_name);
    };
    block();
}
```

![image.png](https://upload-images.jianshu.io/upload_images/1401554-3f7e094f81e50e64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


上图中可以发现，即使`block中使用的是实例对象的属性，block中捕获的仍然是实例对象，并通过实例对象通过不同的方式去获取使用到的属性`。

## block的类型

block对象是什么类型的，之前稍微提到过，通过源码可以知道block中的isa指针指向的是_NSConcreteStackBlock类对象地址。那么block是否就是_NSConcreteStackBlock类型的呢？

我们通过代码用class方法或者isa指针查看具体类型。

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // __NSGlobalBlock__ : __NSGlobalBlock : NSBlock : NSObject
        void (^block)(void) = ^{
            NSLog(@"Hello");
        };

        NSLog(@"%@", [block class]);
        NSLog(@"%@", [[block class] superclass]);
        NSLog(@"%@", [[[block class] superclass] superclass]);
        NSLog(@"%@", [[[[block class] superclass] superclass] superclass]);
    }
    return 0;
}
```

打印内容:

![image.png](https://upload-images.jianshu.io/upload_images/1401554-e995179842ca8cd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上述打印内容可以看出block最终都是继承自NSBlock类型，而NSBlock继承于NSObjcet。那么block其中的isa指针其实是来自NSObject中的。这也更加印证了block的本质其实就是OC对象。

### block的3种类型
```
__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）
__NSStackBlock__ （ _NSConcreteStackBlock ）
__NSMallocBlock__ （ _NSConcreteMallocBlock ）
```
![image.png](https://upload-images.jianshu.io/upload_images/1401554-9f82b2bd4eee3c75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

通过代码查看一下block在什么情况下其类型会各不相同

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1\. 内部没有调用外部变量的block
        void (^block1)(void) = ^{
            NSLog(@"Hello");
        };
        // 2\. 内部调用外部变量的block
        int a = 10;
        void (^block2)(void) = ^{
            NSLog(@"Hello - %d",a);
        };
       // 3\. 直接调用的block的class
        NSLog(@"%@ %@ %@", [block1 class], [block2 class], [^{
            NSLog(@"%d",a);
        } class]);
    }
    return 0;
}
```

通过打印内容确实可以发现block的三种类型

![image.png](https://upload-images.jianshu.io/upload_images/1401554-4471de9286e025a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


但是我们上面提到过，上述代码转化为c++代码查看源码时却发现block的类型与打印出来的类型不一样，c++源码中三个block的isa指针全部都指向_NSConcreteStackBlock类型地址。

我们可以猜测`runtime运行时`过程中也许对类型进行了转变。最终类型当然以runtime运行时类型也就是我们`打印出的类型为准`。

### block在内存中的存储

通过下面一张图看一下不同block的存放区域

![image.png](https://upload-images.jianshu.io/upload_images/1401554-088e879cf639d8c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上图中可以发现，根据block的类型不同，block存放在不同的区域中。 
`__NSGlobalBlock__`(数据段中)直到程序结束才会被回收，不过我们很少使用到`__NSGlobalBlock__`类型的block，因为这样使用block并没有什么意义。

`__NSStackBlock__`类型的block存放在栈中，我们知道栈中的内存由系统自动分配和释放，作用域执行完毕之后就会被立即释放，而在相同的作用域中定义block并且调用block似乎也多此一举。

`__NSMallocBlock__`是在平时编码过程中最常使用到的。存放在堆中需要我们自己进行内存管理。

### block是如何定义其类型

block是如何定义其类型，依据什么来为block定义不同的类型并分配在不同的空间呢？首先看下面一张图

![image.png](https://upload-images.jianshu.io/upload_images/1401554-a0d3ba30d2c81c9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接着我们使用代码验证上述问题，首先关闭ARC回到MRC环境下，因为ARC会帮助我们做很多事情，可能会影响我们的观察。

![调试MRC环境.png](https://upload-images.jianshu.io/upload_images/1401554-9dee622eaded442d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

```
// MRC环境！！！
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // Global：没有访问auto变量：__NSGlobalBlock__
        void (^block1)(void) = ^{
            NSLog(@"block1---------");
        };   
        // Stack：访问了auto变量： __NSStackBlock__
        int a = 10;
        void (^block2)(void) = ^{
            NSLog(@"block2---------%d", a);
        };
        NSLog(@"%@ %@", [block1 class], [block2 class]);
        // __NSStackBlock__调用copy ： __NSMallocBlock__
        NSLog(@"%@", [[block2 copy] class]);
    }
    return 0;
}
```

查看打印内容

![image.png](https://upload-images.jianshu.io/upload_images/1401554-e458146b09dfdfb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过打印的内容可以发现正如上图中所示。 
- 没有访问auto变量的block是`__NSGlobalBlock__`类型的，存放在数据段中。 
- 访问了auto变量的block是`__NSStackBlock__`类型的，存放在栈中。 
- `__NSStackBlock__`类型的block调用copy成为`__NSMallocBlock__`类型并被复制存放在堆中。

上面提到过`__NSGlobalBlock__`类型的我们很少使用到，因为如果不需要访问外界的变量，直接通过函数实现就可以了，不需要使用block。

但是`__NSStackBlock__`访问了auto变量，并且是存放在栈中的，上面提到过，栈中的代码在作用域结束之后内存就会被销毁，那么我们很有可能block内存销毁之后才去调用他，那样就会发生问题，通过下面代码可以证实这个问题。

```
void (^block)(void);
void test()
{
    // __NSStackBlock__
    int a = 10;
    block = ^{
        NSLog(@"block---------%d", a);
    };
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        test();
        block();
    }
    return 0;
}
```

此时查看打印内容

![image.png](https://upload-images.jianshu.io/upload_images/1401554-fe2ad2ae38ed65e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以发现a的值变为了不可控的一个数字。为什么会发生这种情况呢？因为上述代码中创建的block是`__NSStackBlock__`类型的，因此block是存储在栈中的，那么当test函数执行完毕之后，栈内存中block所占用的内存已经被系统回收，因此就有可能出现乱得数据。查看其c++代码可以更清楚的理解。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-15021752a41ba1c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


为了避免这种情况发生，可以通过`copy`将`__NSStackBlock__`类型的block转化为`__NSMallocBlock__`类型的block，将`block存储在堆中`，以下是修改后的代码。

```
void (^block)(void);
void test()
{
    // __NSStackBlock__ 调用copy 转化为__NSMallocBlock__
    int age = 10;
    block = [^{
        NSLog(@"block---------%d", age);
    } copy];
    [block release];
}
```

此时在打印就会发现数据正确

![image.png](https://upload-images.jianshu.io/upload_images/1401554-a0c8e2bd542bd3ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那么其他类型的block调用copy会改变block类型吗？下面表格已经展示的很清晰了。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-c529fa12ead34d5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以在平时开发过程中MRC环境下经常需要使用copy来保存block，将栈上的block拷贝到堆中，即使栈上的block被销毁，堆上的block也不会被销毁，需要我们自己调用release操作来销毁。而在ARC环境下系统会自动调用copy操作，使block不会被销毁。

## ARC帮我们做了什么

在ARC环境下，编译器会根据情况自动将栈上的block进行一次copy操作，将block复制到堆上。

**什么情况下ARC会自动将block进行一次copy操作？** 以下代码都在RAC环境下执行。

### 1\. block作为函数返回值时

```
typedef void (^Block)(void);
Block myblock()
{
    int a = 10;
    // 上文提到过，block中访问了auto变量，此时block类型应为__NSStackBlock__
    Block block = ^{
        NSLog(@"---------%d", a);
    };
    return block;
}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Block block = myblock();
        block();
       // 打印block类型为 __NSMallocBlock__
        NSLog(@"%@",[block class]);
    }
    return 0;
}
```

看一下打印的内容

![image.png](https://upload-images.jianshu.io/upload_images/1401554-50adb6d533f6eb12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**上文提到过，如果在block中访问了auto变量时，block的类型为`__NSStackBlock__`，上面打印内容发现blcok为`__NSMallocBlock__`类型的，并且可以正常打印出a的值，说明block内存并没有被销毁。**

**上面提到过，block进行copy操作会转化为`__NSMallocBlock__`类型，来讲block复制到堆中，那么说明ARC在 block作为函数返回值时会自动帮助我们对block进行copy操作，以保存block，并在适当的地方进行release操作。**

### 2\. 将block赋值给__strong指针时

block被强指针引用时，ARC也会自动对block进行一次copy操作。

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // block内没有访问auto变量
        Block block = ^{
            NSLog(@"block---------");
        };
        NSLog(@"%@",[block class]);
        int a = 10;
        // block内访问了auto变量，但没有赋值给__strong指针
        NSLog(@"%@",[^{
            NSLog(@"block1---------%d", a);
        } class]);
        // block赋值给__strong指针
        Block block2 = ^{
          NSLog(@"block2---------%d", a);
        };
        NSLog(@"%@",[block1 class]);
    }
    return 0;
}
```

查看打印内容可以看出，当block被赋值给__strong指针时，ARC会自动进行一次copy操作。

![image.png](https://upload-images.jianshu.io/upload_images/1401554-ced0c131d19a65dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 3\. block作为Cocoa API中方法名含有usingBlock的方法参数时

例如：遍历数组的block方法，将block作为参数的时候。

```
NSArray *array = @[];
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {

}];
```

### 4\. block作为GCD API的方法参数时

例如：GDC的一次性函数或延迟执行的函数，执行完block操作之后系统才会对block进行release操作。

```
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{

});        
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{

});
```

## block声明写法

通过上面对MRC及ARC环境下block的不同类型的分析，总结出不同环境下block属性建议写法。

MRC下block属性的建议写法

**`@property (copy, nonatomic) void (^block)(void);`**

ARC下block属性的建议写法

**`@property (strong, nonatomic) void (^block)(void);`** **`@property (copy, nonatomic) void (^block)(void);`**

下一篇：[iOS底层原理总结 - 探寻block的本质（二）](https://www.jianshu.com/p/9418ccdbb9b0)

参考文章：https://juejin.im/post/5b0181e15188254270643e88
