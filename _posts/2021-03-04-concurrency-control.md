---
title:  并发控制
classes: wide
categories:
  - 2021-03
tags:
  - concurrency
  - database
---

并发控制的核心是在保证正确结果的基础上，尽可能快的返回结果。

## 常见的并发控制场景

1. 多个数据库事务在并发执行下，需要保证[数据完整性](https://en.wikipedia.org/wiki/Data_integrity)。

2. 多人同时更新某个资源（文档，）

## 常见的并发控制机制

1. 乐观方式（[optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)），也叫乐观锁（optimistic locking）。假定冲突不会发生，把资源冲突检查推迟到提交那一刻，如果检查到有冲突，则交给上层来处理。

**优点**：灵活，无需考虑锁的问题，死锁发生的概率很低。

**缺点**：冲突发生时，需要应用额外处理。

2. 悲观方式，也叫悲观锁。假定冲突会发生，一开始就用锁把资源锁住，直到使用完成后才释放锁，在此期间，其他人无法使用该资源。

**优点**：从源头上避免了冲突的发生。

**缺点**：用到了锁，有相对高的概率发生死锁。有些无意义的锁会限制并发。

## 常见的并发控制实现

### Locking

[Two-phase locking](https://en.wikipedia.org/wiki/Two-phase_locking)

借助于 lock 来实现并发控制，使并发变成顺序执行。常见的 lock：

1. 读锁（Read-lock, shared lock）

2. 写锁（Write-lock, exclusive lock)）

### Timestamp ordering (TO)


### Multi-versioning

[MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)


## Postgres vs Mysql

## Django


## 资源

- https://en.wikipedia.org/wiki/Concurrency_control#Methods

- https://convincedcoder.com/2018/09/01/Optimistic-pessimistic-locking-sql/#why-locking

- https://learning-notes.mistermicheels.com/data/sql/optimistic-pessimistic-locking-sql/

- https://www.citusdata.com/blog/2018/02/22/seven-tips-for-dealing-with-postgres-locks/

- https://draveness.me/database-concurrency-control/

- https://bytes.yingw787.com/posts/2019/01/12/concurrency_with_python_threads_and_locks/

- https://www.2ndquadrant.com/en/blog/postgresql-anti-patterns-read-modify-write-cycles/

