---
title: Lisp expression parsing
classes: wide
categories:
  - 2022-10
tags:
  - python
  - scheme
  - recursive-descent-parsing
  - compilation
type: en
---

Lisp expressions, also known as [symbolic expressions](https://en.wikipedia.org/wiki/S-expression) (S-expressions), are prefixed expressions. In Lisp, the `(quote)` function or `'` can easily parse symbolic expressions into internal list data structures, whereas in other languages, it would take some effort to do a similar conversion.

```lisp
> '(list 1 2 3)
(list 1 2 3)
> '(defun factorial (x)
   (if (zerop x)
       1
       (* x (factorial (- x 1)))))
(defun factorial (x) (if (zerop x) 1 (* x (factorial (- x 1)))))
```

I recently spent a few days looking through the parsing-related topics in the [Dragon Book](https://www.amazon.com/Compilers-Principles-Techniques-Tools-2nd/dp/0321486811) and the [Tiger Book](https://www.amazon.com/Modern-Compiler-Implement-Andrew-Appel/dp/0521607655), and deepened my understanding of some common terms (context-free grammar, recursive descent parsing, LL(1)), and so on. This is recorded here for future reference.

Finally, a parser was implemented using Python, with the following result,

```
>>> parse('(list 1 2 3)')
['list', 1, 2, 3]
>>> parse('''(defun factorial (x)
   (if (zerop x)
       1
       (* x (factorial (- x 1)))))''')
['defun', 'factorial', ['x'], ['if', ['zerop', 'x'], 1, ['*', 'x', ['factorial', ['-', 'x', 1]]]]]
```

## Grammar definition

First, it is necessary to define **context-free grammar**, also called **grammar**. The Lisp expression syntax to be implemented is described as follows.

```
expr -> "(" operator operands ")"
operator -> "string"|expr
operands -> operand*
operand -> "int"|"float"|"string"|expr
```

This syntax contains 4 **production rules**[^1], where the one in double quotes is the **terminal symbol**, the one to the left of `->` is the **nonterminal symbol**, and `->` is read as "can have the form:".

An expression has the form: wrapped in parentheses, containing an `operator` and zero or more `operand`s.

An `operator` has the form: a string or another expression; an `operand` has the form: an integer, a floating point number, a string, or another expression.

These 4 rules form a tree structure, with the root node being the start symbol `expr`, the leaf nodes corresponding to terminals, and the non-leaf nodes corresponding to nonterminals.


## Parse tree

With the syntax defined, we can have **sentences** such as `(string int int int)`, or `(defun factorial (x))`, and so on. For a sentence, we can perform **derivation** to derive a parse tree.

The derivation of the first sentence is as follows,

```
expr
(operator operands)
(string operands)
(string int operand*)
(string int int operand*)
(string int int int)
```

The derivation of the second sentence is as follows,

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

Both examples are **leftmost** derivations, i.e., each derivation replaces the leftmost nonterminal.

A grammar is **ambiguous** if we can find a sentence in it that can derive multiple parse trees. Ambiguous grammars need to be redesigned into non-ambiguous grammars in order to be processed correctly by the program.

Our Lisp expression syntax in this case is unambiguous.

## Predictive parsing

Predictive parsing is a special example of **recursive descent parsing**, which is top-down parsing. Recursive descent is a relatively simple and intuitive algorithm, where each nonterminal (`expr, operator, operands, operand`) in the syntax corresponds to a function.

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

The function in the above implementation only needs to check the first element of `tokens` to select the correct syntax rule, such syntax belongs to the LL(1) category, other categories are LL(2), LL(3).... ...LL(k).

Predictive parsing is the process of "predicting" which rule will be used by the first k elements in `tokens`.

## Full implementation

The final full implementation can be found in [this Gist](https://gist.github.com/xiez/4551065f3168f9d1276ab9e0771313d6). To keep the code as simple as possible, lexical parsing uses `replace, split` instead of the standard regular parsing.

[^1]: The term "production" was introduced by Noam Chomsky in the mid-1950s, as part of his work on generative grammar. According to Chomsky, a generative grammar is a system of rules that can generate or produce all the possible sentences of a language. The term "production" was chosen to reflect the idea that the grammar rules are like factories that produce or generate sentences by combining simpler elements into more complex structures. Just as a factory produces goods by assembling raw materials, a grammar produces sentences by assembling words and phrases according to the rules of the language.
