---
title: 八皇后问题
classes: wide
categories:
  - 2022-10
tags:
  - python
  - scheme
  - games
  - backtracking
---

“八皇后问题”根据[维基百科的定义](https://zh.wikipedia.org/wiki/%E5%85%AB%E7%9A%87%E5%90%8E%E9%97%AE%E9%A2%98):

> 八皇后问题是一个以国际象棋为背景的问题：如何能够在8×8的国际象棋棋盘上放置八个皇后，使得任何一个皇后都无法直接吃掉其他的皇后？为了达到此目的，任两个皇后都不能处于同一条横行、纵行或斜线上。八皇后问题可以推广为更一般的n皇后摆放问题：这时棋盘的大小变为n×n，而皇后个数也变成n。当且仅当n = 1或n ≥ 4时问题有解[1]。

该问题的一个解如下图：

![8-queens](../../assets/images/2022/10/n-queens.png "8 queens")

八皇后或者N皇后问题一般分为寻找所有解法或者某一种解法。

## 寻找所有解法

这里，我们引用 SICP 第二章习题 2.42 的算法，描述大致如下：

1. 假设我们已经找到前 N-1 列皇后的所有有效放置方式，记为 S{P1, P2, ...}

2. 针对S里的每一种方式，我们把皇后放在第N列的每一行上，这样就组成了新的放置方式，记为 S'{P1R1, P1R2 ... P1Rn, P2R1, ... P2Rn, ...}

3. 从新的放置方式S'里，过滤出所有有效的放置方式，也就是最后的结果。

Python 主体代码如下：

```python
def n_queens(n):
    """返回所有的放置方式
    """
    if n == 0:
        # n为0时，没有解法
        return [[]]

    # 前 N-1 列的合法放置方式
    placements = n_queens(n-1)

    new_placements = []
    for row in range(n):
        # 把第N列的每一行，加入到之前的放置方式里, 形成新的放置方式
        new_placements += adjoin_position(placements, (row, n))

    # 最后过滤出有效的放置方式
    return filter_out_valid(new_placements)
```

其中，`adjoin`函数把第N列的皇后邻接到每一种放置方式的开头，这样后面在过滤的时候，判断第一个跟后面的几个皇后位置是否会攻击即可。

Scheme 代码如下：

```lisp
(define (queens board-size)
  (define (queen-cols k)
    (if (= k 0)
        (list empty-board)
        (filter
         (lambda (positions) (safe? k positions))
         (flatmap
          (lambda (rest-of-queens)
            (map (lambda (new-row)
                   (adjoin-position new-row k rest-of-queens))
                 (enumerate-interval 1 board-size)))
          (queen-cols (- k 1))))))
  (queen-cols board-size))
```

Scheme 代码使用了声明式的写法，也就是函数式编程，组合使用`filter`，`map`，相较于命令式的写法，可读性更高。

完整代码：[queens.py](https://github.com/xiez/SICP-exercises/blob/master/2.42/queens.py) [queens.ss](https://github.com/xiez/SICP-exercises/blob/master/2.42/exer.ss)


## 寻找一种解法

我们引用 HtDP [28.2](https://htdp.org/2003-09-26/Book/curriculum-Z-H-35.html#node_sec_28.2) 的算法，描述大致如下：

1. 初始化棋盘, 所有位置均可放置

2. 找到棋盘上第一个可以放置的位置放置皇后，同时把相同行/列/对角线上的位置设为不可放置

3. 从棋盘上找到剩下可以放置的位置

    1. 如果有，则重复第2步

    2. 如果没有，则说明第二步里找到的那个位置无法产生最后的结果，回退到第二步的下一个可以放置的地方

4. 重复2直到N个皇后都放完，或者没有位置可放

Python 主体代码如下：

```python
# 初始化一个二维数组作为空白棋盘，所有值设为 False, 标识可以放置
board = build_board(n)

def n_queens(n, board):
    if n == 0:
        # N 皇后都放置完成
        return board

    # 找到当前棋盘所有可以放置的位置
    positions = find_safe_positions(board)

    for a_posn in positions:
        new_board = placement(board, a_posn)
        ret = n_queens(n-1, new_board)
        if ret is False:
            continue
        else:
            return ret
    return False
```

其中，`placement` 将皇后放置到棋盘的一个位置，同时把相同行/列/对角线上的位置设为不可放置，最后返回一个新的棋盘。

Scheme 主体代码如下：

```lisp
;; placement : Board Nat -> board or false
;; places n queens on a-board. if possible, returns
;; the board. otherwise, returns false
(define (placement a-board n)
  (cond
   [(zero? n) a-board]
   [else
    (let [(possible-places (find-open-spots a-board))]
      (placement/list possible-places a-board n)]))
 
;; placement/list : (Listof Posn) Board Nat -> board or false
;; tries to place n queens in all of the boards. As soon as
;; one of them works out, it returns that one. If none
;; work out, returns false.
(define (placement/list possible a-board n)
  (cond
   [(null? possible) false]
   [else (let [(possible-posn (car possible))]
           (let* [(new-board (add-queen a-board possible-posn))
                  (c (placement new-board (sub1 n)))]
             (cond
              [(boolean? c)
               (placement/list (cdr possible) a-board n)]
              [else c])
             ))]))
```

Scheme 代码使用了 [mutually recursive](https://htdp.org/2003-09-26/Book/curriculum-Z-H-20.html#node_idx_1084), 使得代码更接近问题的表述。

完整代码：[queens.py](https://github.com/xiez/HtDP-exercises/blob/master/28-traversing-graphs/nqueens.py) [queens.ss](https://github.com/xiez/HtDP-exercises/blob/master/28-traversing-graphs/queens.ss)
