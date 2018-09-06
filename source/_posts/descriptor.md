---
title: Python descriptor 描述符
date: 2018-07-13 11:44:18
tags:
    - Python
---

最近看Flask源码时发现很多不熟悉的语法，其中一个就是描述符，在config.py中出现，描述符的用处很多，是Python中很多特性的底层机制，如`properties`, `methods`, `static methods`, `class methods`和`super()`。

### 什么是描述符

描述符一般是一个有绑定动作的属性对象，这个属性的获取、赋值、删除操作和途径被描述符协议重写。对象属性的正常获取顺序是这样的，比如想要获取`a.x`,那么首先查找`a.__dict__['x']`,如果找不到则查看`type(a).__dict__['x']`,如果还没有则查看父类的`__dict__`。

Python中有很多协议，比如迭代对象的迭代器协议，上下文管理协议等，都是靠重写类中以`__`开头和结尾的魔法方法来实现的。描述符协议也不例外，只要实现了`__get__(self, instance, owner)`、`__set__(self, instance, value)`、`__delete__(self, instance)`中任意一个或全部的方法，这个类就变成了一个描述符。如果只定义了`__get__`,则这是一个`non-data descriptor`，定义了`__get__`和`__set__`两个方法的是`data descriptor`，这里的区别，后面会提到。实现这些方法后，对属性进行操作时就不走正常途径，而是调用这几个魔法方法。需要注意的是，描述符必须是一个新式类。

<!--more-->

### 为什么需要描述符

写过的Java的应该有一些印象，类里的属性一般是`private`的，如果想要拿到这个属性，一般是通过一个`public`的`get_xxx`方法来获得属性，重新赋值时也是一个道理。但是这样虽然隐藏了属性，但是后续写代码时都得用`object.get_xxx()`来获取值，而不是`object.attr`,显然第二种方式更简单，更美观，所以Python这种简洁的语言就提供了这样的更简洁的实现方式---描述符协议。

还有一种情况，假设有一个`Person`类，它有一个`age`属性，那么在对年龄赋值时是有一些限制的，比如必须是整数，必须大于0。所以应该在赋值时进行检查，这么一看好像赋值时又需要通过方法`xiaoming.age=cls.examine_age(1000)`,又不美观了，而描述符协议可以在背地里帮我们做这种检查，而我们还是可以使用`xiaoming.age=1000`这个更简洁的语句。

这里有个我之前一直困惑的地方提一下，可能你们觉得不难，但是确实干扰了我很久。那就是这三个魔法方法定义在什么地方，还是回到上面那个例子，好像只有`Person`是一个类，所以我之前一直觉得应该定义在`Person`类中，但其实不是。魔法方法应该定义在一个`Age`类中，然后`age`属性是一个`Age`对象实例。

```python
class Age(object):
    def __init__(self, age):
        self.age = age

    def __get__(self, instance, owner):
        print('instance={}, owner={}'.format(instance, owner))
        return self.age

    def __set__(self, instance, value):
        print('instance={}, value={}'.format(instance, value))
        if value < 0:
            raise AttributeError('age should > 0')
        self.age = value


class Person(object):
    age = Age(100)


xiaoming = Person()
xiaoming.age = 10
print(xiaoming)
print(xiaoming.age)

# output::::::::
# >>> instance=<__main__.Person object at 0x107a24310>, value=10
# >>> <__main__.Person object at 0x107a24310>
# >>> instance=<__main__.Person object at 0x107a24310>, owner=<class '__main__.Person'>
# >>> 10
```
方法中的`instance`属性返回的是获取属性的那个对象，在这里就是`xiaoming`，`owner`是获取属性的对象的类，在这里就是`Person`。

### 描述符的调用机制

上面提到了非描述符属性的获取途径，定义了描述符协议后，`obj.b`的操作将调用`b.__get__(obj)`这个方法来获取属性。描述符的调用机制根据调用对象是**对象**还是**类**有一些区别。

描述符是通过`type.__getattribute__()`方法被调用，这也是为什么描述符必须是在新式类中的原因，继承自`object`的类被称为新式类，否则没有这个方法，则无法调用描述符的方法。

对于对象来说，`object.__getattribute__()`会将`b.x` 转换为 `type(b).__dict__['x'].__get__(b, type(b))`。这个转换通过下面这样的一个优先链：`data descriptors`大于实例变量，实例变量大于 `non-data descriptors`，如果存在`__getattr__()`，则`__getattr__()`优先级最低。完整的C实现在[`PyObject_GenericGetAttr()`](https://docs.python.org/2/c-api/object.html#c.PyObject_GenericGetAttr "PyObject_GenericGetAttr") in [Objects/object.c](https://github.com/python/cpython/tree/2.7/Objects/object.c).

对于类来说，`object.__getattribute__()`会将 `B.x` 转换为`B.__dict__['x'].__get__(None, B)`。Python实现如下：
```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
        return v.__get__(None, self)
    return v
```
### 描述符实例

上面提到了一个最简单的描述符实例，就是对属性进行取值或者赋值时进行额外的操作，同时保持代码的简洁。描述符在Python语言中本来也有很多的应用，但是能力不够，不能很好的理解其中的奥妙，就不误导大家了。主要是`Property`,`Function and method`和`static method and class method`这几个方面，给出链接，有兴趣的可以钻研一下。

### 参考链接

[Python Descriptors, Part 1 of 2](http://martyalchin.com/2007/nov/23/python-descriptors-part-1-of-2/)

[Descriptor HowTo Guide](https://docs.python.org/2/howto/descriptor.html)


