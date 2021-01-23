### Android不同架构分析

Android项目一般有4种主流架构：不分离、MVC、MVP、MVVM，这几种架构是如何一步步演化，不同架构之间有何联系，新项目要如何选用合适框架，本文会试着分享一点自己的见解。

#### （1）不分离

布局写在layout，逻辑控制、网络请求、UI变化全部写在Activity或Fragment，这是Android最原始的开发方式。

###### 优点：代码总量少，逻辑直观清晰，函数跳转简单，适合个人开发

###### 缺点：多人维护困难，单个Activity或Fragment代码量会轻易超过上千行

可优化方案：a.用ButterKnife注解框架，减少获取View的代码，集中点击事件处理

b.用Kotlin

#### （2）MVC

这种架构与不分离相比，多出了一个Model，专门用于逻辑处理，在Activity中直接实例化Model，然后当Activity需要网络请求或进行一些逻辑操作时，直接调用Model处理，之后如果UI产生了变动，通知Activity刷新。

那要怎么通知Activity刷新呢？

个人认为在Android开发中有两种多见的办法，可以起到通知作用，一个是接口，另一个是组件通信（EventBus的组件通信本质上是观察者模式）。

那么在MVC中，采用哪一种更好呢？在网络上多见的方案都是在Model中定义一个接口，在Activity中实现这个接口，这样当Model层想通知刷新UI时，就用接口方法去通知Activity刷新。

那么在MVC中是否可以用组件通信去通知刷新UI呢？答案是可以的，可采用EventBus，但我并不太推荐这种方式，原因有两个：a.组件通信是用消息队列实现的，不宜超大规模使用

b.EventBus不解绑会导致内存泄漏

综上所述，我们可以知道，在MVC架构中，Activity持有Model的引用，并且Activity实现了一个UI刷新的接口，Activity可以直接调用Model，Model用接口来通知Activity刷新。

看到这里，你有没有一个疑惑？

为何Activity可以持有Model的引用，而Model不能持有Activity的引用呢？如果Model可以持有Activity的引用，那岂不是就不需要接口了么？

这个问题的答案就是：Android本身就是一个框架，Activity作为页面的承载类，是不可以随意new一个Activity的，这就类似于后端框架中的控制器类，作为后端接收请求的入口，也是不可以随意new一个的。

那么既然不能new出一个Activity来，又要调用Activity刷新UI，那就要用接口或组件通信。

接口是Java语言提供的一个超级功能，接口可以无视一切限制，在类与类之间架起一道数据桥梁。

读到这里，我想你已经明白了MVC的本质，就是代码拆分。

###### 优点：代码耦合度降低，适合多人开发，比如一个人维护Activity，一个人维护Model，只要接口文件定义好，就可以互不干扰的开发，像不像前后端分离？

###### 缺点：增加了接口相关的代码量，代码用接口分离开来之后，IDE的函数跳转会变得麻烦。

#### （3）MVP

这个架构的出现，是为了将MVC中的Model层进行更精细的解离，分成了Model和Presenter两层，Model只负责网络请求，Presenter层负责逻辑处理。

在这个架构中，Activity直接持有Presenter的引用，Presenter直接持有Model的引用，Model层则不持有任何引用，只做网络请求。

那么问题来了，当Model层的网络请求结束之后，如何通知Presenter去进行逻辑操作呢？这一步采用的方式，可以是接口，也可以是组件通信（或观察者模式），在MVP中多是用Rxjava2（观察者模式），在Model的网络请求方法中，返回的是Observable，然后在Presenter中调用Model的网络请求，并对结果进行处理。

这里我们就明白了，Model层只做网络请求，不会持有其他的任何引用，而Presenter层持有Model的引用，并借用观察者模式处理Model层的返回结果。

第二个问题来了，当Presenter层的逻辑发生变动之后，由于Presenter无法持有Activity引用，如何通知UI刷新呢？

答案在MVC中已经探讨过，就是用接口或组件通信（或观察者模式）。

所以我们必须定义一个接口，用于定义UI的刷新方法，并且在Activity中实现这个接口。

这个接口就是MVP中的多见的命名为View的接口了。

如此以来我们就明白了，MVP中的VP其实就是老的MVC，M就是新的Model，只做网络请求，并用观察者模式返回处理结果。

很多朋友看到这里可能疑惑了？你这MVP和网上说的不一样呀，三个接口呢？契约接口呢？

其实这正是网上诸多教程迷惑人的地方，仔细看一看，在契约接口中的Model、View、Presenter三个接口，真正有数据传递作用的，是不是只有View呢？其他两个接口根本就没有起到数据传递作用。

那么其他两个接口是否没用呢？

答案是有用的，这两个接口虽然没有起到数据传递作用，但是它们起到了定义的作用，规范了Presenter和Model层的行为。

这就是接口的另一个作用，接口是一种简洁的说明，有了接口，就可以概览实现类的功能。

###### 优点：MVP对代码进行再一层拆分，适合更多的人一起开发，比如定义好三个接口之后，Activity、Presenter、Model层就可以各自开发，互不干扰。

###### 缺点：代码分散了，个人开发更麻烦。

#### （4）MVVM

这个架构本质上来讲，就是将MVP中的Presenter改为了ViewModel，从而实现了无需接口就可以解耦的目的。

在Presenter中，由于无法持有Activity的引用，只能用接口的方式去通知UI刷新，这样对个人开发者来说，就显得繁琐。

而使用ViewModel正是可以做到不用接口就去刷新UI，这是因为在ViewModel中持有了DataBinding的引用，可以直接进行UI操作，而这个DataBingding则是在Activity中调用DataBindingUtil.setContentView() 方法获取的。

同时有一个用法是，当layout中设置了data标签并布置了对应信息之后，DataBinding可以直接传入对象进行布局设置，但我强烈建议不要这样去做，因为这会破坏layout的独立性，而且无法debug。

DataBinding还有一些更加复杂的用法，比如用BaseObservable、ObservableField、ObservableCollection实现数据的单双向刷新，同样不太推荐去用。

为何不推荐去用呢？

让我们对比一下Android开发与Vue框架的不同，Android视图用的是xml文件，逻辑使用的是Java或Kotlin，严格遵循两者的分离，而且极其明显的是，在Android开发中，是以Activity之类的逻辑类为主，视图层为辅。

而在Vue框架之中，一个.vue文件可以分为三部分，html、css、js，其中css和js是独立的，而作为视图承载的html，则是占据了中心位置，点击事件、逻辑、样式等很多东西都是直接可以写在html中的。

这也是为何在Web开发中的MVVM思想，迁移到Android之后，会显得如此费解的原因。

由于Android的特性，在layout中写代码是极其不友好的行为。

###### 优点：代码分离的很好，适合个人开发，也适合多人合作开发。

###### 缺点：暂无

综上所述，在Android开发中，比较推荐的架构就是MVVM，但是不要去破坏layout的独立性，如果个人开发小项目的话，不分离也是一个很好的选择。

代码上的不分离并不意味结构上的混乱，Android项目要看起来清晰，一定要注意分包的合理性，也可以对整个项目进行组件化处理。

至于语言选择，比较推荐Kotlin，但是不要使用太多花里胡哨的语法，不然简洁的Kotlin反而会让人更难懂。
