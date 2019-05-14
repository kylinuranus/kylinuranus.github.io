---
layout: post
title:  "JavaScriptCore与Objective-C的交互"
date:   2018-12-23
tag: iOS
permalink: /page/JavaScriptCore.html
type: legacy

---


了解到RN的底层原理大概是JavaScriptCore那一套,所以大致整理一下

### JavaScriptCore基本介绍

- JSValue: 代表一个JavaScript实体，一个JSValue可以表示很多JavaScript原始类型例如boolean, integers, doubles，甚至包括对象和函数。
- JSManagedValue: 本质上是一个JSValue，但是可以处理内存管理中的一些特殊情形，它能帮助引用技术和垃圾回收这两种内存管理机制之间进行正确的转换。
- JSContext: 代表JavaScript的运行环境，你需要用JSContext来执行JavaScript代码。所有的JSValue都是捆绑在一个JSContext上的。
- JSExport: 这是一个协议，可以用这个协议来将原生对象导出给JavaScript，这样原生对象的属性或方法就成为了JavaScript的属性或方法，非常神奇。
- JSVirtualMachine: 代表一个对象空间，拥有自己的堆结构和垃圾回收机制。大部分情况下不需要和它直接交互，除非要处理一些特殊的多线程或者内存管理问题。


### OC调用JS

```
调用一个hello.js 代码 
var obj = "js demo"

function printHello() {
   return "hello world";
}
   
```


```
 NSString *scriptPath = [[NSBundle mainBundle] pathForResource:@"hello" ofType:@"js"];
 NSString *scriptString = [NSString stringWithContentsOfFile:scriptPath encoding:NSUTF8StringEncoding error:nil];
 JSContext *context = [[JSContext alloc] init];
 [context evaluateScript:scriptString];
 JSValue *function = context[@"printHello"];
 JSValue *returnValue = [function callWithArguments:@[]];
 JSValue *obj = context[@"obj"];
    
 NSLog(@"---%@",[returnValue toString]);  ----hello world
 NSLog(@"---%@",[obj toString]); ----js demo 
  
```

 热更新方案: 服务器通过下发字符串形式的JS代码,OC通过JavaScriptCore调用,再通过swizzle方式或者消息转发方式达到修改代码的效果
 
 
### js调用OC

 ```
   注册后,相当于上面的js函数
   self.context[@"printHello"]  = ^(){
        return "hello world";
    };
 ```
 
### JSExport

js调用具体方法，该方法在oc端具体实现

```
@protocol MYJSExport <JSExport>

- (void)addA:(NSInteger)a B:(NSInteger)b;

@end

NS_ASSUME_NONNULL_BEGIN

@interface exportModel : NSObject <MYJSExport>

@end

@implementation exportModel

- (void)addA:(NSInteger)a B:(NSInteger)b
{
    NSInteger sum = a + b;
    NSLog(@"---sum = %ld",sum);
}
@end 


```


```
exportModel *model = [[exportModel alloc] init];
context[@"obj1"] = model;
    
NSString *str = @"obj1.addAB('1','2')";
[context evaluateScript:str]; 
 
```

### 关于循环引用

定义的时候不能在block里直接调用JSValue和JSContext,前者最好通过参数传进block，后者用currentContext去取，不能用weak
因为js的内存管理机制和OC不一样

```
self.jsValue = [JSValue new];
self.context[@"block"] = ^(){
    // 循环引用
    // context = self.context;
    JSContext *context = [JSContext currentContext];
    NSLog(@"%@",value);
};
```

```
self.jsValue = [JSValue new];
 self.context[@"block"] = ^(){
     JSValue *value = self.jsValue;
     NSLog(@"%@",value);
 };

```
