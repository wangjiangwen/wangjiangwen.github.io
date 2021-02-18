---
title: iOS内存管理
date: 2021-01-25 02:24:53
tags:
---

## iOS内存管理演进：MRC -> ARC

### 显示内存管理

* 内存被提前释放（悬停指针 dangling pointer）
* 内存永远无法释放（内存泄漏）

### 垃圾回收

* 判断哪些对象可以回收
  - 引用计数法
  - 可达性分析法
* 垃圾回收算法
  - 标记-清楚法
  - 复制算法
  - 标记整理法
  - 分代收集法

### MRC

manual retain count

#### 基本规则

严格遵守引用计数规则，保持Retain和Release之间的平衡

对象创建好之后，保留计数至少为1

引用计数大于0时对象继续存活；引用计数为0时对象销毁

#### 特点

* 自己创建的对象，自己持有
* 非自己生成的对象，自己可以持有
* 非自己持有的对象无法释放
* 不需要自己持有的对象时，释放它

![image-20210131210806868](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131210806868.png)

![image-20210131210956332](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131210956332.png)



### 延迟释放

```
- (NSObject *)object
{
	NSObject *object = [[NSObject alloc] init];
	//[object release];
	return o;
}
```



#### Autorelease

延迟释放对象

* 手动调用autorelease方法
* 工厂方法返回值
  - -（id）initWithURL:(NSURL*)url;

![image-20210131212832924](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131212832924.png)



#### AutoReleasePool and Runloop

* 每一个线程，包括主线程，都会拥有一个专属的NSRunLoop对象，并且会在有需要的时候自动创建
* 每个event loop开始前，系统会自动创建一个autoreleasepool，并在event loop结束时drain。
* 每一个线程都会维护自己的autoreleasepool堆栈。换句话说 autoreleasepool是与线程紧密相关的，每一个autoreleasepool只对应一个线程。

#### Autorelease的副作用

内存峰值

for循环中含有大量

```
for(int i = 0;i < 100; i++){
	NSError *error;
	NSString *fileContents =  [NSStringstringWithContentsOfURL:urlArray[i]
	                       encoding:NSUTF8StringEncodingerror:&error];

}
```



解决方案:

```
for(int i = 0;i < 100; i++){
	NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
	NSError *error;
	NSString *fileContents =  [NSStringstringWithContentsOfURL:urlArray[i]
	                       encoding:NSUTF8StringEncodingerror:&error];
	[pool release];

}
```



### 内存进化式

![image-20210131213925802](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131213925802.png)



### ARC

ARC是Objective-C编译器的特性，而不是运行时特性或者垃圾回收机制，ARC所做的只不过是在代码编译时为你自动在合适的位置插入retain/release或autorelease，就如同之前MRC时你所做的那样。

![image-20210131214033719](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131214033719.png)



#### ARC-retainable object pointer

* Block指针
* Objective-C的对象指针
* 用\_\_attribute\_\_(NSobject)标记的对象

alloc、new、copy、mutableCopy、init

#### ARC-对象修饰符

* \_\_strong   默认使用
* \_\_weak   不持有对象；当对象没有强引用时自动置为nil
* \_\_unsafe\_unretained 不持有对象；当对象没有强引用时不会置为nil
* \_\_autoreleasing  表明传引用的参数(id*)在返回时是autorelease的

###  Foundation和Foundation的内存管理

Toll-Free Bridging

* __bridge只是声明类型转变，但是不做内存管理规则的转变。
* bridge_retained表示将指针类型转变的同时，将内存管理的责任由原来的Objective-C交给CoreFoundation来处理，也就是，将ARC转变为MRC。
* __bridge_transfer表示将管理的责任由CoreFoundation转交给Objective-C，即将管理方式由MRC转变为ARC。



## 原理

### 为什么weak变量会自动置为nil

* weak表（散列表） weak\_table\_t
* 对象地址为key值   weak_entry_t中的范型对象
* Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

1、初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。
2、添加引用时：objc_initWeak函数会调用 objc_storeWeak() 函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。
3、释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

![image-20210131230124233](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131230124233.png)



### Autorelease底层原理

Autorelease是双向链表，里面有Pool边界（哨兵对象）

![image-20210131231015415](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131231015415.png)

![image-20210131230959249](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131230959249.png)

![image-20210131231031171](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131231031171.png)

![image-20210131231119373](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210131231119373.png)

常见的面试题目：

1.@weakify&@strongify

weakify 实际上生成一个weak对象 weak_self,

strongify 生成一个self，强引用 weak_self

2.MRC和ARC的内存管理

引用计数：引用计数非0时，对象存在；为0时对象销毁

MRC是手动管理引用计数，基于谁持有谁释放、非自己持有的对象无法释放

ARC

3.Block不加@strongify会怎么样

如果不加strongify，里面的self依然会引起内存泄漏

并且在多线程下，会存在野指针风险

4.多层block嵌套需要加多个@strongify么

需要

5.weak的实现原理

weak 哈希表（weak_table_t）、对象地址为key（weak_entry_t object范型）、哈希表中的weak变量（是一个数组或者链表）

6.为什么会有Autorelease？autorelease的原理

延迟释放（return的值得）

双向链表，和线程一一对应，

应用场景：使用容器的block版本的枚举器时，内部会自动添加一个AutoreleasePool

7.Objective-C里面的数组，放进里面的元素都会被持有么？如果被数组持有，那怎么才能做一个不持有元素的数组呢？

会被持有，NSPointerArray

8.NSDictionary的key和value内存语义都是怎样的？•在ARC环境下，函数的返回值已经不再使用Autorelease机制了，知道它是怎么实现的么？但是如果调用方是MRC环境，ARC下面的函数返回值会自动启动Autorelease，知道它是怎么实现的么？

散列表

由此可以看到，编译器在这里再一次做了优化。一般情况下ARC方法的返回值会通过调用objc_autoreleaseReturnValue把对象注册到autorelease pool，此时，编译器插了一条objc_retainAutoreleasedReturnValue();通过上面的学习知道这会跳过把对象注册到autoreleasepool的操作。紧接着是__autoreleasing修饰符产生的objc_autorelease(obj)方法，把对象注册到autorelease pool。这样，就可以避免因为__autoreleasing修饰符和array方法的搭配使用，而把一个对象注册到autorelease pool两次。

9.Toll-FreeBridging是怎么做到的，什么样的CoreFoundation和Foundation对象才能做Toll-FreeBridging？

CoreFundation -> Foundation

CF的对象能够bridge到Foundation指针的原因：Foundation的相关类采用class cluster的设计模式，比如NSString实际是子类__NSCFString实现，而__NSCFString则是用CFString来实现的。

Foundation -> CoreFundation

Foundation的对象能够bridge CF到指针的原因：NSString的运行时实际创建的是__NSCFString等子类，子类的length等方法实现实际是把self作为参数传递给CF方法。

10.如果对象持有一个block，在block里面又使用到了对象的一个实例变量，会形成循环引用么？如果以一个临时变量引用实例变量，在block里使用临时变量又如何呢？
对象的引用计数存在对象内存布局里还是全局的引用计数表里，为什么，两种方式各有什么优缺点？

1）会的

2）临时变量也会持有的

3）引用计数对象内存里，isa指针



11.设计内存管理程序