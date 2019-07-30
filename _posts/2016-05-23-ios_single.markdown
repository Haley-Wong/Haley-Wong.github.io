---
layout:     post
title:      "iOS中的单例你用对了么？"
date:       2016-05-23
author:     "Haley_Wong"
catalog:    true
tags:
    - 设计模式
---

单例模式怎么定义的，可能在不同的语言，不同的书中不完全一样，但是概况开来都应该是:`一个类有且仅有一个实例，并且自行实例化向整个系统提供。`

因此，首先你可能需要确定你是真的需要一个单例类，还是说仅仅是需要一个方便调用的实例化方法。如果你是真的需要一个单例类，那么你就应该确保这个单例类，有且仅有一个实例（不管怎么操作都只能获取到这个实例）。

最近看到一些github上的单例使用，别人的用法，有一些思考，然后写demo测试了下，就这个简单的单例也有一些坑呢，希望能给他人一些提醒。

#Objective-C中的单例

我们通常在OC中实现一个单例方法都是这样：
```
static HLTestObject *instance = nil;

+ (instancetype)sharedInstance
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[[self class] alloc] init];
    });
    return instance;
}
```

可是这样就可以了么？我做了如下测试：
```
    HLTestObject *objct1 = [HLTestObject sharedInstance];
    NSLog(@"%@",objct1);
    HLTestObject *objc2 = [[HLTestObject alloc] init];
    NSLog(@"%@",objc2);
    HLTestObject *objc3 = [HLTestObject new];
    NSLog(@"%@",objc3);
```

看到这个测试，你想到打印结果了么？结果是这样的：
```
2016-05-23 12:52:57.095 PractiseProject[3579:81998] <HLTestObject: 0x7fcf39515510>
2016-05-23 12:52:57.095 PractiseProject[3579:81998] <HLTestObject: 0x7fcf395c4b70>
2016-05-23 12:52:57.095 PractiseProject[3579:81998] <HLTestObject: 0x7fcf395c6890>
```

很明显，通过三种方式创建出来的是不同的实例对象，这就违背了单例类`有且仅有一个实例`的定义。
为了防止别人不小心利用alloc/init方式创建示例，也为了防止别人故意为之，我们要保证不管用什么方式创建都只能是同一个实例对象，这就得重写另一个方法，实现如下：

```
+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    return instance;
}
```

再次用上面的测试代码，结果是这样的：
```
2016-05-23 12:57:37.396 PractiseProject[3618:83975] <HLTestObject: 0x7f88b9488ac0>
2016-05-23 12:57:37.396 PractiseProject[3618:83975] <HLTestObject: 0x7f88b9488ac0>
2016-05-23 12:57:37.396 PractiseProject[3618:83975] <HLTestObject: 0x7f88b9488ac0>
```

好像用不同的构造方法，获取的都是同一个对象，你以为这样就完了？

一般我们的类里肯定都会有一些属性，然后我就添加了两个property：
```
@property (assign, nonatomic)   int height;
@property (strong, nonatomic)   NSObject    *object;
@property (strong, nonatomic)   NSMutableArray  *arrayM;
```

而一些对象类的初始化，或者基础类型的默认值设置都是在init方法里，就像这样：
```
- (instancetype)init
{
    self = [super init];
    if (self) {
        _height = 10;
        _object = [[NSObject alloc] init];
        _arrayM = [[NSMutableArray alloc] init];
    }
    return self;
}
```

我重写了HLTestObject类的`description`方法：
```
- (NSString *)description
{
    NSString *result = @"";
    result = [result stringByAppendingFormat:@"<%@: %p>",[self class], self];
    result = [result stringByAppendingFormat:@" height = %d,",self.height];
    result = [result stringByAppendingFormat:@" arrayM = %p,",self.arrayM];
    result = [result stringByAppendingFormat:@" object = %p,",self.object];
    return result;
}
```
还是用上面的测试代码，测试结果是这样的：
```
2016-05-23 13:14:43.684 PractiseProject[3781:92758] <HLTestObject: 0x7f8a5b458450> height = 20, arrayM = 0x7f8a5b422940, object = 0x7f8a5b4544e0,
2016-05-23 13:14:43.684 PractiseProject[3781:92758] <HLTestObject: 0x7f8a5b458450> height = 10, arrayM = 0x7f8a5b4552e0, object = 0x7f8a5b45a710,
2016-05-23 13:14:43.684 PractiseProject[3781:92758] <HLTestObject: 0x7f8a5b458450> height = 10, arrayM = 0x7f8a5b459770, object = 0x7f8a5b4544e0,
```
可以看到，尽管使用的是同一个示例，可是他们的property值却不一样。

