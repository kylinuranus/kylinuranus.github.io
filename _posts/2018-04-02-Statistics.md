---
layout: post
title:  "埋点方案的整理"
date:  2018-04-02
tag: iOS
permalink: /page/Statistics.html
type: legacy
comments: true

---


  
### 直接埋点
  
  缺点:埋点代码直接和业务代码混淆在一起,增加开发者的负担
  
### 简易的AOP埋点
  
  原理:基于Aspects框架(方法交换)
  
  优点:埋点代码与业务逻辑解耦了
  
  缺点: 
  
  1. 如果要新增埋点代码,则要跟着App一起重新发布 或者 热修复
  
  2. 主要是Aspects的性能消耗
  
  - 在hook过程之中调用了其他多个函数的处理
  - invoke block 的性能消耗
  - 作者也在注释中写道，不适合对每秒钟超过1000次的方法增加切面代码。
  - 用其他方式对 Aspect hook 过的方法进行hook时,如直接替换为新的IMP,新hook得到的原始实现是_objc_msgForward,之前的aspect_hook会失效,新的hook也将执行异常。
  

### 可视化埋点
 
 通过可视化工具配置采集节点，在前端自动解析配置并在合适的时机上报埋点数据，从而实现所谓的“无痕埋点”,应该是现在主流的埋点方式
 
 难点:
 - 事件标识(UI标识): 通过swizzle 各类的生命周期，UIControl的点击,手势,代理等进行方法交换，然后根据一定的规则拼接成唯一标识(比如target+action)，配置表从后台下载
 - 数据关联问题:
    同一控件不同业务 ---- 可以根据tag来判断
    
 缺点：代码还是写死的，如修改埋点还是需要重新发布。

  
 - Swizzle UITableViewDelegate
 
 ``` 
 +(void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        SEL originalAppearSelector = @selector(setDelegate:);
        SEL swizzingAppearSelector = @selector(user_setDelegate:);
        [MethodSwizzingTool swizzingForClass:[self class] originalSel:originalAppearSelector swizzingSel:swizzingAppearSelector];
    });
}



-(void)user_setDelegate:(id<UITableViewDelegate>)delegate
{
    [self user_setDelegate:delegate];
    
    SEL sel = @selector(tableView:didSelectRowAtIndexPath:);
    
    SEL sel_ =  NSSelectorFromString([NSString stringWithFormat:@"%@/%@/%ld", NSStringFromClass([delegate class]), NSStringFromClass([self class]),self.tag]);
    
    
    //因为 tableView:didSelectRowAtIndexPath:方法是optional的，所以没有实现的时候直接return
    if (![self isContainSel:sel inClass:[delegate class]]) {
        
        return;
    }
    
    
    BOOL addsuccess = class_addMethod([delegate class],
                                      sel_,
                                      method_getImplementation(class_getInstanceMethod([self class], @selector(user_tableView:didSelectRowAtIndexPath:))),
                                      nil);
    
    //如果添加成功了就直接交换实现， 如果没有添加成功，说明之前已经添加过并交换过实现了
    if (addsuccess) {
        Method selMethod = class_getInstanceMethod([delegate class], sel);
        Method sel_Method = class_getInstanceMethod([delegate class], sel_);
        method_exchangeImplementations(selMethod, sel_Method);
    }
}


//判断页面是否实现了某个sel
- (BOOL)isContainSel:(SEL)sel inClass:(Class)class {
    unsigned int count;
    
    Method *methodList = class_copyMethodList(class,&count);
    for (int i = 0; i < count; i++) {
        Method method = methodList[i];
        NSString *tempMethodString = [NSString stringWithUTF8String:sel_getName(method_getName(method))];
        if ([tempMethodString isEqualToString:NSStringFromSelector(sel)]) {
            return YES;
        }
    }
    return NO;
}


// 由于我们交换了方法， 所以在tableview的 didselected 被调用的时候， 实质调用的是以下方法：
-(void)user_tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
   ....
}
 
 
 ```
 

### 无痕动态埋点(完全后台控制下发埋点)


  基本思路:
  
  - 如何实现事件标识 
    
     取每个控件自身的ID、类名以及位于所属父组件的Index等特征信息，并逐级向上遍历找到根节点。根节点一般是手动标记的，如果没有标记则默认是视图层次树的顶层节点。最后，将遍历产生的路径上所有节点的特征信息组合在一起，就是这个事件的标识

  - 业务数据关联
    
    1. 后台进行关联处理
    2. 获取到跳转路径,遍历当前控制器的所有属性,拿到需要的业务数据
 
  

  
###  参考链接

  - iOS统计打点那些事: [https://limboy.me/tech/2015/09/09/ios-analytics.html](https://limboy.me/tech/2015/09/09/ios-analytics.html)
  
  - 美团点评前端无痕埋点实践: [https://tech.meituan.com/2017/03/02/mt-mobile-analytics-practice.html](https://tech.meituan.com/2017/03/02/mt-mobile-analytics-practice.html)
  
  - iOS 无痕埋点方案探究:[https://blog.csdn.net/SandyLoo/article/details/81202105](https://blog.csdn.net/SandyLoo/article/details/81202105)
 
  - iOS无埋点数据SDK实践之路:[https://www.jianshu.com/p/69ce01e15042](https://www.jianshu.com/p/69ce01e15042)