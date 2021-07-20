---
title: Python Metaclass
classes: wide
categories:
  - 2021-07
tags:
  - python
  - metaclass
---

Metaclass 国内翻译成「元类」，给人一种高深莫测的感觉，让第一次接触这个概念的人望而却步。

`Meta` 一词根据 [wikipedia](https://en.wikipedia.org/wiki/Meta) 上的定义，一般用做前缀，例如，`metadata` (are data about data), `metaprogramming` (writing programs that manipulate programs), 所以，也可以同样定义 `metaclass` (are class that produce/manipulate class)

`metaclass` 也是一种类，它与 `class` 的关系就像 `class` 与 `object` 的关系，`metaclass` 可以改变它所生成的类的属性和行为。

下面以 python3 为例介绍 metaclass 的语法和使用场景。

## 语法定义

Python3 里定义一个最简单的 class `Foo`, 并实例化一个对象:

```
class Foo:
    pass

>>> foo = Foo()
>>> foo
<__main__.Foo object at 0x1074a13a0>

>>> type(foo)
<class '__main__.Foo'>

>>> type(Foo)
<class 'type'>

>>> type(type)
<class 'type'>

```

另外，为了提供灵活度，也可以动态定义：

```
Foo2 = type('Foo2', (), {})

>>> foo2 = Foo2()
>>> foo2
<__main__.Foo2 object at 0x10e838be0>

>>> type(foo2)
<class '__main__.Foo2'>

>>> type(Foo2)
<class 'type'>
```

由此可见，对象 `foo` 是由 `Foo` 这个生成，而 `Foo` 是由 `type` 这个类生成。

---

接下来，我们可以改变下对象的属性，

```
class Foo:
    a = 42

>>> Foo.a
42
>>> Foo().a
42

```

已经定义好的 class 可以通过修改 `__new__` 方法来实现，

```
def new(cls):
    x = object.__new__(cls)
    x.a = 42
    return x

Foo2.__new__ = new

>>> Foo2().a
42
```

我们也可以尝试修改 `type` 的 `__new__`，

```
>>> type.__new__ = new
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't set attributes of built-in/extension type 'type'
```

然而并不允许，这个很好理解，因为一旦修改了 `type`, 那么就会影响到所有内置的 class。因此，python3 提供了一种额外的机制，这就是 `metaclass`

```
class MyMeta(type):
    def __new__(cls, name, bases, dct):
        x = super().__new__(cls, name, bases, dct)
        if name == 'Bar2':
            x.a = 43
        else:
            x.a = 42
        return x

class Bar(metaclass=MyMeta):
    pass

class Bar2(metaclass=MyMeta):
    pass

>>> Bar.a
42
>>> Bar().a
42
>>> Bar2.a
43
>>> Bar2().a
43

```

`metaclass`, `class`, `object` 之间的关系可以如下图：

![metaclass](https://github.com/xiez/xiez.github.io/raw/master/assets/images/2021/07/metaclass.png "metaclass")

## 应用场景

### Singleton

```
class Singleton(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super(Singleton, cls).__call__(*args, **kwargs)
        return cls._instances[cls]

class MyClass(metaclass=Singleton):
    pass

```

### Class Registry

```
models = {}

class ModelMeta(type):
    def __new__(meta, name, bases, dic):
        cls = super().__new__(meta, name, bases, dic)
        models[name] = cls
        return cls

class Model(metaclass=ModelMeta): pass

class ModelA(Model): pass

class ModelB(Model): pass

>>> models
{'Model': <class '__main__.Model'>, 'ModelA': <class '__main__.ModelA'>, 'ModelB': <class '__main__.ModelB'>}

```

### ORM

```
class Field: pass

class String(Field): pass

class Integer(Field): pass

class ModelMeta(type):
    def __new__(meta, name, bases, attrs):
        fields = {}
        for key, value in attrs.items():
            if isinstance(value, Field):
                value.name = '%s.%s' % (name, key)
                fields[key] = value
        for base in bases:
            if hasattr(base, '_fields'):
                fields.update(base._fields)
        attrs['_fields'] = fields
        return type.__new__(meta, name, bases, attrs)

class Model(metaclass=ModelMeta): pass

class ModelA(Model):
    id = Integer()
    name = String()

>>> ModelA._fields
{'id': <__main__.Integer object at 0x10e9fefd0>, 'name': <__main__.String object at 0x10e9fee20>}

```

## 阅读材料

- [https://realpython.com/python-metaclasses/](https://realpython.com/python-metaclasses/)

- [https://stackoverflow.com/questions/6760685/creating-a-singleton-in-python](https://stackoverflow.com/questions/6760685/creating-a-singleton-in-python)

- [https://stackoverflow.com/questions/392160/what-are-some-concrete-use-cases-for-metaclasses](https://stackoverflow.com/questions/392160/what-are-some-concrete-use-cases-for-metaclasses)

- [https://www.honeybadger.io/blog/python-instantiation-metaclass/](https://www.honeybadger.io/blog/python-instantiation-metaclass/)

- [https://eli.thegreenplace.net/2011/08/14/python-metaclasses-by-example/](https://eli.thegreenplace.net/2011/08/14/python-metaclasses-by-example/)
