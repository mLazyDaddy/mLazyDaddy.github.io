---
layout: post
title:  "objc invokes function"
date:   2019-04-30 12:00:00
categories: iOS
---


#Objective-C方法调用

（以下分析基于objc4-723）

（本文主要介绍在Objective-C中，方法是如何定义的、存在哪里、又是如何被调用的。）

在Objective-C里，调用实例方法或者类方法，实际上是对实例对象或者类对象发送消息，例如，以下是创建一个NSObject对象的代码：

```
NSObject *obj = [NSObject new];
```
编译器会转化成如下形式

```
NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
```

##objc_msgSend
给对象发送消息并获取返回值。

[声明](https://developer.apple.com/documentation/objectivec/1456712-objc_msgsend)

```
id objc_msgSend(id self, SEL op, ...);
```

###参数
* `self`：接收消息的对象；
* `op`：处理消息的方法选择器（类型为SEL，下面有解释）


我们提取[NSObject new]转化后的关键代码

```
objc_msgSend((id)objc_getClass("A"), sel_registerName("new"))
```

其中 `objc_getClass("A")` 是通过类名“A”获取对应类的类对象（Class，见备注）,
`sel_registerName("new")`是通过方法名“new”获取对应的方法选择器。


##类(Class)
这里不对Class作详细分析，只介绍方法相关的部分。Objective-C里类的信息存放在名为Class的结构体指针中（Class的结构体中有个字段叫isa，其类型也是Class）。运行时会为每个类创建两个结构相同的Class对象，一个叫类对象，一个叫元类对象(metaClass)，类对象的isa指向元类对象，类的实例对象（我们在代码里通过`new`等方法创建的对象）的isa指向类对象。在类中定义的实例方法存在类对象中，类方法存在元类对象中。

下面定义了MyClass类，包含一个实例方法和一个类方法

```
@interface MyClass : NSObject
- (void)methodA;
+ (void)methodB;
@end
```

通过clang -rewrite-objc转化代码可以看到生成了两个存放方法列表的结构体：

* `_OBJC_$_INSTANCE_METHODS_MyClass`。保存实例方法`methodA`
* `_OBJC_$_CLASS_METHODS_MyClass`。保存类方法`methodB`


```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_INSTANCE_METHODS_MyClass __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"methodA", "v16@0:8", (void *)_I_MyClass_methodA}}
};

static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CLASS_METHODS_MyClass __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"methodB", "v16@0:8", (void *)_C_MyClass_methodB}}
};
```
下面的代码可以看到

* 结构体`_OBJC_METACLASS_RO_$_MyClass` 保存了类方法方法列表`_OBJC_$_INSTANCE_METHODS_MyClass`。

* 结构体`_OBJC_CLASS_RO_$_MyClass` 保存了实例方法列表`_OBJC_$_CLASS_METHODS_MyClass`。

```
static struct _class_ro_t _OBJC_METACLASS_RO_$_MyClass __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1, sizeof(struct _class_t), sizeof(struct _class_t), 
	(unsigned int)0, 
	0, 
	"MyClass",
	(const struct _method_list_t *)&_OBJC_$_CLASS_METHODS_MyClass,
	0, 
	0, 
	0, 
	0, 
};

static struct _class_ro_t _OBJC_CLASS_RO_$_MyClass __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, sizeof(struct MyClass_IMPL), sizeof(struct MyClass_IMPL), 
	(unsigned int)0, 
	0, 
	"MyClass",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_MyClass,
	0, 
	0, 
	0, 
	0, 
};
```
最后`_OBJC_METACLASS_RO_$_MyClass`存放在元类对象中，而`_OBJC_CLASS_RO_$_MyClass`存放在类对象中。

在上文提到了调用类A的类方法`new`的转换代码

```
objc_msgSend((id)objc_getClass("A"), sel_registerName("new"))
```
中，`objc_getClass("A")`返回的是类A的类对象。

```
id objc_msgSend(id self, SEL op, ...);
```
`objc_msgSend`会根据第一个参数self找到self的isa指向的对象cls，如果self是一个类对象，cls是元类对象；如果self是一个类的实例对象，则cls是类对象。然后`objc_msgSend`从cls的方法列表中找到SEL对应的函数指针IMP，执行该函数。

## SEL
[声明](https://developer.apple.com/documentation/objectivec/sel)

```
typedef struct objc_selector *SEL;
```
SEL代表方法在运行时中的名字，可以简单地认为是一个C字符串。

例如在Xcode执行下面语句

```
SEL sel = @selector(alloc);
```
可以看到sel的值是"alloc"

所有的SEL保存在一个名为namedSelectors的哈希表中，声明如下

```
static NXMapTable *namedSelectors;
```
其中key是含有方法名字的字符串，value为SEL。

## IMP
[声明](https://developer.apple.com/documentation/objectivec/objective-c_runtime/imp?language=objc)

```
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```
IMP是函数指针，指向函数实现的起始地址。

## Method

[声明](https://developer.apple.com/documentation/objectivec/method?language=objc)

```
typedef struct objc_method *Method;

```

而objc_method的结构如下

```
struct objc_method {
    SEL method_name;
    char * method_types;//类型编码
    IMP method_imp;
}  
```
Method实际上代表了方法，包含了方法的基本信息。


我们回顾上文提到的用于保存MyClass的实例方法列表的结构体`_OBJC_$_INSTANCE_METHODS_MyClass`

```

static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[1];
} _OBJC_$_INSTANCE_METHODS_MyClass __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    1,
    {{(struct objc_selector *)"methodA", "v16@0:8", (void *)_I_MyClass_methodA}}
};
```

请注意结构体最后一个字段`method_list`就是存放`objc_method`结构的数组。这里只存了一个方法，我们把这个方法的结构和值组合起来如下

```
struct objc_method {
    SEL method_name;//"methodA"
    char * method_types;//"v16@0:8"
    IMP method_imp;//(void *)_I_MyClass_methodA
} 
```
method_types是经过类型编码得到的字符串。“v”表示方法`methodA `的返回值是void，“16”表示方法参数的总长度是16；每个方法有两个默认参数，第一个是`self`,第二个是`_cmd`(类型为SEL),"@"表示参数`self`是一个object，后面跟着的“0”是参数的起始位置，`_cmd`是SEL用“:”表示，又因为`self`的长度是8（object即Class，长度为8），所以`_cmd`的起始位置是"8"。而SEL的长度是8，两个参数的总长度就是16。

##最后

查找方法的整个流程、动态实现方法、消息转发这几个部分网上已有很多文章解释，所以不再阐述。
