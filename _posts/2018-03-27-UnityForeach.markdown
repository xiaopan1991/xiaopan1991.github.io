---
layout:     post
title:      "Unity游戏开发---聊聊在Unity编程中对foreach的偏见"
subtitle:   " \"到底能不能使用？会不会引起GC\""
date:       2018-03-27
author:     "xiaopan"
header-img: "img/timg1.jpg"
catalog: true
tags:
    - 实用技能
---

> “ 无则加勉, 有则改之. ”


最近一直在忙着面试，很多面试题都非常基础（大多都可以在网上找到原题），昨天我在做一份面试题的时候突然想到了以前网上很多帖子都在说foreach的使用会频繁引起GC操作，在很多介绍unity项目优化的文章中也有提到使用for代替foreach，甚至是上家公司某位很资深的程序也跟我说过不要使用foreach。但是具体是哪天我记不清楚了，我貌似在一本书或是一个帖子上看到，上面说foreach的使用会产生GC操作的bug已经解决了。所以今天就带着这个疑问，经过我的测试，写下了这篇文章。我测试的过程参考的是[这个帖子](https://blog.csdn.net/andrewfan/article/details/59147132)（下文中简称：引用帖子）中介绍的方式。希望我的测试经过和结果能给小伙伴们带去帮助。 

> ##### (我使用的Unity版本为Unity2017.3.0f3)

### GC ALLOC产生情况，测试结果对比：  

#### > 引用帖子中的结果：

方法\类型 | int[] (Array) | List< int > | ArrayList
---|---|---|---
foreach | 不产生|产生 |产生
GetEnumerator | 产生|不产生 |产生

#### > 我的测试结果：

方法\类型 | int[] (Array) | List< int > | ArrayList
---|---|---|---
foreach | 不产生|不产生 |产生
GetEnumerator | 产生|不产生 |产生

#### > 结论：从测试结果上看，只有在对List<int>使用foreach遍历时，结果是不同的  
#### > 猜想：这个Bug应该是解决了

那下面就带着我的猜想来看看具体的测试过程  

---

#### Code：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ForeachTest : MonoBehaviour
{

    private int[] m_intArray;
    private List<int> m_intList;
    private ArrayList m_arrayList;
	
	void Start () {
		
        m_intArray = new int[2];
        m_intList = new List<int>();
        m_arrayList = new ArrayList();

	    for (int i = 0; i < m_intArray.Length; i++)
	    {
	        m_intArray[i] = i;
            m_intList.Add(i);
	        m_arrayList.Add(i);
	    }
    }
	
	
	void Update ()
	{
	    TestIntArrayForeach();
	}

    private void TestIntArrayForeach()
    {
        for (int i = 0; i < 1000; i++)
        {
            foreach (var i1 in m_intArray)
            {
                
            }
        }
    }
    private void TestIntArrayGetEnumerator()
    {
        for (int i = 0; i < 1000; i++)
        {
            var iNum = m_intArray.GetEnumerator();
            while (iNum.MoveNext())
            {
            }
        }
    }
    private void TestIntListForeach()
    {
        for (int i = 0; i < 1000; i++)
        {
            foreach (var i1 in m_intList)
            {

            }
        }
    }
    private void TestIntListGetEnumerator()
    {
        for (int i = 0; i < 1000; i++)
        {
            var iNum = m_intList.GetEnumerator();
            while (iNum.MoveNext())
            {
            }
        }
    }
    private void TestArrayListForeach()
    {
        for (int i = 0; i < 1000; i++)
        {
            foreach (var i1 in m_arrayList)
            {

            }
        }
    }
    private void TestArrayListGetEnumerator()
    {
        for (int i = 0; i < 1000; i++)
        {
            var iNum = m_arrayList.GetEnumerator();
            while (iNum.MoveNext())
            {
            }
        }
    }
}

```
---

#### 测试：int[] (Array)类型 / foreach方法

```
private void TestIntArrayForeach()
{
    for (int i = 0; i < 1000; i++)
    {
        foreach (var i1 in m_intArray)
        {
            
        }
    }
}
```
##### Unity Profiler：

![image](https://xiaopan1991.github.io/img/Foreach/TestIntArrayForeach_unity.png)

##### 通过ILSpy工具查看代码程序集内部的代码  
##### （将工程目录下的：Library\ScriptAssemblies\Assembly-CSharp.dll文件拖入ILSpy工具中，然后找到unity的安装目录下的Unity\Editor\Data\Mono\lib\mono\2.0\mscorlib.dll文件，也拖入ILSpy工具）  
  

![image](https://xiaopan1991.github.io/img/Foreach/TestIntArrayForeach_IL.png)  

在【引用帖子】中作者提到：
>  虽然代码比较长，不熟悉IL的同学也不需要完整理解它们，我们只要知道少数几个重要的IL字段就可以：

> newobj 指令，如果出现newobj 指令，如果跟随值类型，说明它在栈上新建对象，它不会产生GCAlloc；如果后面参数跟随对象类型，则说明它在堆上新建对象，会产生GC Alloc  

> callvirt 指令，它表示函数调用，后方会跟随某个类的某个函数，被调用的函数中也可能会产生GC Alloc  

> box指令，装箱，将值类型封装成指定的对象类型，流程是，弹出计算堆栈上的值类型参数，并使用新建立的一个引用类型对象进行并包装，将包装结果返回计算堆栈。本过程产生GC Alloc。


在上面常常的代码中，没有出现这三个指令，那么也就是说，这方法没有产生新的内存，符合之前的UnityProfiler中的结果。

#### 测试：int[] (Array)类型 / GetEnumerator方法

```
private void TestIntArrayGetEnumerator()
{
    for (int i = 0; i < 1000; i++)
    {
        var iNum = m_intArray.GetEnumerator();
        while (iNum.MoveNext())
        {
        }
    }
}
```
##### Unity Profiler：

![image](https://xiaopan1991.github.io/img/Foreach/TestIntArrayGetEnumerator_unity.png)
这里产生GC Alloc，每帧产生31.3KB的新内存。
##### 通过ILSpy工具查看代码程序集内部的代码 
![image](https://xiaopan1991.github.io/img/Foreach/TestIntArrayGetEnumerator_IL.png)
代码里出现了【callvirt 指令】，查看这个函数调用，代码如下：
![image](https://xiaopan1991.github.io/img/Foreach/TestIntArrayGetEnumerator_IL2.png)
出现了【newobj 指令】，且跟随对象类型System.Array/SimpleEnumerator，新的GC Alloc由此产生。

---

#### 测试：List<int>类型 / foreach方法

```
private void TestIntListForeach()
{
    for (int i = 0; i < 1000; i++)
    {
        foreach (var i1 in m_intList)
        {

        }
    }
}
```
##### Unity Profiler：

![image](https://xiaopan1991.github.io/img/Foreach/TestIntListForeach_unity.png)
这方法没有产生新的内存，符合之前的UnityProfiler中的结果。
##### 通过ILSpy工具查看代码程序集内部的代码:
![image](https://xiaopan1991.github.io/img/Foreach/TestIntListForeach_IL1.png)
代码里出现了【callvirt 指令】，查看这个函数调用，代码如下：
![image](https://xiaopan1991.github.io/img/Foreach/TestIntListForeach_IL2.png)

这里同样也出现了【newobj 指令】，但是应用于值类型，所以，这这函数调用指令也不会产生GCAlloc，符合之前的UnityProfiler中的结果。

# 就在这，就在这，就在这（重要的事情说三遍）  
在【引用帖子】中，这个地方有后文，文中是这样说的：
> ![image](https://xiaopan1991.github.io/img/Foreach/yinyong1.png)

## 但是在我通过ILSpy工具查看代码程序集内部的代码时，在finally代码块中并没有发现原文作者说的这个【box 指令】  
## 所以我认为这正是对List变量进行foreach循环后不会产生GC操作的原因。
---

#### 测试：List<int>类型 / GetEnumerator方法

```
private void TestIntListGetEnumerator()
{
    for (int i = 0; i < 1000; i++)
    {
        var iNum = m_intList.GetEnumerator();
        while (iNum.MoveNext())
        {
        }
    }
}
```
##### Unity Profiler：
![image](https://xiaopan1991.github.io/img/Foreach/TestIntListGetEnumerator_unity.png)
##### 通过ILSpy工具查看代码程序集内部的代码:
![image](https://xiaopan1991.github.io/img/Foreach/TestIntListGetEnumerator_IL1.png)
代码里出现了【callvirt 指令】，查看这个函数调用，代码如下：
![image](https://xiaopan1991.github.io/img/Foreach/TestIntListGetEnumerator_IL2.png)
这里同样也出现了【newobj 指令】，但是应用于值类型，所以，这这函数调用指令也不会产生GCAlloc，符合之前的UnityProfiler中的结果。

---

#### 测试：ArrayList类型 / foreach方法
```
private void TestArrayListForeach()
{
    for (int i = 0; i < 1000; i++)
    {
        foreach (var i1 in m_arrayList)
        {

        }
    }
}
```
##### Unity Profiler：
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListForeach_unity.png)
##### 通过ILSpy工具查看代码程序集内部的代码:
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListForeach_IL1.png)
代码里出现了【callvirt 指令】，查看这个函数调用，代码如下：
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListForeach_IL2.png)
这里出现了【newobj 指令】，且跟随对象类型System.ArrayList/SimpleEnumerator，GC操作由此产生。

#### 测试：ArrayList类型 / GetEnumerator方法
```
private void TestArrayListGetEnumerator()
{
    for (int i = 0; i < 1000; i++)
    {
        var iNum = m_arrayList.GetEnumerator();
        while (iNum.MoveNext())
        {
        }
    }
}
```
##### Unity Profiler：
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListGetEnumerator_unity.png)
##### 通过ILSpy工具查看代码程序集内部的代码:
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListGetEnumerator_IL1.png)
代码里出现了【callvirt 指令】，查看这个函数调用，代码如下：
![image](https://xiaopan1991.github.io/img/Foreach/TestArrayListGetEnumerator_IL2.png)
这里出现了【newobj 指令】，且跟随对象类型System.ArrayList/SimpleEnumerator，GC操作由此产生。
---

# 最终结论：
### 代码中foreach是可以使用的，但是得看用foreach遍历的是什么类型的变量，在新的Unity版本中（起码在Unity2017中）已经修复了这个问题。由此可见一个人的知识存储不仅是要慢慢不断增长，也是需要在增长的过程中不断更新原有的知识。科技在飞速进步，人也需要不断自我提高，资深并不代表你知道的都是对的。如果本文中有写的不正确的地方请联系我！（微信，QQ都在About页签中，欢迎打扰）
