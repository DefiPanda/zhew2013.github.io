---
layout: post
title: "Python multiple assignment is a puzzle"
date: 2014-07-26 19:12:16 -0700
comments: true
categories: 
---
This afternoon I was working on a coding challenge on an online judge for finding the first missing positive from an array. For example, the first missing positive of [4,2,5,7,1] is 3. The problem is pretty straightford and solvable in O(n) time and O(1) space. So without much thought I wrote out the first iteration of code:

```
def firstMissingPositive(A):
    i = 0
    while i < len(A):
        if A[i] > 0 and A[i] - 1 < len(A) and A[A[i] - 1] != A[i]:
            A[i], A[A[i] - 1] = A[A[i] - 1], A[i]
        else:
            i += 1
    for i in range(len(A)):
        if A[i] != i + 1:
            return i + 1
    return len(A) + 1
```

To my surprise, the above code runs forever for input [2,1]. I did a bit debugging and found out `A[i], A[A[i] - 1] = A[A[i] - 1], A[i]` never swap the value correctly. That line is supposed to make `[2,1]` into `[1,2]` in the first iteration of while loop. But instead, after execution of that line the array `A` is still `[2,1]`.

Totally puzzled, I changed `A[i], A[A[i] - 1] = A[A[i] - 1], A[i]` into `A[A[i] - 1], A[i] = A[i], A[A[i] - 1]` instead. Guess what? The modified code passed all test cases.

Unbelieved by the result, I was so intrigued and dicided to find out why.

So I first wrote a simple function as follows:

```
# This one will fail
def test1():
    A = [2,1]
    A[0], A[A[0] - 1] = A[A[0] - 1], A[0]
    return A
```

CPython has a disassembler library to turn Python bytecode into human readable strings (https://docs.python.org/2/library/dis.html). So I use that to transform the bytecode of above code to following:

```
  0 LOAD_CONST               1 (2)
  3 LOAD_CONST               2 (1)
  6 BUILD_LIST               2
  9 STORE_FAST               0 (A)   # A = [2, 1]

 12 LOAD_FAST                0 (A)
 15 LOAD_FAST                0 (A)
 18 LOAD_CONST               3 (0)
 21 BINARY_SUBSCR       
 22 LOAD_CONST               2 (1)
 25 BINARY_SUBTRACT     
 26 BINARY_SUBSCR                    # A[A[0] - 1]
 27 LOAD_FAST                0 (A)
 30 LOAD_CONST               3 (0)
 33 BINARY_SUBSCR                    # A[0]
 34 ROT_TWO                          # Swap top two stack elements, 
                                     # now 1 (A[A[0] - 1] = A[2 - 1] = 1) is at top, 
                                     # and 2 (A[0] = 2) is at the second of top
 35 LOAD_FAST                0 (A)
 38 LOAD_CONST               3 (0)
 41 STORE_SUBSCR                     # Store top stack element 1 to A[0], 
                                     # now A = [1, 1], and 1 is removed 
                                     # from stack and 2 is at the top of stack
 42 LOAD_FAST                0 (A)
 45 LOAD_FAST                0 (A)
 48 LOAD_CONST               3 (0)
 51 BINARY_SUBSCR                    # Load A[0], notice now A[0] == 1
 52 LOAD_CONST               2 (1)
 55 BINARY_SUBTRACT                  # Load A[A[0] - 1] = A[1 - 1] = A[0]
 56 STORE_SUBSCR                     # Store top stack element 2 to A[0], 
                                     # now A = [2, 1]. We are resetting A[0] back to 2!

 57 LOAD_FAST                0 (A)
 60 RETURN_VALUE                     # Return A
```    

The key insight on what's going on inside `A[0], A[A[0] - 1] = A[A[0] - 1], A[0]` is the indices on the left side (`0` of `A[0]` and `A[0] - 1` of `A[A[0] - 1]`) is evaluated NOT before the assignment of the value, but in the process of multiple assignment. We eval the right-hand side values first (`A[A[0] - 1]` => 1, `A[0]` = 2), then swap the top 2 elements in stack, assign the evaluated values stored in the stack (2 on top with 1 belows it) to the left-hand variables, from left to right. So A[0] on left is expectedly assigned to 1. But `A[A[0] - 1]` is equal to `A[1 - 1]` = `A[0]` now, so instead of assigning 2 to `A[1]`, we assign it to `A[0]`.

Notice if all left-hand side incides are evaluated before multiple assignment, `A[A[0] - 1]` will always be the same as `A[2 - 1]` = `A[1]` in the multiple assignment process, and we will get the desired result.

So how to solve this problem? Easy, if we don't want the later read index to be written upon first, we can just make a copy of index first:

```
# Success! If we assign index to a variable
def test2():
    A = [2,1]
    x = A[0] - 1
    A[0], A[x] = A[x], A[0]
    return A
```

Or, if one index depend on a value that can possibly be changed through the multiple assignment process, just assign the dependent value (`A[A[0] - 1]`) first before the assignment of the value it depends on (`A[0]`).

```
# Also suceeded if we change the swapping order
def test3():
    A = [2,1]
    A[A[0] - 1], A[0] = A[0], A[A[0] - 1]
    return A
```

Sometimes the language implementation of variable evaluation order is not in the way you expected, but it is quite an interesting lesson at least for me to learn.

Discuss it on HN:  
https://news.ycombinator.com/item?id=8091943