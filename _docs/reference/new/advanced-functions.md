---
title: Functions, Closures and Recursion
---

## Introduction

Functions in xtlang (or as we will start to call them in this chapter: closures) also have types. In this chapter we will show you how to define a function (or 'Closure') type and some of the more advanced properties that functions in XTLang have, and how to use them. At the end of this chapter you should be a function ninja.

## Function Types

In xtlang, closure types are indicated by square brackets ([]), with the first element inside the brackets being the return type, and any other elements representing the type signature of the function. Closure types are always pointers:

+ `[i64]*`: a pointer to a closure which takes no arguments and returns a single i64
+ `[void]*`: A pointer to a closure which takes no arguments and returns nothing
+ `[i64,double,double]*`: a pointer to a closure that takes two double arguments and returns a single i64
+ `[void,double*,i64*]*`: a pointer to a closure that takes two pointer arguments and returns nothing.

Closures can return any valid type including another closure:

~~~~ sourceCode
(bind-func return-fun:[[i64,i64]*,i64]*
  (lambda (inc)
    (lambda (num) (+ num inc))))
~~~~

Most of the time you will not need to define the function type as Extempore's type inference engine is pretty good at working it out for you:

~~~~ sourceCode
;; Extempore works out that `inc` is an i64 because `1` is an i64
(bind-func simple-inc
  (lambda (inc)
    (+ inc 1)))

;; Here we define the type of the parameter 'num'
(bind-func simple-sum:
  (lambda (num1:i64 num2)
    (+ num1 num2)))
~~~~

Often defining the types for one or more parameters will be sufficient and you will not need to give the function a type.

### Accessing a Closure's Environment

In a previous chapter we noted that you can define an environment for a closure as part of its definition:

~~~~ sourceCode
(bind-func my-closure
  (let ((a:i32 6))
    (lambda ()
      (printf "a:%d\n" a)
      a)))

($ (my-closure)) ;; this should print 6.
~~~~

It is possible in xtlang to access that environment directly using a dot operator `closure_name.slot_name:type`. E.g. `f.a:i32` to access the value and `(f.a:i32 565)` to set it. So let's see how this works with `my-closure`:

~~~~ sourceCode
($ (my-closure.a:i32 8))
($ (my-closure)) ;; this should print 8.

($ (my-closure.a:i32)) ;; this should return 8
~~~~

This will also work for anonymous closures:

~~~~ sourceCode
(bind-func my-closure
  (lambda (x:i64)
    (lambda (y) (+ x y))))

(bind-val f [i64*,i64*]* (myclosure 4))

($ (println (f 5))) ;; prints 9

($ (f.x:i64 11))

($ (println (f 5))) ;; prints 16      

~~~~

And of course the closure's environment can also contain variables:

~~~~ sourceCode
(bind-func my-closure-func
  (let ((f (lambda (x)
    (+ x 1))))
    (lambda (x)
      (f x))))

($ (println (my-closure-func 4))) ;; prints 5

($ (my-closure-func.f (lambda (x) (* x 2))))

($ (println (my-closure-func 4))) ;; prints 8
~~~~
