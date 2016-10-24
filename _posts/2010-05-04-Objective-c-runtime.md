---
layout: post
title: Objective-c Runtime | Objective-c 运行时机制
tags:
- Objective-c
- iOS
categories: iOS
description: 
---
iPhone伪代码（有些地方可以参考一下） ： http://wwwmain.gnustep.org/resources/downloads.php?site=http://ftpmain.gnustep.org/pub/gnustep/   
类是面向对象的一个核心的概念，可以利用类来构建对象，类决定了从该类生成的实例变量和方法。

   从一个类生成的对象拥有同样的行为，各自拥有单独的成员树形，Class-Level 的变量可以声明为static类型，这些变量可以在类方法外面定义，这样在类的任意方法里面都可以访问这个变量；也可以在指定的方法中定义，这样只能在方法内访问。不管哪种情况，所有该类对象都共同拥有同一个static variable,任何一个对象改变static variable的值，其他对象也相应的改变。

   指定一个类A，通过继承该类可以定义一个新类B，新类B中有和父类不同的状态或者行为。在Objective-c中，A叫做B的super class.Objective-c 是一种支持单继承的语言，所以一个类最多只能有一个Super class.

   新类的可以在编译阶段生成，也可以在运行时生成。编译阶段，一个新类通过@interface、@implementation 指令生成。 在接下来的介绍里，你会看到运行时利用运行时库动态的产生class.
   在iPhone OS上，一个类也是一个对象，既然说一个类是一个对象，那么必然有需要一个blueprint来产生类。用来产生类对象的类叫做metaclass.就像对象是类的实例一样，每个类对象也是一个metaclass的实例。metaclass是编译阶段生成的，并且开发者很少直接与该类打交道。
   实际开发中，你通过给一个对象发送“消息”来访问该对象的方法。“消息”包括方法的名字和方法的参数。方法是由C函数实现的，方法的实现与“消息”是不同的。一个对象可以在运行的时候，动态的改变对于给定消息的实现，甚至可以在自己不需要处理或者处理不了的情况下，把消息传递给其他的对象。
   在iPhone OS中，The root class is NSObject.所有的对象必须是这个类或者子类的一个实例，需要注意的是NSObject 没有 superclass.

 1.1 头文件
  在进入runtime system functions之前，需要在你的code 里面包含如下的头文件
```objective-c
#import <objc/objc.h>
#import <objc/runtime.h>
#import <objc/message.h>
```
 Furthermore, you need to target the device, not the simulator, in your build.

1.2 关于NSObject Class 
 问题是什么是NSObject?如果你检查NSObject.h 头文件，你会注意到NSObject 的定义如下：


```objective-c
@interface NSObject <NSObject>{
Class isa;
}
```
   再来看看protocol NSObject，这里面基本囊括了NSObject实现的方法。
```objective-c
@protocol NSObject
- (Class) class;           
- (Class) superclass;          
- (BOOL) isEqual: (id)anObject;    
- (BOOL) isKindOfClass: (Class)aClass; 
- (BOOL) isMemberOfClass: (Class)aClass;
- (BOOL) isProxy;          
- (unsigned) hash;         
- (id) self;               
- (id) performSelector: (SEL)aSelector;
 
- (id) performSelector: (SEL)aSelector
      withObject: (id)anObject;
 
- (id) performSelector: (SEL)aSelector
      withObject: (id)object1
      withObject: (id)object2;
 
- (BOOL) respondsToSelector: (SEL)aSelector;
 
- (BOOL) conformsToProtocol: (Protocol*)aProtocol;
- (id) retain;             
- (id) autorelease          ;
- (oneway void) release;       
- (unsigned) retainCount;      
- (NSZone*) zone;          
- (NSString*) description;     
@end
```