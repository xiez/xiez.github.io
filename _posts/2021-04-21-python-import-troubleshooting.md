---
title: Python import 问题排查
classes: wide
categories:
  - 2021-04
tags:
  - python
---


今天在某个容器环境里执行 `python manage.py shell` 时遇到如下错误：

```
Traceback (most recent call last):
  File "manage.py", line 22, in <module>
    execute_from_command_line(sys.argv)
  File "/usr/local/lib/python3.6/site-packages/django/core/management/__init__.py", line 383, in execute_from_command_line
    utility.execute()
  File "/usr/local/lib/python3.6/site-packages/django/core/management/__init__.py", line 377, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "/usr/local/lib/python3.6/site-packages/django/core/management/__init__.py", line 226, in fetch_command
    klass = load_command_class(app_name, subcommand)
  File "/usr/local/lib/python3.6/site-packages/django/core/management/__init__.py", line 38, in load_command_class
    module = import_module('%s.management.commands.%s' % (app_name, name))
  File "/usr/local/lib/python3.6/importlib/__init__.py", line 128, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 987, in _gcd_import
    return _find_and_load(name, _gcd_import)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 970, in _find_and_load
    return _find_and_load_unlocked(name, import_)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 942, in _find_and_load_unlocked
    _call_with_frames_removed(import_, parent)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 205, in _call_with_frames_removed
    return f(*args, **kwds)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 987, in _gcd_import
    return _find_and_load(name, _gcd_import)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 970, in _find_and_load
    return _find_and_load_unlocked(name, import_)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 959, in _find_and_load_unlocked
    module = _load_unlocked(spec)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 651, in _load_unlocked
    module = module_from_spec(spec)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 569, in module_from_spec
    _init_module_attrs(spec, module)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 512, in _init_module_attrs
    raise NotImplementedError
NotImplementedError
```

开始以为是项目里某些地方弄乱了`pythonpath`, 后来在新的目录里执行 `django-admin startproject` 还是一样的错误。


看了下 `load_command_class` 这个函数，里面调用了 python 的`import_module` 函数。

```
vi /usr/local/lib/python3.6/site-packages/django/core/management/__init__.py  +38

def load_command_class(app_name, name):
    """
    Given a command name and an application name, return the Command
    class instance. Allow all errors raised by the import process
    (ImportError, AttributeError) to propagate.
    """
    module = import_module('%s.management.commands.%s' % (app_name, name))
    return module.Command()
```

起个 python 终端，手工调用 `import_module`，错误重现：

```
root@13a7ab176589:~# python3
Python 3.6.2 (default, Dec  7 2018, 02:38:42)
[GCC 4.8.4] on linux
>>> from importlib import import_module
>>> m = import_module('django.core.management.commands.startproject')

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  ...
  ...
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 512, in _init_module_attrs
    raise NotImplementedError
NotImplementedError
```

通过 print 调试后，发现 python 会到这个目录 `/usr/local/lib/python3.6/site-packages/django/core/management/commands` 搜索模块。

看了下这个目录，没有 `__init__.py` 文件。

```
root@13a7ab176589:~# ll /usr/local/lib/python3.6/site-packages/django/core/management/commands
total 212
drwxr-xr-x 3 root root  4096 Apr 21 11:19 ./
drwxr-xr-x 4 root root  4096 Apr 21 15:55 ../
drwxr-xr-x 2 root root  4096 Apr 21 11:43 __pycache__/
-rw-r--r-- 1 root root  2248 Apr 20 17:39 check.py
-rw-r--r-- 1 root root  5729 Apr 20 17:39 compilemessages.py
-rw-r--r-- 1 root root  4322 Apr 20 17:39 createcachetable.py
-rw-r--r-- 1 root root  1207 Apr 20 17:39 dbshell.py
-rw-r--r-- 1 root root  3373 Apr 20 17:39 diffsettings.py
-rw-r--r-- 1 root root  8479 Apr 20 17:39 dumpdata.py
-rw-r--r-- 1 root root  3557 Apr 20 17:39 flush.py
-rw-r--r-- 1 root root 13763 Apr 20 17:39 inspectdb.py
...
```

