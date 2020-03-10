---
title: SICP 第一章读书笔记 - 递归与迭代
classes: wide
categories:
  - 2020-03
tags:
  - SICP
  - Reading
---

花了三周左右时间读完了 "Structure and Interpretation of Computer Programs" 第一章，完成了大部分的[习题](https://github.com/xiez/SICP-exercises)，在这里记录下书中的一些精华段落，以及个人的一些感悟。

## Iteration vs Recursion

首先，截取一段书中对于“迭代和递归”的描述：

> The contrast between the two processes can be seen in another way. In the iterative case, the program variables provide a complete description of the state of the process at any point. If we stopped the computation between steps, all we would need to do to resume the computation is to supply the interpreter with the values of the three program variables. Not so with the recursive process. In this case there is some additional “hidden” information, maintained by the interpreter and not contained in the program variables, which indicates “where the process is” in negotiating the chain of deferred operations. The longer the chain, the more information must be maintained.

这段描述再配合上计算 factorial 的过程（process），把迭代和递归的区别清晰地展示了出来。

![iterative](https://raw.githubusercontent.com/xiez/xiez.github.io/master/assets/images/2020/03/iterative.gif "iterative process")
![recursive](https://raw.githubusercontent.com/xiez/xiez.github.io/master/assets/images/2020/03/recursive.gif "recursive process")

递归从“形状”上看，先增大再缩小；而迭代却是线性的。这里所谓的“形状”就是计算所需要的空间。
另外，递归还需要把一部分“隐藏状态”交给 interpreter 来维护；而迭代却是把所有的状态放在参数里。这样导致的结果就是，如果计算过程中断，迭代可以从中断的地方开始继续执行，而递归则不行，因为那部分隐藏的状态已经丢失了。

再截取书中的一段注解#30：

> When we discuss the implementation of procedures on register machines in chapter 5, we will see that any iterative process can be realized “in hardware” as a machine that has a fixed set of registers and no auxiliary memory. In contrast, realizing a recursive process requires a machine that uses an auxiliary data structure known as a stack.

当然，递归也有优点，递归能让我们更好的理解问题，而且在处理一些有层次关系的数据时，用递归更自然。

> One should not conclude from this that tree-recursive processes are useless. When we consider processes that operate on hierarchically structured data rather than numbers, we will find that tree recursion is a natural and powerful tool.32 But even in numerical operations, tree-recursive processes can be useful in helping us to understand and design programs.

此外，书中把 Process 和 Procedure 的对比描述地也很精彩。简单来说，Process 属于概念上的东西，跟具体语言无关，而 Procedure 是用具体的编程语言把 Process 实现出来。

迭代过程（iterative process）也可以用递归程序（recursive procedure）来实现，因为 Scheme 之类的语言实现支持“尾递归”（tail-recursive），因此常见的 for,while 之类的循环原语只是作为语法糖（syntatic sugar）而存在。而其他没有支持尾递归的语言（C, Python）则需要借助 for,while 来实现迭代过程。

读完第一章后，不得不佩服本书的两位作者，平淡的几句话就能把很多深刻的问题解释地非常透彻，可能这也就是为什么这本书在老派的程序员当中相当受推崇。

接下来几天（也许几周？）准备写下第一章的重点：使用程序（procedure）来构建抽象，从而使得复杂的逻辑更简单易懂，也更灵活易改。

P.S. 如果想对递归/尾递归有更深入的了解，[这篇文章](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/)绝对值得一看～