因为尽管没有为示例重新分配内存空间，但是因为又执行了init方法，会导致property被重新初始化。
所以我们需要修改单例的实现。

**第一种:**

可以将property的初始化或者默认值设置放到dispatch_once 的block内部：

```
static HLTestObject *instance = nil;

+ (instancetype)sharedInstance
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[[self class] alloc] init];
        instance.height = 10;
        instance.object = [[NSObject alloc] init];
        instance.arrayM = [[NSMutableArray alloc] init];
    });
    return instance;
}

+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    return instance;
}

- (NSString *)description
{
    NSString *result = @"";
    result = [result stringByAppendingFormat:@"<%@: %p>",[self class], self];
    result = [result stringByAppendingFormat:@" height = %d,",self.height];
    result = [result stringByAppendingFormat:@" arrayM = %p,",self.arrayM];
    result = [result stringByAppendingFormat:@" object = %p,",self.object];
    return result;
}
```

来看看测试结果：

```
2016-05-23 13:29:14.856 PractiseProject[3909:99058] <HLTestObject: 0x7fa72270c570> height = 20, arrayM = 0x7fa722716c10, object = 0x7fa7227140e0,
2016-05-23 13:29:14.856 PractiseProject[3909:99058] <HLTestObject: 0x7fa72270c570> height = 20, arrayM = 0x7fa722716c10, object = 0x7fa7227140e0,
2016-05-23 13:29:14.856 PractiseProject[3909:99058] <HLTestObject: 0x7fa72270c570> height = 20, arrayM = 0x7fa722716c10, object = 0x7fa7227140e0,
```

**第二种：**

```
static HLTestObject *instance = nil;

+ (instancetype)sharedInstance
{
    return [[self alloc] init];
}

- (instancetype)init
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super init];
        instance.height = 10;
        instance.object = [[NSObject alloc] init];
        instance.arrayM = [[NSMutableArray alloc] init];
    });
    return instance;
}

+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    return instance;
}

- (NSString *)description
{
    NSString *result = @"";
    result = [result stringByAppendingFormat:@"<%@: %p>",[self class], self];
    result = [result stringByAppendingFormat:@" height = %d,",self.height];
    result = [result stringByAppendingFormat:@" arrayM = %p,",self.arrayM];
    result = [result stringByAppendingFormat:@" object = %p,",self.object];
    return result;
}
```

测试结果：

```
2016-05-23 13:31:44.824 PractiseProject[3939:100662] <HLTestObject: 0x7fa9da711a70> height = 20, arrayM = 0x7fa9da707ca0, object = 0x7fa9da70a940,
2016-05-23 13:31:44.825 PractiseProject[3939:100662] <HLTestObject: 0x7fa9da711a70> height = 20, arrayM = 0x7fa9da707ca0, object = 0x7fa9da70a940,
2016-05-23 13:31:44.825 PractiseProject[3939:100662] <HLTestObject: 0x7fa9da711a70> height = 20, arrayM = 0x7fa9da707ca0, object = 0x7fa9da70a940,
```

> 注意：
以上代码均是使用ARC的方式管理内存，如果你还在使用MRC（这也太不与时俱进了）。那你还需要重写 retain 和release方法，防止示例引用计数的改变。

# Swift中的单例

利用Swift中的一些特性，Swift中的单例可以超级简单，like this:

```
class HLTestObject: NSObject {
    static let sharedInstance = HLTestObject(); 
}
```

可是这样就可以了么？同样写一段测试代码：

```
let object1 = HLTestObject.sharedInstance;
print(object1);

let object2 = HLTestObject();
print(object2);
```

打印结果却是这样的：
```
<SwiftProject.HLTestObject: 0x7f90ebc74e50>
<SwiftProject.HLTestObject: 0x7f90ebe5cf40>
```
所以，我们必须禁用到构造方法：
```
class HLTestObject: NSObject {
    static let sharedInstance = HLTestObject();

    private override init() {
    }
}
```
如果有实例属性需要初始化，就可以这样：

```
class HLTestObject: NSObject {
    
    var height = 10;
    
    var arrayM: NSMutableArray
    
    var object: NSObject
    
    static let sharedInstance = HLTestObject();
    
    private override init() {
        object = NSObject()
        arrayM = NSMutableArray()
        super.init()
    }
}
```

当然，由于Swift的特性，在Swift中创建单例的方式也不止一种，需要注意的是要确保该类有且仅有一个实例就OK了。

Have Fun!


