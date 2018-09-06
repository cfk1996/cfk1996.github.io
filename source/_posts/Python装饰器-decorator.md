---
title: Python装饰器(decorator)
date: 2018-05-29 19:10:09
tags:
    - Python
---
装饰器语法是Python中一个很重要的语法，刚学Python时就有接触，但当时理解起来很困难，不过最近这一次学习装饰器的时候豁然开朗，可能是看的次数多了吧，于是记录下来，加深对装饰器的理解。

装饰器是一种设计模式，我买了一本设计模式的书还没有开始看，关于设计模式这一块以后再讨论。

装饰器是用于给一些已有的函数添加一些额外功能的，比如计时、日记、计数等等。这些功能其实也是可以在已有函数中直接实现的，但是为什么不这样做呢？大概有下面这些原因：

1. 已有函数遍布整个项目，修改起来很麻烦，且容易出错
2. 已有函数为第三方库，无法修改
3. 需要添加的功能与函数本来的逻辑没有任何关系，将其添加进原函数不合适。


<!--more-->
## 装饰器基础

理解装饰器首先要知道在Python中，函数也是一个对象。像String,List,tuple这些数据结构一样，可以作为其他函数的参数传递。

```python
def test(func):
    print('begin')
    func()
    print('after')

def func_a():
    print("i'm func_a")

test(func_a)
# output:
# begin
# i'm func_a
# after
```

其次需要知道的是函数也可以定义在另一个函数的内部并且可以被返回。

```python
def test():
    print('1111')
    def inner():
        print('22222')
    return inner

a = test()
# output: 1111 注意这时已经有输出了
a()
# output: 2222
```

## decorator语法糖

有了上面那些基础知识，我们已经有能力理解装饰器了。既然装饰器是语法糖，肯定是简化后的语法。那么他没化简之前是一个什么模样呢？

```python
def decorator(a_func_to_decorator):
    def wrapper():
        # do something before func
        print('begin------')
        a_func_to_decorator()
        # do something after func
        print('after------')
    return wrapper

def func_to_decorator():
    print("chenfeikun zhen shuai")

func_to_decorator = decorator(func_to_decorator)
func_to_decorator()
# output:
# begin------
# chenfeikun zhen shuai
# after------
```

上面这就是一个最简单的装饰器了，以后遇到每一个需要装饰的函数，只要将这个函数作为参数重新传入`decorator`函数就行。但是这样写不好看啊，所以Python就推出了一个更漂亮的写法。

```python
def decorator(a_func_to_decorator):
    # 如上

@decorator
def func_to_decorator():
    print("chenfeikun tai shuai le")

func_to_decorator()
# output:
# begin------
# chenfeikun tai shuai le
# after------
```

所以`@decorator`是对`func_to_decorator = decorator(func_to_decorator)`的一种简写，也就是语法糖。

## 传递参数给被装饰的函数

当然我们需要被装饰的函数一般不会这么简单，这个被装饰的函数可能是需要参数的，那么如何把这些参数原封不动的传递给被装饰的函数呢？因为被装饰函数已经作为最外层函数的参数，所以只能放在里层的`wrapper`函数里了。

```python
def decorator(a_func_to_decorator):
    def wrapper(a, b):
        print('begin-----')
        print(a_func_to_decorator(a, b))
        print('after-----')
    return wrapper

@decorator
def add(c, d):
    return c+d

add(5, 9)
# output:
# begin-----
# 14
# after-----
```

## 方法与函数

上面我们提到的都是函数,如果类里的方法也需要装饰呢？其实方法和函数几乎是一样的，唯一的区别就是方法需要一个**self**参数。上面也提到了如何传参数，所以很容易就可以写出下面的方法装饰器。

```python
def decorator(a_method_to_decorator):
    def wrapper(self, name):
        print('begin-----')
        a_method_to_decorator(self, name)
        print('after-----')
    return wrapper

class Car(object):
    @decorator
    def talk(self, name):
        print('Hi, {}. dududu~~~'.format(name))

audi = Car()
audi.talk('chenfeikun')
# output:
# begin-----
# Hi, chenfeikun. dududu~~~
# after-----
```

