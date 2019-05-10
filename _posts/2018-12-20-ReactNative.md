---
layout: post
title:  "ReactNative小玩杂谈"
date:   2018-12-20
tag: iOS
permalink: /page/ReactNative.html
type: legacy

---


ReactNative 是移动端推出的一种跨平台方案,其大致原理是通过你写的JS代码通过JavaScriptCore去翻译成相应的OC语言.

玩了一个多月的ReactNative,基本上学习了一遍环境配置流程,布局,传值,网络和一些基本框架,刚开始写非常的不适应ES6和React的写法,而且遇到一个问题会纠结很久,当然我明白学习是一个过程.

缺点:

- 我遇到的，编译的时候库会不稳定,有时候莫名其妙连接某个库失败，也有可能是我配置的问题
- 相对来说学习成本还是非常高的, 要学习一门框架React 和 一门新的语言JS，而且还要熟悉ES6的语法


优点:

- 它的布局写法真的是非常的快,当你熟悉以后,编写代码的速度比iOS快的不是一点半点
- 组件化路由比较方便，现成的react-navigation
- 热更新，iOS现在为数不多的热更新解决方案了吧


个人觉得RN的定位应该在于一些APP中需要及时热更新页面或者一些基础的UI布局界面不涉及音视频或者高性能需求等页面中,可以作为一种尝试。


记录一个坑爹的问题

```

    <TouchableOpacity style={styles.instructions}
                          onPress={() => { this.doSomething() }}>
          
    </TouchableOpacity>


    <TouchableOpacity style={styles.instructions}
                          onPress={this.doSomething()}>
        
    </TouchableOpacity>
    
  doSomething(){
    alert('哈哈哈哈')
  }
  
  
  上面第一种写法会在调用的时候执行，相当于增加了bind，而下面这种写法会立即执行，

```

练习代码链接：[https://github.com/kylinuranus/ReactNative_MMM](https://github.com/kylinuranus/ReactNative_MMM)


