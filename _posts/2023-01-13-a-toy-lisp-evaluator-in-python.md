---
title: 一个 Python 实现的 Lisp 解释器
classes: wide
categories:
  - 2023-01
tags:
  - python
  - lisp
---

SICP 第四章里详细讲解了自循环解释器（meta-circular evaluator），也就是使用 Lisp 语言实现一个解释器来运行 Lisp 程序。该解释器只有几百行 Scheme 代码，却展示了一些重要的概念，例如 eval-apply 循环，静态绑定（static binding），延迟求值等。为了加深理解，这里使用 Python3 来实现这个解释器的一部分，目标是能够运行书上第二章习题 2.42 的“八皇后”代码。

在本文里，我们不使用任何的面向对象的技术，只使用list这种基础的数据结构，这样更贴近书本上的实现。

## 解析器（parser）

解析器作为解释器的「前端」，主要负责把用户输入的代码转换成内部的数据结构，也就是所谓的抽象语法树（AST）。我们沿用[上篇文章](https://blog.justinzx.com/lisp-expression-parsing/)里的 parser，最后出来的 AST 如下：

```
>>> parse('''(define (fib n)
  (if (< n 2)
      1
      (+ (fib (- n 2))
         (fib (- n 1)))))''')

['define', ['fib', 'n'], ['if', ['<', 'n', 2], 1, ['+', ['fib', ['-', 'n', 2]], ['fib', ['-', 'n', 1]]]]]
```

这里我们使用列表嵌套列表的方式来表示树型的数据结构。这种数据结构也称为「表达式」（expression），表达式分两种：

- 基础表达式（primitive expression），例如： `42`，`3.14`之类的数值，`"hello"`之类的字符串以及`+`,`*`之类的符号。

- 复合表达式（compound expression），例如：`['+', 1, 2]` 组合了多个基础表达式。

所有复合表达式在 Python 里的数据结构都是列表，而基础表达式在 Python 里表示为数值和字符串。

复合表达式也称为组合（combination），最左边的元素称为操作符（operator）, 剩下的元素称为操作数（operands）。

## 解释-应用循环（eval-apply cycle）

解释器的核心是两个函数 `eval` 和 `apply`,

`eval` 负责分析表达式的语义，如果是基础表达式中的数值和字符串，则转换为 Python 对应的类型；如果是符号，则查找符号对应的值。如果是复合表达式，则根据表达式的操作符来选择不同的处理方式。

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

- 数值和字符串：`42`,`3.14`或者`"hello"`之类的由双引号包裹的字符串

- 符号：`foo` `bar` 之类的符号

- 条件语句：`['if', ...]`

- 多个条件语句：`['cond', ...]`

- lambda 函数： `['lambda', ...]`

- 变量或函数定义：`['define', ...]`

- 除上述之外的其他复合表达式：`['foo', ...]`

### number and string

能够被转换成 Python 的`int`和`float`以及`str`的基础表达式，我们称为能够被 `self_evaluating`。例如：`123`, `3.14`, `"hello"`

**注意：** 这里我们仅支持双引号包裹的字符串。

例如：

```
** >> 1
1
** >> 3.14
3.14
** >> "hello"
hello
```

### variable

无法被 `self_evaluating` 的基础表达式，统称为变量，例如：`+`, `hello`等。当解释器遇到变量时，需要从环境（Environment）里查找变量对应的值，如果没有则报错。

例如：

```
** >> +
<built-in function add>
** >> square
Error: square is not defined in the ENV.
```

### if

if 表达式的语法为：`(if <predicate> <consequent> <alternative>)`，例如：

```
** >> (if (= 1 1) true false)
True
** >> (if (= 1 2) true false)
False
```

对应的表达式如下：

```
['if', predicate-expr, consequent-expr, alternative-expr]
```

`predicate-expr` 可以为`true`, `false`之类的基础表达式，也可以为`['=', 1, 2]`之类的复合表达式。如果 `predicate-expr` 为 True，则执行`consequent-expr`，否则执行 `alternative-expr`。


### cond

cond 表达式的语法为：

```
(cond (predicate-1 expression-1)
      (predicate-2 expression-2)
      ...
      (else expression-n))
```

例如：

```
** >> (cond ((= 1 1) "1==1") ((= 2 2) "2==2") (else "else.."))
1==1
** >> (cond ((= 1 2) "1==1") ((= 2 2) "2==2") (else "else.."))
2==2
** >> (cond ((= 1 2) "1==1") ((= 2 3) "2==2") (else "else.."))
else..
```

对应的表达式如下：

```
['cond', [predicate-1, expr-1], [predicate-2, expr-2], ..., ['else', expr-n]]
```

我们把 cond 表达式转换成嵌套的 if 表达式：

```
['if', predicate-1, expr-1,
    ['if', predicate-2, expr-2,
        expr-n
        ]]
```

这样做一层转换，可以不引入额外的分支处理逻辑，让解释器的代码更简洁和统一。

### lambda

lambda 表达式的语法为：

```
(lambda (arguments) body)
```

例如：

```
** >> (lambda (x) (+ 1 x))
['procedure', ['x'], [['+', 1, 'x']], [{'+': <built-in function add>, '-': <built-in function sub>, '*': <built-in function mul>, '/': <built-in function truediv>, '=': <built-in function eq>, '<': <built-in function lt>, '>': <built-in function gt>, 'display': <built-in function print>, 'cons': <function <lambda> at 0x10ab925e0>, 'car': <function <lambda> at 0x10ab92820>, 'cdr': <function <lambda> at 0x10ab92790>, 'null': (), 'null?': <function <lambda> at 0x10ab92a60>, 'true': True, 'false': False, 'list': <function list_impl at 0x10ab92700>, 'abs': <built-in function abs>}]]
```

对应的表达式如下：

```
['lambda', [formal-arguments], body]
```

lambda 一词等同于 `make-procedure`，也就是构建一个函数，最后返回的是一个函数对象，包含了形式参数（formal-arguments）、函数体以及对参数和函数体求值所需要的环境（ENV）。

### define

define 表达式既可以定义变量，也可以定义函数，语法为：

```
(define variable value)

(define (name arguments) body)
```

例如：

```
** >> (define a 42)
None
** >> (define (foo x) (+ x 1))
None
```

对应的表达式如下：

```
['define', variable, value]

['define', [name, arguments], body]
```

如果是变量定义，则把 `vairable -> value` 关系写入到环境里。

如果是函数定义，则先转换成 lambda 表达式，并对其求值后生成一个函数对象，再把 `name -> procedure` 关系写入到环境里。

### combination

除上述特定语义的表达式外，其他复合表达式统一归为函数，需要递归对操作符和操作数求值后，把函数对象（procedure）以及参数值（actual-arguments）传给 `apply` 函数，`apply`负责具体的函数应用。例如：

`(+ a 1)` 对应的表达式 `['+', 'a', 1]` 的求值步骤如下：

1. `eval('+', ENV)` -> `<built-in function add>`

2. `eval('a', ENV)` -> 42

3. `eval(1, ENV)` -> 1

4. 把函数对象`<built-in function add>`以及对应的实际参数列表`(42,1)`传给 `apply`做具体的函数应用。


`(foo 1)` 对应的表达式`['foo', 1]`的求值步骤如下：

1. `eval('foo', ENV)` -> 函数对象 `['procedure', [], [['+', 3, 1]], ENV]`

2. `eval(1, ENV)` -> 1

3. 把函数对象`['procedure', [], [['+', 3, 1]], ENV]`以及对应的实际参数列表`(1)`传给 `apply`做具体的函数应用。

---

`apply` 负责具体的函数应用，如果是 Python 内置函数，则调用 Python 的 `apply` 函数。如果是自定义的函数对象，则在**新的环境**里依次对函数体求值。新的环境扩展了当前的环境，并包含了形参到实际参数的映射`formal-arguments->actual-arguments`。

```
def apply_(proc, args):
    if is_primitive_proc(proc):
        return apply_in_python(proc, args)
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

`eval` 和 `apply` 这两个函数互相调用，形成了互递归（mutual recursion）。如下图：

![eval-apply cycle](https://sarabander.github.io/sicp/html/fig/chap4/Fig4.1a.std.svg)



### 环境模型（Environment Model）

表达式本身没有意义，一个表达式只有在某个环境中才有意义。例如：`['+', 1, 1]`，只有在环境中找个'+'对应的函数，整个表达式才有意义。因此，`eval` 函数都需要带上 env 环境参数。

SICP 3.2章:

> Indeed, one could say that expressions in a programming language do not, in themselves, have any meaning. Rather, an expression acquires a meaning only with respect to some environment in which it is evaluated.

环境是多个 frame 的序列，frame 类似哈希表，包含某个符号对应的值。每次执行 `apply <proc-obj>` 会新创建一个新的 frame，插入到序列的开头，从而成为一个新的环境。

在一个环境里，获取一个变量的值需要遍历多个frame，直到找到第一个包含该变量的frame；如果遍历完成没有找到，则表示该变量未绑定。

我们通过3个具体的例子来看环境模型如何作用在 eval/apply 循环里。


### 递归函数

阶乘函数是最简单的一类递归函数。

```scheme
** >> (define (fact n)
  (if (< n 2)
      1
      (* n (fact (- n 1)))))

** >> (fact 5)
120
```

`define fact (λ (n) ...)` 经过`eval`后，在全局的环境变量E0里绑定了 `fact -> <fact-proc>`，其中，`fact-proc`包含一个指向环境 E0 的指针。

`(fact 5)` 在环境 E0 的 eval/apply 步骤如下：

1. `(fact 5)` 是复合表达式（combination），先 eval 操作符`fact`，再 eval 操作数`(5)`，最后 apply 两者的结果

    1. `fact` 的 eval 结果为 `<fact-proc>`

2. 操作数`(5)`的 eval 结果为`(5)`

3. apply 上面的结果

    1. 新增 frame `{n->5}`，作为新的环境 E1: `{n->5} -> E0`

    2. eval `<fact-proc>` 的内容 `['if', ...]`，当前 n 为5，eval `(* n (fact (- n 1)))`这个复合表达式，

        1. eval `(fact 4)` in E1

        2. apply ...,  新环境 E2: `{n->4} -> E1`

            1. eval `(fact 3)` in E2

            2. apply ..., 新环境 E3: `{n->3} -> E2`

                1. eval `(fact 2)` in E3

                2. apply ..., 新环境 E4: `{n->2} -> E3`

                    1. eval `(fact 1)`，in E4

                    2. apply ..., 新环境 E5: `{n->1} -> E4`

最后，Python 帮我们把调用栈上的 n 相乘得到 120。



#### 内嵌函数

内嵌函数是指在函数里再定义多个局部函数，这些局部函数可以使用外部函数的变量值，也就是所谓的「闭包」（closure）。


```scheme
** >> (define (foo a)
  (define (bar x)
    (+ x a))
  (bar 42))

** >> (foo 1)
43

```

`(foo 1)` 在环境 E0 的 eval/apply 步骤如下：

1. `(foo 1)` 是复合表达式，先 eval 操作符`foo`，再 eval 操作数`(1)`，最后 apply 两者的结果

    1. `foo` 的 eval 结果为 `<foo-proc>`

2. 操作数`(1)`的 eval 结果为`(1)`

3. apply 上面的结果

    1. 新增 frame `{a->1}`，作为新的环境 E1: `{a->1} -> E0`

    2. eval `<foo-proc>` 的内容 `['define', ...]`， E1 新增 frame `bar-><bar-proc>`，

        1. eval `(bar 42)` in E1， 先 eval 操作符`bar`得到 `<bar-proc>`, 再 eval 操作数 `(42)` 得到 `(42)`

        2. apply 上述结果

            1. 新增 frame `{x->42}`，作为新的环境 E2: `{x->42} -> E1`

            2. eval `(+ x a)` in E2，其中 x 在 E2 中绑定，a 在 E1 中绑定，所以，结果为42


#### 高阶函数

高阶函数是指一个函数的返回值为函数对象，而不是具体的值。

```scheme
** >> (define (f x)
  (define (g)
    x)
  g)

** >> (f 10)
['procedure', [], ['x'], [{'x': 10, 'g': [...]}, {...}]]

** >> ((f 10))
10
```

`((f 10))` 在环境 E0 的 eval/apply 步骤如下：

1. `((f 10))` 是复合表达式，先 eval 操作符`(f 10)`，再 eval 操作数`()`，最后 apply 两者的结果
 
    1. `(f 10)` 也是复合表达式, 先 eval 操作符`f` ，得到函数对象 `<f>`, 再 eval 操作数 `10`，结果为10
 
    2. apply `<f>` 和 `10`
 
        1. 新增 frame `{x->10}`，作为新的环境 E1
 
        2. 在环境 E1 里 eval `<f>` 的内容（ `(define (g) x)` 在 E1 里绑定了`g -> <g>`，`g`在 E1 里查找 g 对应的函数对象`<g>`并返回）
 
2. 操作数`()`的 eval 结果为空列表
 
3. apply 上面的结果
 
    1. 新增空 frame，作为新的环境 E2
 
    2. eval `<g>` 的内容（`x` 在 E1 里绑定，结果为 10）


#### 静态绑定 vs 动态绑定

静态绑定也叫词汇绑定（lexical binding），

Racket 例子如下：

```scheme
> (let* ((a 1) (f (lambda (x) (+ x a))))
    (let ((a 2))
      (f 1)))
2
```

```scheme
> (let* ((a (make-parameter 1)) (f (lambda (x) (+ x (a)))))
    (let ((c 1))
      (parameterize ((a 42))
        (f 1))))
43
```


#### 优点 vs 缺点

环境模型有两个特点：

1. 每次 apply 函数对象都会插入新的 frame 作为新的 eval 环境，不同环境间数据隔离。

2. 局部函数可以通过父环境指针找到“包裹”（enclosing）它的所有变量绑定。

这两个特点可以让我们实现类似面向对象里的 class 的概念。

Racket 例子如下：

```racket
(define (make-account balance)
  (define (withdraw amount)
    (set! balance
          (- balance amount))
    balance)
  (define (deposit amount)
    (set! balance
          (+ balance amount))
    balance)
  (define (dispatch op)
    (cond ((equal? op 'balance) balance)
           ((equal? op 'withdraw) withdraw)
           ((equal? op 'deposit) deposit)
           (else (error "Unknown op -- " op))))
  dispatch)

(define A (make-account 100))
(define B (make-account 100))
(A 'balance)                            ;100
(B 'balance)                            ;100
((A 'withdraw) 10)                      ;90
((B 'withdraw) 20)                      ;80
((A 'deposit) 20)                       ;110
((B 'deposit) 10)                       ;90
```

对应的 Python 代码：

```python
def make_account(balance):
    def deposit(amount):
        nonlocal balance
        balance += amount
        return balance
    def withdraw(amount):
        nonlocal balance
        balance -= amount
        return balance
    def dispatch(op, val=0):
        if op == 'balance':
            return balance
        elif op == 'deposit':
            return deposit
        elif op == 'withdraw':
            return withdraw
        else:
            assert False, f'Invalid op: {op}'
    return dispatch
```

环境模型在实现上，需要遍历所有的 frame 才能查到某个变量，性能上不够高效，所以真实的解释器一般会借助于 lexical addressing 的策略，通过在 frame 上标注额外信息来快速定位到某一个 frame 。具体可参照 SICP 5.5.6。


## 读取-解释-输出循环（REPL）


