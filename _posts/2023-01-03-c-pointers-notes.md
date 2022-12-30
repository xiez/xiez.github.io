---
title:  C pointers notes
classes: wide
categories:
  - 2023-01
tags:
  - c&c++
type: note
---

Pointer is one of two things that many people just never really fully comprehend, another is recursion.

## What is a pointer?

A pointer is just a regular variable that happens to hold the address of another variable inside. Just that - no more, no less!

## Pointer and Array

A pointer is a variable that contains the memory address of the first value in the array.

An array is simply a label to the memory address of the first value in the array.

Take the example from [this excellent blog post](https://eli.thegreenplace.net/2009/10/21/are-pointers-and-arrays-equivalent-in-c),

```
char array_place[100] = "don't panic";
char* ptr_place = "don't panic";
```

### the difference between the two in the graphical explanation

![array.png](https://eli.thegreenplace.net/images/2009/10/array_place.png)

![pointer.png](https://eli.thegreenplace.net/images/2009/10/ptr_place.png)

### the difference between the two in accessing element (assembly code)

```
    char a = array_place[7];

0041137E  mov  al,byte ptr [_array_place+7 (417007h)]
00411383  mov  byte ptr [a],al

    char b = ptr_place[7];

00411386  mov  eax,dword ptr [_ptr_place (417064h)]
0041138B  mov  cl,byte ptr [eax+7]
0041138E  mov  byte ptr [b],cl
```

> The semantics of arrays in C dictate that the array name is the address of the first element of the array. Hence in the assignment to a, the 8th character of the array is taken by offsetting the value of array_place by 7, and moving the contents pointed to by the resulting address into the al register, and later into a.

> On the other hand, the semantics of pointers are quite different. A pointer is just a regular variable that happens to hold the address of another variable inside. Therefore, to actually compute the offset of the 8th character of the string, the CPU will first copy the value of the pointer into a register and only then increment it. This takes another instruction

## Pointer and Reference

In C++, A reference refers to an object. It is like a new temporary name for an **existing object** -- an alias.

One important thing about a reference is that, once it's created, and initialized to refer to a particular object, it will always refer to the same object.

```
#include <iostream>

// gcc v11.2
int main() {
    int a = 1;
    int b = 2;

    // Compilation error: 'refa' declared as reference but not initialized
    // int & refa;

    // OK: refa is an alias of a
    int & refa = a;

    // OK: change refa and a to b, which is 2
    refa = b;

    // --------------------
    a = 42;

    // OK: declare pa is a pointer to int, but not initialized
    int * pa;
    // &pa: address of pa is 0x7ffd05a9c180
    // pa: value of pa is 0

    // OK: assign value of pa to the address of b
    pa = & a;
    // &pa : address of pa is 0x7ffd05a9c180
    // pa: value of pa is 0x7ffc9ffe9f90
    // *pa: value of pa pointed to is 42

    return 0;
}
    
```

## Pointer and Const

A variable can be a const, and so is a pointer. Combining const and pointer can sometimes be confusing.

The source of all the confusion is our insistence on reading text from left to right. Let’s play with reversing some of the declarations. Just remember that **asterisk means "pointer to a"**.

```
Link const * pLink;  // pLink, is a, pointer to a, const, Link
Link * const pLink; // pLink, is a, const, pointer to a, Link
Link const * const pLink; // pLink, is a, const, pointer to a, const, Link
```

If a pointer is a const, that means the pointer variable can not be changed.

If the variable a pointer pointed to is a const, that means that the variable can not be changed, but the pointer can be changed to point to another variable.



## Resources

- https://eli.thegreenplace.net/2003/07/23/correct-usage-of-const-with-pointers

- https://eli.thegreenplace.net/2009/10/21/are-pointers-and-arrays-equivalent-in-c

- http://www.icodeguru.com/cpp/CppInAction/index.htm
