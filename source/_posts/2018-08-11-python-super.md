---
title: python 多态和 super 用法
date: 2018-08-10 09:19:44
update: 2018-08-10 09:19:44
categories: Python
tags: [python, 继承, super, 多态]
---

python 中的多态实现非常简单，只要是在子类中实现和父类同名的方法，便能实现多态，如果想在子类中调用父类的方法，有多种方法，但是当涉及菱形继承等问题是，super 就成为了比较好的解决方案。

<!--more-->

### 普通继承

对于比较简单的继承关系，通常在子类中有两种方法来执行父类的方法，示例如下。

基类：

```python
class Base(object):
    def __init__(self):
        print("init Base")
```

示例 1：

```python
class A(Base):
    def __init__(self):
        # 通过父类显式执行
        Base.__init__(self)
        print("init A")

a = A()
```

输出：

```python
init Base
init A
```

示例 2：

```python
class B(Base):
    def __init__(self):
        # 调用 super
        super(B, self).__init__()
        print("init B")

b = B()
```

输出: 

```python
init Base
init B
```

可以看到，两种方法都可以调用父类的方法对父类进行初始化。需要注意的是，**两种方法都要传入 self**，但是在**子类中调用父类的 super 中传入的 self是子类对象。**

### 菱形继承

当有多重继承，特别是菱形继承时，这两种方法就有区别了，示例如下。

示例 1：

```python
class Base(object):
    def __init__(self):
        print("init Base")
        
class A(Base):
    def __init__(self):
        Base.__init__(self)
        print("init A")

class B(Base):
    def __init__(self):
        Base.__init__(self)
        print("init B")
        
class C(A, B):
    def __init__(self):
        A.__init__(self)
        B.__init__(self)
        print("init C")
c = C()
```

输出：

```python
init Base
init A
init Base
init B
init C
```

可以看到，Base 被 init 了两次，至于其缺陷，在 C++ 中就已经讨论过了，反正就是不符合我们的预期，不想这种实现。C++ 中通过虚继承解决菱形继承问题，在 python 中可以使用 super 规避这种缺陷。

示例2：
```python
class Base(object):
    def __init__(self):
        print("init Base")
        
class A(Base):
    def __init__(self):
        super(A, self).__init__()
        print("init A")

class B(Base):
    def __init__(self):
        super(B, self).__init__()
        print("init B")
        
class C(A, B):
    def __init__(self):
        super(C, self).__init__()
        print("init C")
c = C()
```

输出
```python
init Base
init B
init A
init C
```

运行这个新版本后，你会发现每个 `__init__()` 方法只会被调用一次了。
 
为了弄清它的原理，我们需要花点时间解释下 python 是如何实现继承的。对于你定义的每一个类，python 会计算出一个所谓的方法解析顺序（MRO）列表。 这个 MRO 列表就是一个简单的所有基类的线性顺序表。我们可以看一下 C 的 MRO 表。

```python
>>> C.__mro__
(__main__.C, __main__.A, __main__.B, __main__.Base, object)
```

为了实现继承，python 会在 MRO 列表上**从左到右**开始查找基类，直到找到第一个匹配这个属性的类为止。

而这个 MRO 列表的构造是通过一个 C3 线性化算法来实现的。 我们不去深究这个算法的数学原理，它实际上就是合并所有父类的 MRO 列表并遵循如下三条准则：

* 子类会先于父类被检查
* 多个父类会根据它们在列表中的顺序被检查
* 如果对下一个类存在两个合法的选择，选择第一个父类

必须牢记：**MRO 列表中的类顺序会让你定义的任意类层级关系变得有意义。**

**当使用 `super()` 函数时，python 会在 MRO 列表上继续搜索下一个类（这是一种嵌套实现）。** 只要每个重定义的方法统一使用 `super()` 并只调用它一次， 那么控制流最终会遍历完整个 MRO 列表，每个方法也只会被调用一次。 这也是为什么在第二个例子中你不会调用两次 `Base.__init__()` 的原因。换句话说，`super` 调用了次且仅有一次所有的父类。

由于 super 递归调用的会继续搜索的特性，可能会出现一些意向不到的效果，比如下面这个例子：

```python
class A(object):
    def spam(self):
        print('A.spam')
        super(A, self).spam()

class B(object):
    def spam(self):
        print('B.spam')

class C(A, B):
    pass

c = C()
c.spam()

C.__mro__
```

输出
```python
A.spam
B.spam
(__main__.C, __main__.A, __main__.B, object)
```

为啥 `c.spam()` 会同时调用 A 和 B 的 `spam()`？其实看到 MRO 顺序就明白了：

* 首先在 c 的类 C 中查找 spam 方法，没有找到就查找 A 中的 spam 方法。
* 调用 A 中的 spam 方法，然后遇到 A 的 super 调用，继续在 MRO 顺序表中查找 spam 方法。注意，这里本来调用了 A 的 spam 就应该返回的，但是 super 的存在，导致了继续递归。
* 遇到 B 的 spam 方法，调用，结束。

### super 的使用

对于 python2 和 python3，super 的用法有一些区别：

原因：

* python2 没有默认继承 object
* python3 默认全部继承 object 类，都是新式类

用法区别：

* python2： super(开始类名，self).函数名()
* python3：super().函数名()

#### 参考资料

* [python3官方文档](http://python3-cookbook.readthedocs.io/zh_CN/latest/c08/p07_calling_method_on_parent_class.html)
* https://mozillazg.com/2016/12/python-super-is-not-as-simple-as-you-thought.html

