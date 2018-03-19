---
layout: post
title: python 多重继承之拓扑排序
categories: [python]
description: python 多重继承之拓扑排序
keywords: python,topological-sorting
---

> 最近在学python，学到class 多重继承，降到了c3算法，这里记录一下


<!--more-->

### 一、什么是拓扑排序

在图论中，**拓扑排序(Topological Sorting)** 是一个 **有向无环图(DAG,Directed Acyclic Graph)** 的所有顶点的线性序列。且该序列必须满足下面两个条件：

* 每个顶点出现且只出现一次。
* 若存在一条从顶点A到顶点B的路径，那么在序列中顶点A出现在顶点B的前面。

例如，下面这个图：

![](/images/posts/original.png)

它是一个DAG图，那么如何写出它的拓扑顺序呢？这里说一种比较常用的方法：

* 从DAG途中选择一个没有前驱(即入度为0)的顶点并输出
* 从图中删除该顶点和所有以它为起点的有向边。
* 重复1和2直到当前DAG图为空或当前途中不存在无前驱的顶点为止。后一种情况说明有向图中必然存在环。

![](/images/posts/topological-sorting.png)

于是，得到拓扑排序后的结果是{1,2,4,3,5}

下面，我们看看拓扑排序在python多重继承中的例子


### 二、python 多重继承

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
class A(object):
    def foo(self):
        print('A foo')
    def bar(self):
        print('A bar')

class B(object):
    def foo(self):
        print('B foo')
    def bar(self):
        print('B bar')

class C1(A,B):
    pass

class C2(A,B):
    def bar(self):
        print('C2-bar')

class D(C1,C2):
    pass

if __name__ == '__main__':
    print(D.__mro__)
    d=D()
    d.foo()
    d.bar()
```

> 首先，我们根据上面的继承关系构成一张图，如下

![](/images/posts/python-inherit.png)

* 找到入度为0的点，只有一个D，把D拿出来，把D相关的边剪掉
* 现在有两个入度为0的点(C1,C2)，取最左原则，拿C1，剪掉C1相关的边，这时候的排序是{D,C1}
* 现在我们看，入度为0的点(C2),拿C2,剪掉C2相关的边，这时候排序是{D,C1,C2}
* 接着看，入度为0的点(A,B),取最左原则，拿A，剪掉A相关的边，这时候的排序是{D,C1,C2,A}
* 继续，入度哦为0的点只有B，拿B，剪掉B相关的边，最后只剩下object
* 所以最后的排序是{D,C1,C2,A,B,object}

我们执行上面的代码，发现`print(D.__mro__)`的结果也正是这样，而这也就是多重继承所使用的C3算法啦

> 为了进一步熟悉这个拓扑排序的方法，我们再来一张图，试试看排序结果是怎样的，它继承的内容是否如你所想



```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
class A(object):
    def foo(self):
        print('A foo')
    def bar(self):
        print('A bar')

class B(object):
    def foo(self):
        print('B foo')
    def bar(self):
        print('B bar')

class C1(A):
    pass

class C2(B):
    def bar(self):
        print('C2-bar')

class D(C1,C2):
    pass

if __name__ == '__main__':
    print(D.__mro__)
    d=D()
    d.foo()
    d.bar()
```

> 还是先根据继承关系构一个继承图

![](/images/posts/python-inherit2.png)

* 找到入度为0的顶点，只有一个D，拿D，剪掉D相关的边
* 得到两个入度为0的顶点(C1,C2),根据最左原则，拿C1，剪掉C1相关的边，这时候序列为{D,C1}
* 接着看，入度为0的顶点有两个(A,C1),根据最左原则，拿A，剪掉A相关的边，这时候序列为{D,C1,A}
* 接着看，入度为0的顶点为C2,拿C2，剪掉C2相关的边，这时候序列为{D,C1,A,C2}
* 继续，入度为0的顶点为B，拿B，剪掉B相关的边，最后还有一个object
* 所以最后的序列为{D,C1,A,C2,B,object}

最后，我们执行上面的代码，发现`print(D.__mro__)`的结果正如上面所计算的结果


最后的最后，python继承顺序遵循C3算法，只要在一个地方找到了所需的内容，就不再继续查找