正因为函数和方法是如此的相似，我们有时侯可能想要写一个适用于两者的装饰器。那就别忘了Python还有一个重要的特性，它可以传递可变个数的参数，即`*args`和`**kwargs`，这两个参数的含义应该都知道，不知道的可以去看一下，很简单。所以我们又可以写出函数和方法通用的装饰器。

```python
def decorator(something_to_decorator):
    def wrapper(*args, **kwargs):
        print('begin-----')
        print('args: {}'.format(args))
        print('kwargs: {}'.format(kwargs))
        something_to_decorator(*args, **kwargs)
        print('after-----')
    return wrapper

class Car(object):
    @decorator
    def talk(self, *args, **kwargs):
        print('talk-args: {}'.format(args))
        print('talk-kwargs: {}'.format(kwargs))

@decorator
def talk_2(name):
    print('my name is {}'.format(name))

audi = Car()
audi.talk(1,2,3,a='a',b='b')
# output:
# begin-----
# args: (<__main__.Car object at 0x7ff7c0e46240>, 1, 2, 3)
# kwargs: {'b': 'b', 'a': 'a'}
# talk-args: (1, 2, 3)
# talk-kwargs: {'b': 'b', 'a': 'a'}
# after-----
talk_2('chenfeikun')
# output:
# begin-----
# args: ('chenfeikun',)
# kwargs: {}
# my name is chenfeikun
# after-----
```

## 传递参数给装饰器

如果想向外层的那个`decorator`函数传递参数呢，这能不能做到呢？当然是可以的，不过外层的`decorator`函数和内层的`wrapper`函数的参数都已经被上面提到的功能使用了，哪里还有位置放置新的参数呢？

这其实和装饰器的概念理解起来是一样的，`decorator`函数可以返回一个`wrapper`函数，那么我们可以用一个更外层的函数**A**(不会取名字了，将就一下)来包住`decorator`函数，并将其返回，**A**函数的作用就是返回/生成一个装饰器，那么就可以给**A**传递一些参数来控制如何装饰器，多说无益，代码可能看着简单一些。

```python
def A(choose):
    def decorator_a(a_func_to_decorator):
        def wrapper():
            print('decorator_A---begin')
            a_func_to_decorator()
            print('decorator_A---after')
        return wrapper

    def decorator_b(a_func_to_decorator):
        def wrapper():
            print('decorator_B---begin')
            a_func_to_decorator()
            print('decorator_B---after')
        return wrapper

    if choose == 'a':
        return decorator_a
    return decorator_b

type_a = A('a')
@type_a
def talk_1():
    print('talk_1---dududududu~~~~~')

@A('b')
def talk_2():
    print('talk_2---dududududu~~~~~')
talk_1()
# output:
# decorator_A---begin
# talk_1---dududududu~~~~~
# decorator_A---after
talk_2()
# output:
# decorator_B---begin
# talk_2---dududududu~~~~~
# decorator_B---after
```

对比一下上面**talk_1**和**talk_2**的写法，其实是一样的，但是后面一种更加简单，更符合装饰器的形式。

## functools模块

上面的装饰器还有一个毛病，最后一个代码中的`talk_2`函数其实已经不是原来的`talk_2`函数了，因为装饰器的最初样子为
```python
talk_2 = decorator(talk_2)
```
相当于talk_2指向了另外一个对象，这个对象是decorator返回的wrapper，原来与**talk2**绑定的一些属性就会被修改。而**functools**模块就是帮助我们不让这些东西被修改，下面举个例子。

```python
def test():
    pass

print(test.__name__)
# output:
# test
def decorator(func):
    def wrapper():
        func()
    return wrapper

@decorator
def test():
    pass

print(test.__name__)
# output:
# wrapper
```

可以看到函数的__name__属性被修改了，但是用了functools模块可以解决这个问题。

```python
import functools
def decorator(func):
    @functools.wraps(func)
    def wrapper():
        func()
    return wrapper

@decorator
def test():
    pass

print(test.__name__)
# output:
# test
```

## 参考资料

[stackoverflow](https://stackoverflow.com/questions/739654/how-to-make-a-chain-of-function-decorators)