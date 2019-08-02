---
layout:     post
title:      "Runtime系列（二）--Runtime的使用场景"
date:       2016-07-27
author:     "Haley_Wong"
catalog:    true
tags:
    - Runtime
---

Runtime 理解介绍的文章非常多，我只想讲讲Runtime 可以用在哪里，而我在项目里哪些地方用到了runtime。多以实际使用过程为主，来介绍runtime的使用。

**那么runtime 怎么使用？可以用在哪些场景下呢？**

首先，使用runtime 相关API，要`#import <objc/runtime.h>`

### 1.运行时获取某个类的属性或函数
运行时动态获取某个类的属性或者函数等，可以用来做很多事情，如json 解析、数据库结果解析、判断某个类的子类等。

##### 1.1解析、转化为Model
```
// 获取属性列表
objc_property_t * class_copyPropertyList(Class cls, unsigned int *outCount) 
// 获取属性名
const char *property_getName(objc_property_t property)
// 获取属性类型
const char *property_getAttributes(objc_property_t property)
```

> 以上方法可以用来：
> 
* 解析json数据，转化为Model对象。
* 解析数据库查询结果，转化为Model 对象。

这里有动态获取类的属性的示例代码片段：
```
    unsigned int outCount, i;
    objc_property_t *properties = class_copyPropertyList([self class], &outCount);
    for (i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
        //获取属性名
        NSString *propertyName = [NSString stringWithCString:property_getName(property) encoding:NSUTF8StringEncoding];
        //获取属性类型等参数
        NSString *propertyType = [NSString stringWithCString: property_getAttributes(property) encoding:NSUTF8StringEncoding];
        /*
         各种符号对应类型，部分类型在新版SDK中有所变化，如long 和long long
         c char         C unsigned char
         i int          I unsigned int
         l long         L unsigned long
         s short        S unsigned short
         d double       D unsigned double
         f float        F unsigned float
         q long long    Q unsigned long long
         B BOOL
         @ 对象类型 //指针 对象类型 如NSString 是@“NSString”
         propertyType，你可以打印出来，看看它是什么。
         要判断某个属性的类型，只需要[propertyType hasPrefix:@"Ti"]
          这代表它是int 类型。
         */
    }
    free(properties);
```
##### 1.2判断某个类的子类
有时候我们在程序中需要判断某个类是否是另一个类的子类。这个功能也可以利用runtime类实现，这里有示例代码：
```
    int numClasses;
    Class *classes = NULL;
    numClasses = objc_getClassList(NULL,0);
    
    if (numClasses >0 )
    {
        classes = (__unsafe_unretained Class *)malloc(sizeof(Class) * numClasses);
        numClasses = objc_getClassList(classes, numClasses);
        for (int i = 0; i < numClasses; i++) {
            if (class_getSuperclass(classes[i]) == [xxxxClass class]){
                id class = classes[i];
                // 执行某个方法 或者 做其他事情
                [class performSelector:@selector(xxxxMethod) withObject:nil];
            }
        }
        free(classes);
    }
```
以上两段示例代码摘自我之前写的FMDB Model 封装：[JKDBModel](https://github.com/Haley-Wong/JKDBModel)，你可以去看更详尽的解析和使用过程。
##### 1.3获取某个类的实例变量
如果你还需要获取某个类的实例变量做什么操作的话，可以使用如下这几个API：
```
// 获取实例变量数组
Ivar * class_copyIvarList(Class cls, unsigned int *outCount)
// 获取实例变量名称
const char * ivar_getName( Ivar ivar)
// 获取实例变量类型
const char * ivar_getTypeEncoding( Ivar ivar)
```

这面有获取实例变量的示例代码片段：
```
    unsigned int outCount, i;
    
    Ivar *ivaries = class_copyIvarList([Son class], &outCount);
    for (i = 0; i < outCount; i++) {
        Ivar ivar = ivaries[i];
        NSString *ivarName = [NSString stringWithCString:ivar_getName(ivar) encoding:NSUTF8StringEncoding];
        NSString *ivarType = [NSString stringWithCString:ivar_getTypeEncoding(ivar) encoding:NSUTF8StringEncoding];
        NSLog(@"名称:%@---类型：%@",ivarName,ivarType);
          /*
         各种符号对应类型，部分类型在新版SDK中有所变化，如long 和long long
         c char         C unsigned char
         i int          I unsigned int
         l long         L unsigned long
         s short        S unsigned short
         d double       D unsigned double
         f float        F unsigned float
         q long long    Q unsigned long long
         B BOOL
         @ 对象类型 //指针 对象类型 如NSString 是@“NSString”
         */
    }
    free(ivaries);

```
##### 1.4获取某个类的方法
获取某个类的方法，会包含这个类的property 的set 和get 方法，但是不包括父类的property set 和get 方法，不包括父类的方法（如果在当前类覆写，就包括）。

主要API：
```
// 获取方法数组
Method * class_copyMethodList(Class cls, unsigned int *outCount)
// 获取方法的 SEL
SEL method_getName( Method method)
// 获取方法名
const char* sel_getName(SEL aSelector)
```
获取方法数组的示例代码片段：
```
    unsigned int outMethodCount, j;
    Method *methods = class_copyMethodList([Son class], &outMethodCount);
    for (j = 0; j < outMethodCount; j++) {
        Method method = methods[j];
        SEL selector = method_getName(method);
        if (selector) {
            NSString *methodName = [NSString  stringWithCString:sel_getName(selector) encoding:NSUTF8StringEncoding];
            NSLog(@"方法:%@",methodName);
        }
    }
    free(methods);
```
### 2.运行时替换方法（Method Swizzling）
Method Swizzling 的使用需要谨慎，因为一不小心可能就会导致无法排查的Bug，毕竟它替换的是官方的API，有些API内部做了什么事情，很难完全把握。

**使用场景，需要监控用户经常打开的界面，以及在某界面停留的时长。**

我们可以怎么做呢？

写一个UIViewController 的Category，然后在类别中，添加自定义的方法:如-xxxviewDidAppear:和-xxxviewDidDisappear：方法，然后在-load 方法中，用自定义的方法替换原来的方法。
```
+ (void)load {
        static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];         
        // When swizzling a class method, use the following:
                    // Class class = object_getClass((id)self);

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling
- (void)xxx_viewWillAppear:(BOOL)animated {
        [self xxx_viewWillAppear:animated];
        NSLog(@"xxx_viewWillAppear: %@", self);
        // 在这里，我们可以发送一个消息到服务器，或者做其他事情等。
}
```

以上示例代码摘自：[Objective-C Runtime 运行时之四：Method Swizzling](http://southpeak.github.io/blog/2014/11/06/objective-c-runtime-yun-xing-shi-zhi-si-:method-swizzling/)

关于Method Swizzling，他是把两个方法的实现部分互换了。

比如上面我们调用-xxx_viewWillAppear：，因为-xxx_viewWillAppear: 和-viewWillAppear:的实现部分互换后，其实执行的时候，并不会执行上面的这个实现，而是调用-viewWillAppear:的内部实现。所以上面的代码，完全不会产生循环调用。

还是写段代码说明吧：
```
- (void)viewWillAppear:(BOOL)animated {
    NSLog(@"这是原来的方法");
}

- (void)xxx_viewWillAppear:(BOOL)animated {
    NSLog(@"xxx_viewWillAppear: %@", self);
    // 在这里，我们可以发送一个消息到服务器，或者做其他事情等。
}
```
假如上面这俩方法用method swizzling 替换后，我们调用`-xxx_viewWillAppear：`会打印`这是原来的方法`;而调用`-viewWillAppear:`会打印`xxx_viewWillAppear: `。这里需要细细体会一下。

关于Method Swizzling更多的注意点请看原文[Method Swizzling](http://southpeak.github.io/blog/2014/11/06/objective-c-runtime-yun-xing-shi-zhi-si-:method-swizzling/)

### 3.对象关联（Associated Objects）
对象关联（或称为关联引用）本来是Objective-C 运行时的一个重要特性，它能让开发者**对已经存在的类在扩展中添加自定义的属性**。

需要用的以下三个函数：
```
void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)

id objc_getAssociatedObject(id object, void *key)

void objc_removeAssociatedObjects(id object)
```
众所周知，OC 中的Category 中不能添加新的属性，但是我们通过`Associated Objects`可以间接的实现往类上添加自定义的属性。

不能添加属性的根本原因是不会帮我们自动添加对象的实例变量，也不会帮我们生成set 和get方法，虽然set /get 方法可以自己实现，但是没有实例变量来存储数据。

![](/img/blogs/runtime_practice/img_01.webp)

很容易看懂官方文档对参数的描述，但是key 需要注意一下：

通常推荐的做法是添加的属性最好是 static char类型的，当然更推荐是指针型的。通常来说该属性应该是常量、唯一的、在适用范围内用getter和setter访问到，所以通常我们这样写：

```
static char kAssociatedObjectKey;

objc_setAssociatedObject(self, &kAssociatedObjectKey, object, OBJC_ASSOCIATION_RETAIN_NONATOMIC)
objc_getAssociatedObject(self, &kAssociatedObjectKey);
```
当然，对于key 还有更好的做法，那就是selector。用selector 的示例在下面。

下面用代码演示如何在Category中添加一个新的属性。

**这是Son+AssociatedObject.h**

```
#import "Son.h"

@interface Son (AssociatedObject)

/** 家庭住址 */
@property (copy, nonatomic) NSString            *address;
/** 身高 */
@property (assign, nonatomic)   int             height;

@end
```

**这是Son+AssociatedObject.m**

```
#import "Son+AssociatedObject.h"
#import <objc/runtime.h>

@implementation Son (AssociatedObject)

- (void)setAddress:(NSString *)address
{
    objc_setAssociatedObject(self, @selector(address), address, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)address
{
    return objc_getAssociatedObject(self, @selector(address));
}

- (void)setHeight:(int)height
{
    NSNumber *heighNum = [NSNumber numberWithInt:height];
    objc_setAssociatedObject(self, @selector(height), heighNum, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (int)height
{
    NSNumber *heightNum = objc_getAssociatedObject(self, @selector(height));
    return heightNum.intValue;
}

@end
```
虽然上面有提到`void objc_removeAssociatedObjects(id object)`,但是不要轻易使用这个函数，因为它会移除所有的关联对象。我们一般要移除某个关联对象，只需要用`objc_setAssociatedObject`传入nil即可。

补充一个关联对象的使用场景：

**你在使用AlertView 或者ActionSheet的时候，有没有很苦恼不能在点击的代理方法中方便的获取到Model对象呢？**

除了在控制器中添加一个property 这种方式外；

我们也可以为AlertView 或者ActionSheet 添加一个关联对象，这样就可以在代理方法中方便的获取到Model 对象啦。

这里如果我们为AlertView 或者ActionSheet 添加Category来实现的话，代码跟上面为Son 添加类别基本一样，对象类型改为id 类型即可。

或者我们在控制器中调用的时候，添加关联对象也可以。这时候就用这种方式：

```
static char kAssociatedObjectKey;
objc_setAssociatedObject(self, &kAssociatedObjectKey, object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(self, &kAssociatedObjectKey);
```
UIAlertController 也跟上面一样。

关于`Associated Objects`的使用，有两个为Category扩展功能，使得Category中也能方便的添加属性以及相应的getter 和setter 的例子。

[OC 自动生成分类属性方法](http://nathanli.cn/2015/12/14/objective-c-%E5%85%83%E7%BC%96%E7%A8%8B%E5%AE%9E%E8%B7%B5-%E5%88%86%E7%B1%BB%E5%8A%A8%E6%80%81%E5%B1%9E%E6%80%A7/)

[一个库--DProperty](https://github.com/kissrobber/DProperty)

### 4.运行时动态创建一个类
我在某控制器中测试写了这么一个方法，来创建一个MyClass 类。项目中并不存在叫MyClass 的类文件。

```
- (void)createClass
{
    Class MyClass = objc_allocateClassPair([NSObject class], "MyClass", 0);
    // 1.添加一个叫name 类型为NSString的实例变量，第四个参数是对其方式，第五个参数是参数类型
    if (class_addIvar(MyClass, "name", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    }
    // 2.添加一个property
    // 这里需要注意，添加property之前需要先添加一个与之对应的实例变量
    if (class_addIvar(MyClass, "_address", sizeof(NSString *), 0, "@")) {
        NSLog(@"add ivar success");
    }
    
    objc_property_attribute_t type = {"T", "@\"NSString\""};
    objc_property_attribute_t ownership = { "C", "" };
    objc_property_attribute_t backingivar = { "V", "_address"};
    objc_property_attribute_t attrs[] = {type, ownership, backingivar};
    class_addProperty(MyClass, "address", attrs,2);
    
    // 3.添加函数， myclasstest是已经实现的函数，"v@:"这种写法见参数类型连接
    class_addMethod(MyClass, @selector(myclasstest:), (IMP)myclasstest, "v@:");
    // 4.注册这个类到runtime系统中就可以使用他了
    objc_registerClassPair(MyClass);
    // 5.生成了一个实例化对象
    id myobj = [[MyClass alloc] init];
    NSString *str = @"名字";
    // 6.给刚刚添加的变量赋值
    //    object_setInstanceVariable(myobj, "itest", (void *)&str);在ARC下不允许使用
    [myobj setValue:str forKey:@"name"];
    [myobj setValue:@"这是地址" forKey:@"address"];
    // 7.调用myclasstest方法，也就是给myobj这个接受者发送myclasstest这个消息
    [myobj myclasstest:10];
}

//这个方法实际上没有被调用,但是必须实现否则不会调用下面的方法
- (void)myclasstest:(int)a
{
    NSLog(@"啊啊啊啊啊");
}
//调用的是这个方法
static void myclasstest(id self, SEL _cmd, int a) //self和_cmd是必须的，在之后可以添加其他参数
{
    Ivar v = class_getInstanceVariable([self class], "name");
    //返回名为name的ivar的变量的值
    id o = object_getIvar(self, v);
    //成功打印出结果
    NSLog(@"name is %@", o);
    NSLog(@"参数 a is %d", a);
    
    objc_property_t property = class_getProperty([self class], "address");
    NSString *propertyName = [NSString stringWithUTF8String:property_getName(property)];
    id value = [self valueForKey:propertyName];
    NSLog(@"address is %@", value);
}
```

关于运行时创建一个新类，上面的注释已经写的很详细了。

Have Fun！

