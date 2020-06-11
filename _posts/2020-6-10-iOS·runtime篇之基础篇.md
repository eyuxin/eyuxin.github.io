---
layout:     post
title:      runtime解读
subtitle:   runtime解读只基础篇
date:       2020-6-10
author:     eyuxin
header-img: img/post-bg-e3.jpg
catalog: true
tags:
    - iOS
    - runtime
---



## runtime

Runtime 是指将数据类型的确定由**编译时**推迟到了**运行时**。它是一套底层的纯 C 语言 API，我们平时编写的 Objective-C 代码，最终都会转换成 runtime 的 C 语言代码。不过，runtime API 的实现是用 C++ 开发的（源码中的实现文件都是 `.mm` 文件）。

## objc_object

我们知道 Objective-C 是面向对象开发的，而 C 语言则是面向过程开发，这就需要**将面向对象的类转变成面向过程的结构体**。

Objective-C 对象是由 `id` 类型表示的，它本质上是一个指向 `objc_object` 结构体的指针。

```c
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;
// public & private method...
}
```

我们看到 `objc_object` 的结构体中只有一个对象，就是指向其类的 `isa` 指针。

当向一个对象发送消息时，runtime 会根据实例对象的 `isa` 指针找到其所属的类。

## objc_class

Objective-C 的类是由 `Class` 类型来表示的，它实际上是一个指向 `objc_class` 结构体的指针。

```c
typedef struct objc_class *Class;
```

objc_class结构体中有很多变量

```c
struct objc_class : objc_object {
    // 指向类的指针(位于 objc_object)
    // Class ISA;
    // 指向父类的指针
    Class superclass;
    // 用于缓存指针和 vtable，加速方法的调用
    cache_t cache;             // formerly cache pointer and vtable
    // 存储类的方法、属性、遵循的协议等信息的地方
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    // class_data_bits_t 结构体的方法，用于返回 class_rw_t 指针（）
    class_rw_t *data() { 
        return bits.data();
    }
    // other methods...
}

struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
    
    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    
    Class firstSubclass;
    Class nextSiblingClass;
    
    char *demangledName;

#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif
    // other methods
}
```

`objc_class` 继承自 `objc_object`，因此它也拥有了 `isa` 指针。除此之外，它的结构体中还保存了指向父类的指针、缓存、实例变量列表、方法列表、遵守的协议等。

>   关于super
>
>   1.  super不是指针 不指向父类的实例 
>
>   2.  在当前类打印self class 和 super class 都会返回当前类
>
>   3.  从runtime的底层API来看..调用[self class] 的时候是调用了objc_msgSend(self,@selector(class)),直接从当前实例里找class的实现; 
>
>       调用[super class]的时候是调用了objc_msgSendSuper

## 元类

元类（metaclass）是类对象的类，它的结构体和 `objc_class` 是一样的。

由于所有的类自身也是一个对象，我们可以向这个对象发送消息，比如调用类方法。那么为了调用类方法，这个类的 `isa` 指针必须指向一个包含类方法的一个 `objc_class` 结构体。而类对象中只存储了实例方法，却没有类方法，这就引出了元类的概念，元类中保存了创建类对象以及类方法所需的所有信息。

