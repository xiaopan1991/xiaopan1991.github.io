---
layout:     post
title:      "Unity游戏开发---Update的10000次调用"
subtitle:   " \"N call 1 VS 1 call N. \""
date:       2018-03-29
author:     "xiaopan"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    - 实用技能
---

> “ 继续努力. ”

这篇文章我们来总结一下一个经常被问到的话题，关于Unity中Update的调用。小标题中“N call 1 VS 1 call N”的意思是：N个带有Update的脚本各调用一次 VS 在一个Update里面调用N次自己写的用于更新的UpdateMe方法。

Unity Blog中有一篇详细讲述这个问题的文章：[10000 Update() calls](https://blogs.unity3d.com/cn/2015/12/23/1k-update-calls/)  
网上有网友进行过翻译，我参考的是这个帖子：[Unity3d 10000 Update() calls 性能优化](https://blog.csdn.net/huutu/article/details/50522585)

这两篇文章看完就能对这个问题有很清晰的了解，知道以后项目该用什么样的方案。

---

如果你不想费时间看文章，可以参考一下我的总结  

#### 重点1：Update方法如何被调用
熟悉MonoBehaviour的小伙伴应该知道，在MonoBehaviour类中有很多的所谓的魔术方法（Magic Methods），就是指Awake()，OnEnable()，Start()，FixedUpdate()，Update()，LateUpdate() ， OnDisable()等等方法。  

Unity在每次需要调用魔法方法的时候并不是使用System.Reflection（反射）来定位它们的。  

取而代之的是，在首次访问某个继承自MonoBehaviour类时，脚本运行时Mono或IL2CPP会检查其脚本中是否存在魔术方法，以及相关信息是否已经被缓存。如果发现有，则将方法注册进循环列表中，例如，如果一个脚本中定义了Update方法，它将被加入到一个需要每帧都跟新的脚本列表中。  

在游戏过程中，Unity只是简单的循环迭代所有的列表，然后执行其中的方法。这也就是为什么你的魔术方法是public或是private并不重要的原因。  

#### 重点2：Update方法调用的成本消耗
根据Unity Blog文中的实验数据表明，每次执行Update方法都是有成本消耗的，具体的消耗在哪？  

Unity为了防止游戏出错或崩溃，做了很多了不起的事情：  
> 1.判断脚本挂载的GameObject是否已经被激活？  
> 2.它是否在Update循环中被销毁？  
> 3.脚本中是否存在Update方法？  
> 4.怎么处理这个Update循环中创建的MonoBehaviour？  

等等一系列问题，然而我们自己写的Manager脚本中并没有处理这些判断，仅仅是循环迭代了一堆对象，调用了他们的UpdateMe之类的方法而已。

---

![Unity函数生命周期图](https://xiaopan1991.github.io/img/UpdateCall/UpdateCall.jpg)