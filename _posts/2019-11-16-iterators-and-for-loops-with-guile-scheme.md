---
layout: post
title: "Iterators and for loops with Guile Scheme"
tags: scheme lisp iterators guile emacs
---

I've been very slowly working towards procedurally generating space stations using [Guile Scheme](https://www.gnu.org/software/guile/), the GNU Project's implementation of Scheme and its official scripting language. Scheme is a minimal language by design, and I have found it lacking proper support for iterative for loops, which are very useful in procedural generation. Thanks to Scheme's incredible expressive power via both hygienic Lisp macros and first-class continuations, building this functionality into Scheme is a lot easier than you might expect.

<!-- more -->

## Inspiration

My starting point was Common Lisp's `loop` macro, an incredibly useful tool in any Lisp programmer's arsenal that defines an entire sub-language for looping control constructs. `loop` can be used like this:

```lisp
(loop for i in '(1 2 3 4) do
  (format t "Value of i: %a %~" i))
```

This code does what you'd expect:

```
Value of i: 1
Value of i: 2
Value of i: 3
Value of i: 4
```

Scheme deliberately omits macros like `loop` from its specification - including a bulky, very specific sub-language would violate the general rules of Scheme's design. Some implementations provide their own copy of the `loop` macro, while others such as Racket (not *technically* Scheme, but let's not get too pernickety) include their own custom looping constructs. For example, Racket lets you do this:

```racket
(for ([i '(1 2 3)])
    (display i))
```

