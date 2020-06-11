---
layout:     post
title:      消息转发机制
subtitle:   iOS中的消息转发机制
date:       2020-6-10
author:     eyuxin
header-img: img/post-bg-e2.jpg
catalog: true
tags:
    - iOS
    - 消息转发机制
---



## 消息发送

OC中的方法调用本质上是一个**消息发送**的过程，所有继承自NSObject的类都可以通过runtime的```objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)```来进行消息发送。

消息发送的流程大致如下：

1.  检测这个selector的target是不是nil，OC允许我们对一个nil对象执行任何方法不会Crash，因为运行时会被忽略掉。
2.  查找这个类的实现IMP，先从cache里查找，如果找到了就运行对应的函数去执行相应的代码。
3.  如果cache中没有找到就找类的方法列表中是否有对应的方法。
4.  如果类的方法列表中找不到就到父类的方法列表中查找，一直找到NSObject类为止。
5.  如果还是没找到就要开始进入**动态方法解析**和**消息转发**

再来回归到objc_msgSend这个方法， 默认隐藏了两个参数： self和_cmd。`self`指向对象本身，`_cmd`指向方法本身。例如：

```objective-c
[array insertObject:foo atIndex:5];
```

```c
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);
```



## 方法解析和消息转发

没有方法的实现，程序会在运行时挂掉并抛出 `unrecognized selector sent to …` 的异常。但在异常抛出前，Objective-C 的运行时会给你三次拯救程序的机会：

-   Method resolution
-   Fast forwarding
-   Normal forwarding

#### 1.动态方法解析： Method Resolution

**resolveInstanceMethod/resolveClassMethod + class_addMethod**

首先，Objective-C 运行时会调用 `+ (BOOL)resolveInstanceMethod:`或者 `+ (BOOL)resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数并返回 YES， 那运行时系统就会重新启动一次消息发送的过程。

```objective-c
void fooMethod(id obj, SEL _cmd)  
{
    NSLog(@"Doing foo");
}

+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if(aSEL == @selector(foo:)){
        class_addMethod([self class], aSEL, (IMP)fooMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod];
}
```

这里的v代表返回类型是void, @代表self的类型是id, :代表_cmd类型是SEL。更详细的官方文档 [在这里](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

#### 2.快速转发： Fast Forwarding

**forwardingTargetForSelector**

runtime系统允许我们替换消息的接收者为其他对象。通过`- (id)forwardingTargetForSelector:(SEL)aSelector`方法。如果此方法返回的是nil 或者self,则会进入消息转发机制（`- (void)forwardInvocation:(NSInvocation *)invocation`），否则将会向返回的对象重新发送消息。

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if(aSelector == @selector(foo:)){
        return [[BackupClass alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

#### 3.完整消息转发 Normal Forwarding

**forwardInvocation+methodSignatureForSelector**

与上面不同，可以理解成完整消息转发，是可以代替快速转发做更多的事。

```objective-c
- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL sel = invocation.selector;
    if([alternateObject respondsToSelector:sel]) {
        [invocation invokeWithTarget:alternateObject];
    } else {
        [self doesNotRecognizeSelector:sel];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *methodSignature = [super methodSignatureForSelector:aSelector];
    if (!methodSignature) {
        methodSignature = [NSMethodSignature signatureWithObjCTypes:"v@:*"];
    }
    return methodSignature;
}
```

这一步可以讲一个消息发给多个对象。

## 消息转发实际场景

#### 1.防止crash

```objective-c
@implementation NSObject (CrashLogHandle)

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    //方法签名
    return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"NSObject+CrashLogHandle---在类:%@中 未实现该方法:%@",NSStringFromClass([anInvocation.target class]),NSStringFromSelector(anInvocation.selector));
}

@end
```

因为在category中复写了父类的方法，会出现警告 解决办法就是在Xcode的Build Phases中的资源文件里，在对应的文件后面 -w ，忽略警告。

#### 2.苹果系统API迭代造成API不兼容的新方案

一般处理API版本兼容问题可以用respondsToSelector或者根据iOS版本来区分

我们其实也可以用消息转发机制来处理

例如：tableView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;

这个API是iOS11才支持的版本。iOS11以下要使用

setAutomaticallyAdjustsScrollViewInsets:NO

我们可以这样做：

```objective-c
#import "UIScrollView+Forwarding.h"

@implementation UIScrollView (Forwarding)

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector { // 1
    
    NSMethodSignature *signature = nil;
    if (aSelector == @selector(setContentInsetAdjustmentBehavior:)) {
        signature = [UIViewController instanceMethodSignatureForSelector:@selector(setAutomaticallyAdjustsScrollViewInsets:)];
    }else {
        signature = [super methodSignatureForSelector:aSelector];
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation { // 2
    
    BOOL automaticallyAdjustsScrollViewInsets  = NO;
    UIViewController *topmostViewController = [self cm_topmostViewController];
    NSInvocation *viewControllerInvocation = [NSInvocation invocationWithMethodSignature:anInvocation.methodSignature]; // 3
    [viewControllerInvocation setTarget:topmostViewController];
    [viewControllerInvocation setSelector:@selector(setAutomaticallyAdjustsScrollViewInsets:)];
    [viewControllerInvocation setArgument:&automaticallyAdjustsScrollViewInsets atIndex:2]; // 4
    //这里要设置第三个参数 前两个参数已经确定了
    [viewControllerInvocation invokeWithTarget:topmostViewController]; // 5
}

@end
```



#### 3.模拟多继承

OC中没有多继承， 如果想实现类似Java的多继承可以采用：

1.  遵守多个协议
2.  利用消息转发机制

>   虽然转发可以实现继承功能，但是NSObject还是必须表面上很严谨，像`respondsToSelector:`和`isKindOfClass:`这类方法只会考虑继承体系，不会考虑转发链。

#### 4.热更新

**JSPatch**的热更新功能利用了完整的消息转发实现了获取参数的问题

参考：

[iOS消息转发机制](https://juejin.im/post/5ae96e8c6fb9a07ac85a3860)

