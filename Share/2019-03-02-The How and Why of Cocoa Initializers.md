# Cocoa 初始化正确写法于原因

https://mikeash.com/pyblog/the-how-and-why-of-cocoa-initializers.html

为什么必须怎么写？


```
- init {
    if((self = [super init])) {
        // set up instance variables and whatever else here
    }
    return self;
}
```


## 调用 [super init]

* 如果想沿用父类的实现
* 遵守 designated initializer


## 检查 nil 

* 父类有可能初始化失败，如果父类的初始化参数不正确


## 赋值 （self = [super init]）

* [super init] 可能会返回nil 或者 不同的指针
* 如类簇，不同的类
* 单例，已存在的类
* 如果self为其他对象，防止Crash

> 如果直接继承**NSObject**则不需要准备标准写法，因为NSObject直接返回self

