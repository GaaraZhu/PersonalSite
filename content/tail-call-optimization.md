---
title: "Tail call optimization in Scala"
date: 2015-10-02T15:29:15+13:00
draft: false
tags: ["Performance tuning", "Algorithm", "Scala"]
readingTime: 3
customSummary: Recursion provides us an easy way to solve some problems, and makes the code simpler and easier to follow. But one problem of it is the memory cost caused by the recursive call stack, which could be critical for memory sensitive applications like mobile ones.
---

Recursion provides us an easy way to solve some problems, and makes the code simpler and easier to follow. But one problem of it is the memory cost caused by the recursive call stack, which could be critical for memory sensitive applications like mobile ones.

An alternative is to use a while loop instead of the recursion, but while loop usually is not easy to implement and to understand in the beginning. Most times, we come up with solutions using recursion first, then try to convert it into a while loop style algorithm. But actually in some languages like Scala, it does provide the build-in support to compile the "special recursion" code to a while loop style. The "special recursion" here means that your recursion call should always be in a tail position, that is the caller does nothing other than returning the value of the recursive call.

Let's take the fibonacci number problem as an example. Normally our solution would be something like:
  
&nbsp;
```Javascript
 def fibonacci(x: Int): Int = {
    if (x <= 2) x - 1
    else fibonacci(x - 1) + fibonacci(x - 2) ------(1)
  }
```
  
&nbsp;  
&nbsp;
Here since the recursion caller at position 1 does more than just returning the recursive call result, instead there is an add operation inside, which means we need special memory to store these temporary values as well as the call stacks. But if we try to write it in a tail-recursive way, things will be different.
  
&nbsp;
```Javascript
  def fibonacci(x: Int, acc1: Int, acc2: Int): Int = {
    if (x <= 2) acc2
    else fibonacci(x - 1, acc2, acc1 + acc2)  --------(2)
  }
```
  
&nbsp;  
&nbsp;
Writing your recursion in a tail-call style, sometimes is a simple optimization for your program, helping to avoid the memory cost cased by the call stack explosion. By the way, in Scala it only works when you do the recursive call directly using the function itself, below two cases will not work:
  
&nbsp;
```Javascript
  // case 1
  val a = fibonacci _
  def fibonacci(x: Int, acc1: Int, acc2: Int): Int = {
    if (x <= 2) acc2
    else a(x - 1, acc2, acc1 + acc2)
  }
 
  // case 2
  def fibonacci2(x: Int, acc1: Int, acc2: Int): Int = {
    if (x <= 2) acc2
    else fun(x - 1, acc2, acc1 + acc2)
  }
  def fun(x: Int, acc1: Int, acc2: Int): Int = {
    fibonacci2(x, acc1, acc2)
  }
```
  
&nbsp;  
&nbsp;
