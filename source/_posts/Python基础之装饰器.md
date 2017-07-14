---
title: Python基础之装饰器
date: 2016-12-03 10:09:21
tags:
- Python
categories:
- Python
---

# 1. 装饰器简介

　　首先，`@func_decorator` 长成这个样子的东西，在 Python 中被称为装饰器（Decorator）。第一眼看到它，我还以为又遇到了 Java 中的注解（Annotation），有一种很熟悉的感觉。仔细了解下来，发现装饰器与注解有点类似又有点不同。

　　装饰器是用来修饰方法、函数或者类的，其作用是在原先的基础上完成一些额外的功能，利用装饰器来达到在不影响原先代码的情况下增加功能的需求。相比较而言，注解在 Java 中是可以用来修饰方法、函数、类和**变量**的，且其作用仅仅类似于注释的作用，当注解和 Java 中的反射机制结合在一起才完成了更多的功能。

　　下面还是如何使用装饰器吧。

# 2. 装饰器的分类

　　装饰器可以用于修饰函数和类，并且装饰器还分带参数和不带参数两种，所以我们分别来讲这几种不同的装饰器该如何编写，使用。

## 2.1 修饰函数的装饰器

### 2.1.1 不带参数的装饰器

　　假定我们之前有这么一个函数：

``` python
def func_test():
    print 'This is func_test() running....'
```

　　现在，有这么一个需求过来：要求在这个函数运行开始前和运行结束后记录一下。之前的我肯定二话不说，操起键盘，上来就改成：

``` python
def func_test():
    print '%s() start.' %(func_test.__name__)
    print 'This is func_test() running....'
    print '%s() finish.' %(func_test.__name__)
```

　　但是如果我用装饰器我可以怎么写呢。

　　首先，使用装饰器之前需要先定义装饰器，Python 定义装饰器和定义函数式一样的：

``` python
def func_decorator(func):
    def wrapper(*args,**kw):
        print '%s() start.' %(func.__name__)
        func(*args,**kw)
        print '%s() finish.' %(func.__name__)
    return wrapper
```

　　然后怎么使用这和装饰器呢，很简单：

``` python
@func_decorator
def func_test():
    print 'This is func_test() running....'

if __name__ == '__main__':
	func_test()
```

　　就是在原先的基础上增加了一行代码，做到的效果和我原先更改两行代码的效果是一样的。

![](http://oc4wmeyj8.bkt.clouddn.com/func_decorator_without_args.PNG)

　　在回过头来看一下 `func_decorator` 的代码，其实不难理解，因为 Python 一切都是对象，所以装饰器，把函数 `func_test` 当成一个参数传入一个新的函数中，在新的函数 `wrapper` 中增加我们所需要的功能，最后将这个经过**装饰**的函数返回，使其替代原先的函数。


### 2.1.2 带参数的装饰器

　　有了不带参数的装饰器的基础，带参数的装饰器就比较容易理解了。我们看到装饰器的定义和函数的定义没有什么区别，我们可以向函数传递参数，那么我们也可以向装饰器传递参数。假定我们现在还是原来那个函数：

``` python
def func_test():
    print 'This is func_test() running....'
```

　　新的需求是：输出函数使用对象。

　　我们可以定义这样一个装饰器：

``` python
def func_decorator_with_args(user):
    def decorator(func):
        def wrapper(*args,**kw):
            print 'The user of func is %s' %(user)
            return func(*args,**kw)
        return wrapper
    return decorator
```

　　然后使用这个装饰器：

``` python
@func_decorator_with_args('sg')
def func_test():
    print 'This is func_test() running....'

if __name__ == '__main__':
	func_test()
```

　　结果如下：

![](http://oc4wmeyj8.bkt.clouddn.com/func_decorator_with_args.PNG)

　　给定义带参数的装饰器的时候也可以向函数一样指定默认值。

## 2.2 修饰类的装饰器

　　看过用来修饰函数的装饰器，我们再来看看修饰类的装饰器。修饰函数的装饰器是将原函数包装成一个新的函数并且返回，修饰类的装饰器也是一样的道理，是将原先的类包装成一个新的类并且返回。

　　假设我们有这么一个类：

``` python
class People(object):
    def __init__(self,age,name):
        self.age = age
        self.name = name
    def show(self):
        print 'Age:%d Name:%s' %(self.age,self.name)
```

　　需求是在每次调用 `show()` 函数的时候，记录这个函数一共被调用了几次。

　　我们定义了这么一个装饰器，并使用这个装饰器：

``` python
def class_decorator(a):
    class NewClass(object):
        def __init__(self,age,name):
            self.show_count = 0
            self.wrapped = a(age,name)
        def show(self):
            self.show_count += 1
            print 'Call show() %d ' %(self.show_count)
            self.wrapped.show()
    return NewClass

@class_decorator
class People(object):
    def __init__(self,age,name):
        self.age = age
        self.name = name
    def show(self):
        print 'Age:%d Name:%s' %(self.age,self.name)

if __name__ == '__main__':
    p = People(22,'sg')
    p.show()
    p.show()
```

　　可以得到结果：

![](http://oc4wmeyj8.bkt.clouddn.com/class_decorator.PNG)

　　可以看到 `class_decorator` 这装饰器是创建了一个 `NewClass` 来替代被装饰的类，比较重要的点就是，调用被装饰类的 `init()` 方法来获得被装饰类的一个对象，并用这个对象完成对原先方法的调用。

# 3. 装饰器的用途

　　装饰器的作用和设计模式中的装饰模式吻合，在尽量保持原代码不动的情况下增加程序的功能。但是从我目前的感觉来说，Pyhton 的装饰器用于修饰函数还是可以的，但是用于修饰类的装饰器，我觉得并不方便，因为装饰器的代码完全需要依赖于原先的类的定义，我定义一个装饰器，我还不如重新定义一个类来继承原类。

　　总结来说，我知道装饰器该怎么用了，但是用在哪，怎么**用好**装饰器，我任然需要学习。