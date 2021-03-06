---
layout:      post
title:       单例模式在 Python 中的应用
subtitle:    Tornado IOLoop 中 double check 形式的单例模式解析，另：4.7……
date:        2015-04-07 14:00:11
author:      Damnever
keywords:    Damnever,设计模式,单例模式,Python,Tornado, double check
description: 单例模式在 Python 中的应用，tornado.IOLoop中double check的单例模式
categories:
  - 设计模式
  - Python
tags:
  - 单例模式
  - Tornado

---

单例模式
<span class="caption text-muted">保证一个类仅有一个实例，并提供一个访问它的全局访问点。</span>


---

来看看`tornado.IOLoop`中的单例模式：
{% highlight python %}
class IOLoop(object):

    @staticmethod
    def instance():
        """Returns a global `IOLoop` instance.

        Most applications have a single, global `IOLoop` running on the
        main thread.  Use this method to get this instance from
        another thread.  To get the current thread's `IOLoop`, use `current()`.
        """
        if not hasattr(IOLoop, "_instance"):
            with IOLoop._instance_lock:
                if not hasattr(IOLoop, "_instance"):
                    # New instance after double check
                    IOLoop._instance = IOLoop()
        return IOLoop._instance
{% endhighlight %}

为什么这里要`double check`？

---

### 简单的单例模式

先来看看代码：
{% highlight python %}
class Singleton(object):

    @staticmathod
    def instance():
        if not hasattr(Singleton, '_instance'):
            Singleton._instance = Singleton()
        return Singleton._instance
{% endhighlight %}
在 Python 里，可以在真正的构造函数`__new__`里做文章：
{% highlight python %}
class Singleton(object):

    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
        return cls._instance
{% endhighlight %}
这种情况看似还不错，但是不能保证在多线程的环境下仍然好用，看图：
![singleton.png]({{ site.baseurl }}/img/post/2015-04-07.png)
出现了多线程之后，这明显就是行不通的。

---

### 上锁使线程同步

上锁后的代码：
{% highlight python %}
import threading

class Singleton(object):

    _instance_lock = threading.Lock()
    
    @staticmethod
    def instance():
        with Singleton._instance_lock:
            if not hasattr(Singleton, '_instance'):
                Singleton._instance = Singleton()
        return Singleton._instance
{% endhighlight %}
这里确实是解决了多线程的情况，但是我们只有实例化的时候需要上锁，其它时候`Singleton._instance`已经存在了，不需要锁了，但是这时候其它要获得`Singleton`实例的线程还是必须等待，锁的存在明显降低了效率，有性能损耗。

---

### 全局变量

在 Java/C++ 这些语言里还可以利用全局变量的方式解决上面那种加锁（同步）带来的问题：
{% highlight java %}
class Singleton {

    private static Singleton instance = new Singleton();
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        return instance;
    }
    
}
{% endhighlight %}
在 Python 里就是这样了：
{% highlight python %}
class Singleton(object):

    @staticmethod
    def instance():
        return _g_singleton

_g_singleton = Singleton()

# def get_instance():
#     return _g_singleton
{% endhighlight %}
但是如果这个类所占的资源较多的话，还没有用这个实例就已经存在了，是非常不划算的，Python 代码也略显丑陋……

---

### 总结

所以出现了像`tornado.IOLoop.instance()`那样的`double check`的单例模式了。在多线程的情况下，既没有同步（加锁）带来的性能下降，也没有全局变量直接实例化带来的资源浪费。

---

### 6-25 补充（使用 decorator）：

https://wiki.python.org/moin/PythonDecoratorLibrary#Singleton

{% highlight python %}
import functools

def singleton(cls):
    ''' Use class as singleton. '''

    cls.__new_original__ = cls.__new__

    @functools.wraps(cls.__new__)
    def singleton_new(cls, *args, **kw):
        it =  cls.__dict__.get('__it__')
        if it is not None:
            return it

        cls.__it__ = it = cls.__new_original__(cls, *args, **kw)
        it.__init_original__(*args, **kw)
        return it

    cls.__new__ = singleton_new
    cls.__init_original__ = cls.__init__
    cls.__init__ = object.__init__

    return cls

#
# Sample use:
#

@singleton
class Foo:
    def __new__(cls):
        cls.x = 10
        return object.__new__(cls)

    def __init__(self):
        assert self.x == 10
        self.x = 15

assert Foo().x == 15
Foo().x = 20
assert Foo().x == 20
{% endhighlight %}

https://wiki.python.org/moin/PythonDecoratorLibrary#The_Sublime_Singleton

{% highlight python %}
def singleton(cls):
    instance = cls()
    instance.__call__ = lambda: instance
    return instance

#
# Sample use
#

@singleton
class Highlander:
    x = 100
    # Of course you can have any attributes or methods you like.

Highlander() is Highlander() is Highlander #=> True
id(Highlander()) == id(Highlander) #=> True
Highlander().x == Highlander.x == 100 #=> True
Highlander.x = 50
Highlander().x == Highlander.x == 50 #=> True
{% endhighlight %}

### 8-27 更：

我认为政治正确的单例模式，见：[singleton.py](https://github.com/Damnever/Note/blob/master/singleton.py)

***
