# 理解上下文 Context

***Context***，也就是上下文，是 Android 中常用的类，Android 中的四大组件都会涉及到 Context。

## Context 的关联类
**Context** 的使用场景大概分为两类：

* 使用 Context 调用方法，比如启动 Activity、访问资源、调用系统服务等等。
* 作为参数传入 Context ，比如创建 Dialog 、弹出 Toast 等等。

Activity、Service 和 Application 都间接的继承自 Context ，因此一个应用程序中 Context 的数量等于 Activity 和 Service 的数量加1。

Context 是一个抽象类，内部定义了很多静态常量和方法，它的具体实现类是 **ContextImpl** 。而与 Context 相关联的类还有 **ContextWrapper** 、**ContextThemeWrapper** 等等。

![Context的关联类](https://i.imgur.com/0R851iw.png)

ContextImpl 和 ContextWrapper 都直接继承自 Context ，ContextWrapper 内部定义了 Context 类型的 mBase 对象，而 mBase 具体指向 ContextImpl。设计上使用了装饰者模式， ContextWrapper 是装饰类，它对 ContextImpl 进行包装，ContextWrapper 主要起到方法传递的作用，ContextWrapper 中几乎所有的方法调用都最终调用了 ContextImpl 中的方法。ContextThemeWrapper、Application、Service 都继承自 ContextWrapper，因此它们都可以通过 mBase 来调用 Context 中的方法。ContextThemeWrapper 中含有跟主题相关的方法，因此，需要用到主题的 Activity 继承了 ConteThemeWrapper。

Context 采用装饰模式主要有以下有点：

* 使用者能够方便的使用 Context。
* 如果 ContextImpl 发生了变化，装饰类 ContextWrapper 不需要做任何修改。
* ContextImpl 的实现不会暴露给使用者，而使用者也无需关心这个。
* 通过组合的方式扩展 ContextImpl 的功能，在运行时选择不同的装饰类实现不同的功能。

## Application Context 的创建过程
![Application Context 的创建过程的流程图](https://i.imgur.com/SKP91Kx.png)

## ApplicationContext 的获取过程
![](https://i.imgur.com/T1ggUkB.png)

## Activity Context 的创建过程
![Activity Context 的创建过程的流程图](https://i.imgur.com/SdtaCGN.png)

## Service Context 的创建过程
![](https://i.imgur.com/yo8ubQX.png)