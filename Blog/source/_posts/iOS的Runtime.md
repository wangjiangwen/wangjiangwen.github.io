---
title: iOS的Runtime
date: 2021-01-25 02:26:04
tags:
---

## Runtime特征

Runtime是Objective-C 面向对象、内存模型、消息派发、动态绑定等特性的实现者。

## 类和对象

```
typedef struct objc_class *Class;

struct objc_object{
   Class  isa; 
}
typedef struct objc_object *id;
```

```
struct objc_class {
	Class isa;
	Class super_class;
	const char *name;
	long version;
	long info;
	long instance_size;
	struct objc_ivar_list *ivar;
	struct objc_method_list *methodList;
	struct objc_cache *cache;
	struct objc_protocol_list *protocols
}
```

```
struct objc_method{
	SEL method_name;
	char *method_types;
	IMP method_imp;
}
```

```isa指针
struct objc_method_list{
	struct objc_method_list *obsolete;
	int method_cout;
	int space;
	/* variable length structure*/
	struct objc_method method_list[1]
}

```

```
struct objc_ivar{
		char *ivar_name;
		char *ivar_type;
		int ivar_offset;
		int space
}
```

```
struct objc_ivar_list{
	int ivar_cout;
	int space;
	struct objc_ivar ivar_list[1];
}
```

对象的struct比较简单，用*id作为objc_object结构体的指针别名，首个struct成员isa是Class类型的指针，正是该变量所属的类

Class类型也是struct，是结构体objc_class的别名；用于描述类的struct，首个struct成员也是isa，isa也是Class类型的指针变量，类的isa会指向称之为metaclass（元类）的struct，metaclass进一步抽象了类的特性

metaclass是Class类型，它的首个成员也是isa，指向的还是Class类型的指针变量，不同的是metaclass的isa最终指向的是它自身。

objc_class还存放了类的metadata（类的实例方法、类的实例变量以及类的超类指针等）。

Class类型的struct正好是一套嵌套的设计，它正体现了面向对象无限抽象的理念，最终实现上指向自己则是实际工作的需要。





## isa指针

8个字节，包含引用计数、是否有弱引用、是否weak



## KVC和KVO

isa swizzle 就是把当前某个实例对象的isa指针，指向一个新创建的中间累，在这个新类造的中间类上面做hook方法或者别的事情，这样不会影响这个类的其他实例对象，仅仅影响当前的实例对象。

### 动态创建一个class

* objc_allocateClassPair
* class_addMethod
* Class_addIvar
* Objc_registerClassPair

### isa swizzle 的应用-KVO

* 当某个类的属性对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的setter方法
* 派生类在被重写setter方法内实现真正的通知机制
* 如果原类为xxx，那么生成的派生类名为NSKVONotifying_xxx.
* 每个类对象中都有一个isa指针指向当前类，当一个类对象第一次被观察，那么系统会将isa指针指向动态生成的派生类，从而在给被监控属性赋值时执行的是派生类的setter方法。

![image-20210202025854904](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202025854904.png)

![image-20210202025908403](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202025908403.png)



## 方法决议：调用方法过程

### 查找过程

在Objective-C中，大部分的消息传递中的“消息”都会被编译器转化为：

•idobjc_msgSend(idself,SELop,...);

![image-20210202030726424](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202030726424.png)

![image-20210202030752852](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202030752852.png)

![image-20210202030806828](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202030806828.png)

![image-20210202030825009](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202030825009.png)



### objc_msgsend

objc_msgSend是使用汇编语言编写的。其原因是：其一是使用纯C是无法编写一个携带未知参数并跳转至任意函数指针的方法。单纯从语言角度来讲，也没有必要增加这样的功能。其二，对于objc_msgSend来说速度是最重要的，只用汇编来实现是十分高效的。

当然，我们也不希望所有的查询过程都是通过汇编来实现。一旦启用了非汇编语言那么就会降低速度。所以我们将消息分成了两个部分，即objc_msgSend的高速路径(fastpath)，此处所有的实现使用的是汇编语言，以及缓慢路径(slowpath)部分，此处的实现手段均为C语言。在高速路径中我们可以查询方法指针的缓存表，如果找到直接跳转。否则，则使用C代码来处理这次查询。



### 消息转发

![image-20210202031004905](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202031004905.png)

## Method Swizzle

method swizzle是利用Objective-C的runtime特性，动态改变SEL（方法编号）和IMP（方法实现）的对应关系，达到Objective-C方法调用流程改变的目的。

![image-20210202030509922](/Users/wangjiangjiao/Library/Application Support/typora-user-images/image-20210202030509922.png)

## 问题

1. 研究消息转发的汇编代码，找出是如何调用到forwardingTargetForSelector的？

   

2. Category可以覆写原来类的方法，那覆写后怎么调用到原来类的方法呢？

   

3. self和super

   

4. 方法调用过程：

   除了Runtime哪些

   能说出栈内存和函数调用之间的关系，能说出返回地址，入参压栈等概念，但具体细节不清楚