手工添加 `__init__.py` 后，`django-admin startproject` 可以正常工作。

---

那么问题来了，为什么会没有 `__init__.py`，这个文件在 python 里用来标识一个目录为 package，从而可以被 `import`。

翻了几页 Django 项目在 github 上的 commit 历史，发现在这个 [commit](https://github.com/django/django/commit/ccc25bfe4f0964a00df3af6f91c2d9e20159a0c2) 被移除了。通过 commit 信息，继而找到这个 [ticket](https://code.djangoproject.com/ticket/23919)，里面主要记录了移除 python2 相关的条目，里面提到了 `namespace package`。

### Namespace Package

后来有在 SO 上找个这个[帖子](https://stackoverflow.com/questions/37139786/is-init-py-not-required-for-packages-in-python-3-3)，从 python3.3 开始，引入了 [Implicit Namespace Package](https://www.python.org/dev/peps/pep-0420/)，如果目录没有`__init__.py`，那么这个目录会被当作 `namespace package` 被加载，与之对应的是 `regular package`。

通过测试发现确实如此，

```
root@13a7ab176589:~# ls ba*
bar:
__init__.py  __init__.pyc  __pycache__  a.py  b

baz:
__pycache__  shell.py

root@13a7ab176589:~# python3
Python 3.6.2 (default, Dec  7 2018, 02:38:42)
[GCC 4.8.4] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import bar
>>> import baz
>>>
```

然而，`import_module` 却无法工作，

```
root@13a7ab176589:~# python3
Python 3.6.2 (default, Dec  7 2018, 02:38:42)
[GCC 4.8.4] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from importlib import import_module
>>> m = import_module('bar')
>>> m = import_module('baz')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python3.6/importlib/__init__.py", line 128, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 987, in _gcd_import
    return _find_and_load(name, _gcd_import)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 970, in _find_and_load
    return _find_and_load_unlocked(name, import_)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 959, in _find_and_load_unlocked
    module = _load_unlocked(spec)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 651, in _load_unlocked
    module = module_from_spec(spec)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 569, in module_from_spec
    _init_module_attrs(spec, module)
  File "/usr/local/lib/python3.6/importlib/_bootstrap.py", line 512, in _init_module_attrs
    raise NotImplementedError
NotImplementedError
```

初步怀疑是 python 3.6.2 的bug，于是找 3.6.8 测试，发现可以工作。

```
➜  Desktop python3
Python 3.6.8 (v3.6.8:3c6b436a57, Dec 24 2018, 02:04:31)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from importlib import import_module
>>> m = import_module('bar')
>>> m = import_module('baz')
>>>
```

OK, 问题解决，耗时一小时左右。


## 经验教训

保持工具库的版本更新（至少补丁版本需要最新），不仅可以消除安全问题，还可以规避各种诡异的bug引起的时间投入。

## 思考：Implicit Namespace Package

Implicit Namespace Package 解决的问题背景在[这篇PEP](https://www.python.org/dev/peps/pep-0402/#id9)以及 [Guido 的邮件](https://mail.python.org/pipermail/python-dev/2006-April/064400.html)里有介绍。

正如[这篇帖子](https://stackoverflow.com/a/48804718/1490421)所讲，99%的情况下，我们需要的是 regular package，为了解决极少数的情况，引入了 Implicit Namespace Package 从而增加语言本身的复杂度，真的好吗？

另外，附上 [Zen of Python](https://www.python.org/dev/peps/pep-0020/) 第二条:

> Explicit is better than implicit.


## 推荐阅读

- [http://python-notes.curiousefficiency.org/en/latest/python_concepts/import_traps.html](http://python-notes.curiousefficiency.org/en/latest/python_concepts/import_traps.html)