![](https://i.loli.net/2020/06/11/ylqkYdU4me2cMxO.png)

假如 `person` 对象也能调用 `sleep` 方法，那我们就无法区分它调用的就究竟是 `+ (void)sleep;` 还是 `- (void)sleep;`。

**类对象是元类的实例，类对象的 `isa` 指针指向了元类。**

当向对象发消息，runtime 会在这个对象所属类方法列表中查找发送消息对应的方法，但当向类发送消息时，runtime 就会在这个类的 meta class 方法列表里查找。所有的 meta class，包括 Root class，Superclass，Subclass 的 isa 都指向 Root class 的 meta class，这样能够形成一个闭环。

## Method

```c
typedef struct method_t *Method;
```

```
struct method_t {
    // 方法选择器
    SEL name;
    // 类型编码
    const char *types;
    // 方法实现的指针
    MethodListIMP imp;
}
```

所以 Method 和 SEL、IMP 的关系就是 Method = SEL + IMP + types。关于 types 的写法，参考 [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)。

## SEL

SEL 又称**方法选择器**，是一个指向 `objc_selector` 结构体的指针，也是 `objc_msgSend` 函数的第二个参数类型。

```c
typedef struct objc_selector *SEL;
```

方法的 `selector` 用于表示运行时方法的名称。代码编译时，会根据方法的名字（不包括参数）生成一个唯一的整型标识（ Int 类型的地址），即 SEL。

**一个类的方法列表中不能存在两个相同的 SEL**，这也是 **Objective-C 不支持重载**的原因。

**获取 SEL** 的方式有三种：

-   `sel_registerName` 函数
-   Objective-C 编译器提供的 `@selector()` 方法
-   `NSSeletorFromString()` 方法

## IMP

IMP 本质上就是一个函数指针，**指向方法实现的地址**。

```c
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```

参数说明：

-   id：指向 self 的指针（如果是实例方法，则是类实例的内存地址；如果是类方法，则是对应的类对象）
-   SEL：方法选择器
-   …：方法的参数列表

SEL 与 IMP 的关系类似于哈希表中 key 与 value 的关系。采用这种哈希映射的方式可以加快方法的查找速度。

## cache_t

`cache_t` 表示类缓存，是 object_class 的结构体变量之一。

```c
struct cache_t {
    // 存放方法的数组
    struct bucket_t *_buckets;
    // 能存储的最多数量
    mask_t _mask;
    // 当前已存储的方法数量
    mask_t _occupied;
    // ...
}
```

为了加速消息分发，系统会对方法和对应的地址进行缓存，就放在 `cache_t` 中。

实际运行中，大部分常用的方法都是会被缓存起来的，runtime 系统实际上非常快，接近直接执行内存地址的程序速度。

## category_t

```c
struct category_t {
    // 是指类名，而不是分类名
    const char *name;
    // 要扩展的类对象，编译期间是不会定义的，而是在运行时阶段通过 name 对应到相应的类对象
    classref_t cls;
    // 实例方法列表
    struct method_list_t *instanceMethods;
    // 类方法列表
    struct method_list_t *classMethods;
    // 协议列表
    struct protocol_list_t *protocols;
    // 实例属性
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    // 类（元类）属性列表
    struct property_list_t *_classProperties;
    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }
    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

这里涉及到一个经典问题：

**分类中可以添加实例变量/成员变量/属性吗？**

答案是：**分类中无法直接添加实例变量和成员变量, 但允许添加属性**

这是因为**在分类的结构体当中，没有“实例变量/成员变量”的结构，但是有“属性”的结构**。

那属性可以直接添加吗？并不是，虽然分类的 `.h` 中没有报错信息，`.m` 中却报出了如下的警告，且运行时会报错。

给分类手动添加 setter 和 getter 方法，这是一种有效的方案。

我们知道 `@property = ivar + setter + getter`。

可以通过 `objc_setAssociatedObject` 和 `objc_getAssociatedObject` **向分类中动态添加属性**，后面会说到。

## 消息传递的流程

![](https://i.loli.net/2020/06/11/IeURDJtqNl4LZdg.png)

也就是查找 IMP 的过程：

-   先从当前 class 的 cache 方法列表里去查找。
-   如果找到了，如果找到了就返回对应的 IMP 实现，并把当前的 class 中的 selector 缓存到 cache 里面。
-   如果类的方法列表中找不到，就到父类的方法列表中查找，一直找到 NSObject 类为止。
-   最后再找不到，就会进入动态方法解析和消息转发的机制(见下一篇文章)。

## runtime应用

#### 1. 关联对象动态添加属性

关联对象 runtime 提供了 3 个 API 接口：

```c
// 获取关联的对象
id objc_getAssociatedObject(id object, const void *key);
// 设置关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
// 移除关联的对象
void objc_removeAssociatedObjects(id object);
```

-   `object`：被关联的对象
-   `key`：关联对象的唯一标识
-   `value`： 关联的对象
-   `policy`：内存管理的策略



关于policy:

```c
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

#### 2.方法添加和方法交换

1.  方法添加

    ```c
    class_addMethod([self class], sel, (IMP)fooMethod, "v@:");
    ```

2.  方法交换 method swizzling

通过下面的方法获取方法的实现

```c
void class_getInstanceMethod(Class aClass, SEL aSelector);
```

然后通过下面的方法交换两个方法

```c
void method_exchangeImplementations(Method m1, Method m2);
```

示例代码：

```objective-c
#import <objc/runtime.h>
@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

有关method swizzling的注意事项：

1.  **在交换方法实现后记得要调用原生方法的实现（除非你非常确定可以不用调用原生方法的实现）**
2.  **避免冲突**：为分类的方法加前缀，一定要确保调用了原生方法的所有地方不会因为你交换了方法的实现而出现意想不到的结果。

#### 3.KVO的实现

KVO的大致实现原理：

当一个对象使用了KVO监听，iOS系统会修改这个对象的isa指针(**sa-swizzling**)，改为指向一个全新的通过Runtime动态创建的子类NSKVONotifyin_xxx，子类拥有自己的set方法实现，set方法实现内部会顺序调用**willChangeValueForKey方法、原来的setter方法实现、didChangeValueForKey方法，而didChangeValueForKey方法内部又会调用监听器的observeValueForKeyPath:ofObject:change:context:监听方法。**

#### 4.获取属性信息

**class_copyPropertyList**获取属性列表

**property_getName**获取属性名称

**property_getAttributes**获取属性类型

MJExtension是用的这一特性实现Model和Dictionary转换

```objective-c
- (instancetype)initWithDict:(NSDictionary *)dict {
    if (self = [self init]) {
        // 1、获取类的属性及属性对应的类型
        NSMutableArray * keys = [NSMutableArray array];
        NSMutableArray * attributes = [NSMutableArray array];
        /*
         * 例子
         * name = value3 attribute = T@"NSString",C,N,V_value3
         * name = value4 attribute = T^i,N,V_value4
         */
        unsigned int outCount;
        objc_property_t * properties = class_copyPropertyList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            objc_property_t property = properties[i];
            // 通过 property_getName 函数获得属性的名字
            NSString * propertyName = [NSString stringWithCString:property_getName(property) encoding:NSUTF8StringEncoding];
            [keys addObject:propertyName];
            // 通过 property_getAttributes 函数获得属性类型
            NSString * propertyAttribute = [NSString stringWithCString:property_getAttributes(property) encoding:NSUTF8StringEncoding];
            [attributes addObject:propertyAttribute];
        }
        // 立即释放properties指向的内存
        free(properties);

        // 2、根据类型给属性赋值
        for (NSString * key in keys) {
            if ([dict valueForKey:key] == nil) continue;
            [self setValue:[dict valueForKey:key] forKey:key];
        }
    }
    return self;
}
```

#### 5.获取实例变量 Ivar

**class_copyIvarList**获取实例变量列表

**ivar_getName**获取实例变量名称

我们可以利用它实现通用的NSCoding的归档和解档

```objective-c
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}
```

#### 6.利用消息转发机制

见[消息转发篇]([http://e-yuxin.com/2020/06/10/iOS-runtime%E7%AF%87%E4%B9%8B%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6/](http://e-yuxin.com/2020/06/10/iOS-runtime篇之消息转发机制/))



参考：



