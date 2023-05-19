---
title: Lisp 表达式解析
classes: wide
categories:
  - 2022-10
tags:
  - python
  - scheme
  - recursive-descent-parsing
  - compilation
---

Lisp 表达式也叫符号表达式（S-expression），是一种前缀表达式。在 Lisp 里，`(quote)` 函数或者`'`可以方便地把符号表达式解析成内部的列表数据结构，而在其他语言里，如果需要做类似的转换，则需要花一些功夫。

```lisp
> '(list 1 2 3)
(list 1 2 3)
> '(defun factorial (x)
   (if (zerop x)
       1
       (* x (factorial (- x 1)))))
(defun factorial (x) (if (zerop x) 1 (* x (factorial (- x 1)))))
```

最近花了几天时间，翻看了[龙书](https://www.amazon.com/Compilers-Principles-Techniques-Tools-2nd/dp/0321486811)和[虎书](https://www.amazon.com/Modern-Compiler-Implement-Andrew-Appel/dp/0521607655)的语法解析相关内容，加深了一些常见术语（context-free grammar, recursive descent parsing, LL(1)）等的了解，在这里记录下供以后参考。最后使用 Python 实现了一版解析器，解析效果如下：

```
>>> parse('(list 1 2 3)')
['list', 1, 2, 3]
>>> parse('''(defun factorial (x)
   (if (zerop x)
       1
       (* x (factorial (- x 1)))))''')
['defun', 'factorial', ['x'], ['if', ['zerop', 'x'], 1, ['*', 'x', ['factorial', ['-', 'x', 1]]]]]
```

## 语法定义 (Grammar definition)

首先，需要定义一下上下文无关的语法，也叫语法（grammar）。这次需要实现的 Lisp 表达式语法描述如下：

```
expr -> "(" operator operands ")"
operator -> "string"|expr
operands -> operand*
operand -> "int"|"float"|"string"|expr
```

这个语法包含4条产生式规则（production rule）[^1]，其中，双引号里的为终端符号（terminal symbol），`->` 左边的为非终端符号（nonterminal symbol），`->` 读为“可以有这样的形式：”。

一个表达式的形式为：使用括号包裹，包含一个 `operator` 以及0个或多个 `operand`，

一个 `operator` 的形式为：字符串或者另一个表达式；一个 `operand` 的形式为：整数、浮点数、字符串或者另一个表达式。

这4条规则形成了一个树状结构，根节点为起始符号`expr`，叶子节点对应到 terminals，非叶子节点对应到 nonterminals。


## 解析树（Parse tree）

有了语法定义后，我们可以生成一些句子（sentence），例如：`(string int int int)`，或者`(defun factorial (x))` 等等。

针对一个句子，我们可以进行推导（derivation），从而得出一个解析树。

第一个句子的推导如下：

```
expr
(operator operands)
(string operands)
(string int operand*)
(string int int operand*)
(string int int int)
```

第二个句子的推导如下：

```
expr
(defun operands)
(defun operands)
(defun factorial operands)
(defun factorial expr)
(defun factorial (operator operands))
(defun factorial (x operands))
(defun factorial (x))
```

这两个例子都是最左推导（leftmost derivation），也就是每次推导都把最左边的 nonterminal 进行替换。

如果语法里我们可以找到一个句子可以推导出多个解析树，那么这个语法是有歧义的（ambiguous）。有歧义的语法需要被重新设计转换成无歧义的语法，才能别程序正确处理。

我们这次的 Lisp 表达式语法是无歧义的。

## 预测性解析（Predictive parsing）

预测性解析是递归下降解析（recursive descent parsing）的特殊例子，属于自上而下（Top-down）解析。递归下降是一种比较简单直观的算法，语法中的每一个 nonterminal (`expr, operator, operands, operand`) 对应到一个函数。

```
def expr():
    tokens.pop(0)    # remove (
    operator()
    operands()
    tokens.pop(0)    # remove )

def operator():
    if tokens[0] == "(":
        expr()
    else:
        tokens.pop(0)    # string

def operands():
    while tokens[0] != ")":
        operand()

def operand():
    if tokens[0] == "(":
        expr()
    else:
        tokens.pop(0)    # int, float, string
```

上述实现里的函数只需要检查`tokens`的第一个元素，就可以选择正确的语法规则，这样的语法属于 LL(1) 类别，对应的其他类别还有 LL(2), LL(3)...LL(k)。

所谓预测性解析，也就是通过`tokens`里的前k个元素，来“预测”哪一条规则被使用。

## 完整实现

最后的完整实现可以参见[这个 Gist](https://gist.github.com/xiez/4551065f3168f9d1276ab9e0771313d6)，为了让代码尽量简洁，词法解析这块使用了`replace,split`而不是标准的正则解析。

[^1]: "production"一词是由诺姆·乔姆斯基于1950年代中期提出的，作为他在生成式语法方面的工作的一部分。根据乔姆斯基的理论，生成式语法是一种可以生成（produce）语言中所有可能句子的规则系统。选择“production”这个术语是为了反映语法规则就像工厂一样，通过将更简单的元素组合成更复杂的结构来生成或产生句子的概念。就像工厂通过组装原材料生产商品一样，语法通过按照语言规则组装单词和短语来生成句子
