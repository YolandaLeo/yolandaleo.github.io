---
layout: post
categories: 'development'
tag: Haskell
title: Basics of Haskell Learning I
---
My Haskell guide: [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters)
It is a pretty fun guide leading freshmen into the Haskell world:)

<!--more-->
It seems named functions are of highest priority when executing:
```Haskell
Prelude> succ 9 * 10
100
Prelude> succ (9 * 10)
91
```
Understanding **if**: 
    if is an expression - every expression and function must return something - else is mandatory with if.

```Haskell
doubleSmallNumber x = if x > 100 then x else x*2 + 1
doubleSmallNumber' x = (if x > 100 then x else x*2) + 1
```
``` ' ```in Haskell is valid character to use in function name: either denote a strict version of a function (one that isn't lazy) or a slightly modified version of a function or a variable.

**list**
```Haskell
*Main> let testStr = "string"
*Main> testStr
"string"
*Main> testStr ++ ['a', 'b']
"stringab"
```

Note the difference cost between concatenate a small list to a huge list, and put the small list at the beginning of the huge list:
```Haskell
ghci > [1,2,3,4,5,6,7] ++ [8]
ghci > 'A' : " SMALL CAT"
ghci > 5 : [1,2,3,4,5]
ghci > [1,2,3,4,5] !! 2
3
ghci > head [5,4,3,2,1]
5
ghci > tail [5,4,3,2,1]
[4,3,2,1]
ghci > last [5,4,3,2,1]
1
ghci > init[5,4,3,2,1]
[5,4,3,2]
Prelude> 4 `elem` [1,2,3,4]
True
Prelude> elem 4 [1,2,3,4]
True
```
When using <, <=, > and >= to compare lists, they are compared in lexicographical order.

An interesting phenomenon when going through list functions:
```Haskell
Prelude> [0.1,0.3..1]
[0.1,0.3,0.5,0.7,0.8999999999999999,1.0999999999999999]
Prelude> [0.5..1]
[0.5,1.5]
```
When I saw the example in the guide, I just thought it won't appear every time. However..it is exactly the same result in my console. After I searched in StackOverflow, I got the [answer](https://stackoverflow.com/questions/9810002/floating-point-list-generator) and tried the example [0.5..1]. 

It is quite subtle..

The application of list comprehension:
```Haskell
Prelude> let nouns = ["hobo","frog","pope"]
Prelude> let adjectives = ["lazy","grouchy","scheming"]
Prelude> [adjective ++ " " ++ noun | adjective <- adjectives, noun <- nouns]
["lazy hobo","lazy frog","lazy pope","grouchy hobo","grouchy frog","grouchy pope","scheming hobo","scheming frog","scheming pope"]
Prelude> length' xs = sum[1 | _<-xs]
Prelude> length' adjectives
3
```

**tuple**
```Haskell
Prelude> fst (8,11)
8
Prelude> snd (8,11)
11
Prelude> zip [1,2,3,4,5] [5,5,5,5,5]
[(1,5),(2,5),(3,5),(4,5),(5,5)]
Prelude> zip [1..] ["apple", "orange", "cherry", "mango"]
[(1,"apple"),(2,"orange"),(3,"cherry"),(4,"mango")]  
```
Cute functions, huh?
Let's get the comprehension and tuple functions into a simple application.
```Haskell
Prelude> let triangles = [ (a,b,c) | c <- [1..10], b <- [1..10], a <- [1..10] ]
Prelude> let rightTriangles = [ (a,b,c) | c <- [1..10], b <- [1..c], a <- [1..b], a^2 + b^2 == c^2]   
Prelude> let rightTriangles' x = [ (a,b,c) | c <- [1..10], b <- [1..c], a <- [1..b], a^2 + b^2 == c^2, a+b+c == x]  
Prelude> rightTriangles' 24
[(6,8,10)]
```
