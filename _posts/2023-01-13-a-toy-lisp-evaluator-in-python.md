---
title: 一个 Python 实现的 Lisp 解释器
classes: wide
categories:
  - 2023-01
tags:
  - python
  - lisp
---

SICP 第四章里详细讲解了自循环解释器（meta-circular evaluator），也就是使用 Lisp 语言实现一个解释器来运行 Lisp 程序。该解释器只有几百行 Scheme 代码，却展示了一些重要的概念，例如 eval-apply 循环，动态绑定，延迟求值等。为了加深理解，这里使用 Python3 来实现这个解释器的一部分，目标是能够运行书上第二章习题 2.42 的“八皇后”代码。

在本文里，我们不使用任何的面向对象的技术，只使用list，tupple这类基础的数据结构，这样更贴近书本上的实现。

## 解析器（parser）

解析器作为解释器的「前端」，主要负责把用户输入的代码转换成内部的数据结构，也就是所谓的抽象语法树（AST）。我们沿用[上篇文章](https://blog.justinzx.com/lisp-expression-parsing/)里的 parser，最后出来的 AST 如下：

```
['defun', 'factorial', ['x'], ['if', ['zerop', 'x'], 1, ['*', 'x', ['factorial', ['-', 'x', 1]]]]]
```

这里我们使用列表嵌套列表的方式来表示树型的数据结构。这种数据结构也成为表达式（expression）。有两类表达式：

- 基础表达式（primitive expression）

基础表达式例如： `42`， `+`

- 复合表达式（compound expression）



## 解释-应用循环（eval-apply cycle）

解释器的核心是两个函数 `eval` 和 `apply`,

`eval` 负责分析表达式的语义，根据不同类型选择不同的执行路径。

```
def eval_(exp, env):
    if is_self_evaluating(exp):
        return self_evaluating(exp)
    elif is_variable(exp):
        return lookup_variable_value(exp, env)
    elif is_if(exp):
        return eval_if(exp, env)
    elif is_cond(exp):
        return eval_if(cond_to_if(exp), env)
    elif is_lambda(exp):
        return make_compound_procedure(
            lambda_parameters(exp),
            lambda_body(exp),
            env,
        )
    elif is_definination(exp):
        return eval_defination(exp, env)
    elif is_combination(exp):
        return apply_(
            eval_(get_combination_operator(exp), env),
            list_of_values(get_combination_operands(exp), env),
        )
    else:
        raise Exception(f"Unknown expression type -- EVAL {exp}")
```

表达式的类型如下：

- `42`,`3.14`或者`"hello"`之类的基础类型，则转为 Python 里的数值或字符串。

- `foo`之类的变量，则转为这个变量对应的值。

- `['if', ...]`，则根据条件执行相应的分支。

- `['cond', ...]`，则转换成嵌套的if，并根据测试条件执行相应的分支。

- `['lambda', ...]`，则构造对应的函数。

- `['define', ...]`，则执行变量赋值。

- `['foo', ...]` 之类的其他列表，则当作函数调用 `apply`。



`apply` 负责把参数应用到函数里，这里的函数可能是基础函数（`+`, `abs`等），也可能是自定义函数（`fib`, `queens`等）。

```
def apply_(proc, args):
    if is_primitive_proc(proc):
        return run_python_apply(proc, args)
    elif is_compound_procedure(proc):
        return eval_sequence(
            procedure_body(proc),
            extend_environment(
                procedure_params(proc), args, procedure_environment(proc)
            ),
        )
    else:
        raise Exception(f"Unknown procedure type -- APPLY {proc}")
```

这两个函数互相调用，形成了互相递归。

### number and string

### variable

### if

### cond

### lambda

### define

### function

### 环境（Environment）


## 读取-解释-输出循环（REPL）