This is elegant and easy to use - and while it's not a purely functional approach to programming, there's a very strong argument for the case that [Lisp is not a functional programming language.](https://letoverlambda.com/index.cl/guest/chap5.html)
Racket also provides some nice list generator functions in the style of Python's `range` function, like so:

```racket
(for ([i (in-range 3)])
    (display i))
```

My task was simple: to recreate these functions in Guile Scheme, which I find to be less opinionated and restrictive that Racket in general, and which has significantly better support for Emacs via [Geiser](https://www.nongnu.org/geiser/).

## Easy mode: for loops over lists

First things first: it's extremely easy to write a macro to iterate over every item in a list. All you need to do is `map` a lambda function containing the body of the for loop over the list, like this:

```scheme
(define-syntax for
  (syntax-rules (in)
    ((for element in list body ...)
     (map (lambda (element) 
              body ...)
            list))))
```

If you're not familiar with `syntax-rules` in Scheme, the important parts are lines 3 and 4. First, we pattern match a call that looks like `(for i in LIST body)`. That gets expanded to a call to `map`; an example macro-expansion looks like this:

```scheme
(for i in '(1 2 3)
  (display i))
  
;; expands to ... 

(map (lambda (i)
       (display i))
     '(1 2 3))
```

As you can see, this has the effect of running `body ...` on every element in the list. This is great, but limited - what if you wanted to write an iterator over the keys of a map, for example?

## More general iterators: generator functions

I'm sure there are a wide variety of clever ways of doing generic iterators. In fact, one way of doing it might be just to build a list of all the elements you want to iterate over and pass that to the `for` macro we just wrote. For example, `(in-range 4)` could return `'(0 1 2 3)`. This is a pretty heavyweight solution, however, as it forces you to evaluate the whole thing before looping (I'm sure I could get around this with some kind of lazy evaluation, but this isn't Haskell so it'd need a lot more work!)

The alternative is to have iteration be done using *generator functions* - that is, functions that return multiple times and 'yield' an answer each time, returning to where they left off when called again. I've written a simple generator function library using something called delimited continuations, which allow for new control structures to be added easily to Scheme. I'll write a little more about continuations at the end of this article, as a sort of appendix. 

An `in-range` function written with my library breaks down into two parts: the `counting` function, which is essentially the `range` function from Python enforced to take three arguments, and the `in-range` function which allows a variable number of args and fills in the details as appropriate. My `in-range` function is extremely messy, because I couldn't be bothered to figure out Guile's pattern matching mechanisms - so it just uses `car`, `cdr`, etc to decompose the arguments instead.

```scheme
;; n - starting point
;; m - goal (exclusive)
;; o - step
(define (counting n m o)
  (make-generator
   (lambda (yield)         ;; take in the yield function as a parameter from make-generator
    (let loop ((i n))      ;; body of the generator; loop over every value
        (when (< i m)      
          (yield i)        ;; yield each value one at a time
          (loop (+ i o))))
    (yield #:done))))      ;; when loop is done, conventionally yield #:done

(define (in-range . args)
  (case (length args)
    ((1) 
     ; 0 to n
     (counting 0 (car args) 1))
    ((2)
     ; n to m
     (counting (car args) (cadr args) 1))
    ((3)
     ; n to m counting o
     (counting (car args) (cadr args) (caddr args)))))
```
The meat is in the `counting` function. `make-generator` takes a lambda with a `yield` argument - this is the body of the generator, and yield can be used to temporarily suspend execution and return a value. When the generator is called next, it returns to right after the call to `yield`.

```scheme
> (define iterator (in-range 4))

> (iterator)
0

> (iterator)
1

> (iterator)
2

> (iterator)
3

> (iterator)
#:done
```

As you can see, this correctly resumes control each time the iterator is called, and returns #:done to signal that the iterator has finished. This is hacky, but makes it very clear when an iterator has exhausted its input. The internals of the `make-generator` function are complicated, using Guile's `call-with-prompt` library functions. I'll post the source at the end of the article and go through it there, but for brevity I'll skip over it for now.

## Towards a generalised `for` macro

Having written a basic `for` macro and a generators library, the next stage is adapting `for` to automatically work with my generators. This proved to be the most difficult part of the problem, and was a great exercise in learning Scheme's macros. 

The general idea of what this macro is trying to do is this:

1. Evaluate the function given to produce an iterator (e.g. `(in-range 4)` produces an iterator function)
2. Take a function body and the name of a variable (e.g. `i`) to bind the result to (e.g. `(for i in (...) body...)`)
3. Evaluate the iterator to produce a result, bind that value to `i` and execute the function body.
4. Repeat step 3 until the iterator produces the keyword value `#:done` indicating that it's exhausted its input.

My first attempts used `syntax-rules`, but I struggled with introducing a new variable into the body for the `i` in `(for i in (in-range 4) ...)`. This is tricky due to Scheme's hygienic macro system, which prevents any variables from being captured by a macro (a classic problem in Common Lisp, Clojure and others). As such, I tried to rewrite the macro in the significantly more powerful `syntax-case`, which has the full expressive power of a traditional Scheme macro while maintaining hygiene. Crucially, `syntax-case` allows the programmer to *deliberately* break hygiene when they want to.

However, it turned out that my `syntax-case` adventure was a wild goose chase all along. In fact, it's pretty simple to introduce new bindings in a `syntax-rules` macro, but I was using it incorrectly and trying to do something way more difficult.

```scheme
(define-syntax for
  (syntax-rules (in)
    ((for (i (iterator ...)) body ...)
     (let loop ((it (iterator ...))) 
         ((lambda (i) 
            (unless (eq? i #:done)
              body ...
              (loop it)))
          (it))))))
```

Here, a binding to `i` is supplied via the inner `lambda` call. This works perfectly alongside the `in-range` function, as expected:

```scheme
(for [i (in-range 10)]
  (display i)
  (newline))
0
1
2
3
4
5
6
7
8
9
```

This can even be combined with the list-based `for` macro I wrote earlier to support both types of syntax, which is messy but convenient.
If the user supplies a `for i in ...`, it uses `map` over a list; if they supply `for (i (iterator ...))` it uses the generator function-based code.

```scheme
(define-syntax for
  (syntax-rules (in)
    ((for (i (iterator ...)) body ...)
     (let loop ((it (iterator ...))) 
         ((lambda (i) 
            (unless (eq? i #:done)
              body ...
              (loop it)))
          (it))))
    ((for element in list body ...)
     (map (lambda (element) 
              body ...)
            list))))
```

## Conclusion
Lisp macros are easy; Scheme macros are (in my opinion) significantly more subtle, due in part to their much greater expressive power. In this article I've written a very simple iterator on top of a more general coroutine library. I've found this very enjoyable, and would recommend it as an exercise to someone looking to learn the features that make Scheme special.

### Appendix: First-Class Continuations and the `make-generator` function
Scheme was the first language to provide support continuations as first-class objects, which is one of the key features that set it aside from Common Lisp and most other programming languages. By supporting continuations in this way, it's possible to add almost any new control construct (such as generator functions, exceptions or cooperative threading) to the language. 

Continuations can be *really* tricky to get your head around at first. You can read a pretty decent explanation [here](https://courses.cs.washington.edu/courses/cse341/04wi/lectures/15-scheme-continuations.html), but if you're confused in any way leave a comment or contact me directly - I will gladly talk you through them and maybe write up my explanation as another post.

Despite their incredible utility, using the `call-with-current-continuation` function (which captures the continuation as an object) to build control structures can be very inefficient - to the point where it's [considered harmful](http://okmij.org/ftp/continuations/against-callcc.html) as a programming pattern by some Schemers. However, don't despair! There's another way to program with continuations that only saves the portion of the call stack that's actually useful, as delimited by 'prompts' in user code. These *delimited* continuations are composable, lightweight in memory and a whole lot faster - the only problem is there are a million implementations, as with all things Lisp. I've chosen to use Guile's inbuilt `call-with-prompt` function, which while clunky does exactly what I want. Here's my implementation of `make-generator`:

```scheme
;; import for delimited continuations
(use-modules (ice-9 control))

(define (make-generator proc)
  (define tag (make-prompt-tag))
  (define done #f)
  ;; code executed at each step; this will change to reflect the latest continuation on each tick
  ;; here, (lambda (v) ...) is the yield argument
  (define (thunk)
    (proc (lambda (v)
            (abort-to-prompt tag v))))
  (lambda ()
    ;; always return #:done if we're done here
    (if done
        #:done
        ;; call the thunk with a prompt; abort-to-prompt will return here
        ;; when it returns, the handler will be executed
        (call-with-prompt tag
          thunk
          ;; handler; update continuation and return value
          (lambda (continuation return-val)
            (cond 
             ((eq? #:done return-val) 
              (set! done #t)
              #:done)
             (else 
              (set! thunk continuation)
              return-val)))))))
```
