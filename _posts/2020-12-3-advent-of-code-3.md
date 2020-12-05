---
layout: post
title: "Advent of Code 2020 Day 3: Toboggan Trajectory"
tags: adventofcode coding
categories: coding
---

Day 3 sees us riding a toboggan down a slope in the hope of catching our flight.
A pleasing puzzle - I spent most of my time wrestling with parsing though. Once parsed it was no trouble.

You can find my solution [on my GitHub.](https://github.com/willosborne/aoc-2020/blob/master/src/advent_2020/day-3.clj)

Our input is a map of rows, either empty (`.`) or with a tree (`#`). The rows tile endlessly on the X axis, and we must ride our toboggan at a constant diagonal downwards until we hit the bottom of the map, counting trees as we go.


## Parsing
This one would probably have been quite easy, but I wanted to get better at Instaparse.
Here's our grammar:

```clojure
(def parser
  (insta/parser
   "<S> = (TREE|EMPTY)+
TREE = <'#'>
EMPTY = <'.'>"))
```

This spits out a list of rows looking like this:

```clojure
([:EMPTY] [:EMPTY] [:TREE] [:EMPTY] [:EMPTY] [:TREE] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:TREE] [:EMPTY] [:EMPTY] [:EMPTY] [:TREE] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY] [:TREE] [:EMPTY] [:EMPTY] [:EMPTY] [:EMPTY])
```

Still hadn't figured out transformers at this point, so I needed a quick map to extract the individual items and splice them into one list per row. I also wrote a nifty function to print the map to the console:

```clojure
(def column (->> (io/resource "day3.txt")
                 (slurp)
                 (str/split-lines)
                 (map parser)
                 (map #(map first %)))) ;; splice lists out into top level

(defn print-grid [grid]
  (doall
   (for [line grid]
     (println (apply str (map #(case %
                                 :TREE \#
                                 :EMPTY \.) line)))))
  nil)
```

`print-grid` is a mess - but it works. The main line to pay attention to here is `(map #(map first %))` - this takes a list of lists, and extracts the elements out. It 'flattens' the list; `[[1] [2] [3]]` is mapped to `[1 2 3]`.

## Solving the puzzle
The key idea is to keep adding to X and Y by a fixed amount, *wrapping around on the X axis as you go*. This is a simple modulo operation.

```clojure
(defn get-grid-wrapped [grid x y]
  (let [w (count (first grid))
        h (count grid)
        x-index (mod x w)]
    (if (>= y h)
      :out-of-bounds
      (nth (nth grid y) x-index))))
```

My function finds the value of the given coords, wrapping on X. It helpfully returns `:out-of-bounds` if you run out of vertical space - something neat you couldn't do in Haskell! It's a little ugly, particularly the `(nth (nth ...) ...)`, but it works.

Now we need to loop through the grid, moving gradually down until we get an `:out-of-bounds`, and counting the number of `:TREE`s we hit along the way.
We're in functional land now, so we do this with recursion!

Recursion in functional programming is special: when you call a function in the *tail position* it can be optimised to a `goto` statement. In other words, it consumes no stack space and you can loop forever without blowing up your process with a stack overflow.

However, Clojure runs on the Java Virtual Machine, which can't do this by default - so it provides its own construct, `recur`.
`recur` is simple - it just recursively calls either the current function or the `loop` you're inside with the given parameters.

```clojure
(defn run
  [grid dx dy]
  (defn run-inner [x y num-trees]
    (case (get-grid-wrapped grid x y)
      :out-of-bounds num-trees
      :TREE (recur (+ x dx)
                   (+ y dy)
                   (+ num-trees 1))
      :EMPTY (recur (+ x dx)
                    (+ y dy)
                    num-trees)
      (throw "error")))
  (run-inner 0 0 0))
```

This function looks gnarly, but it's quite simple. We define an inner function to make the top-level signature nicer, and to allow us to specify the slope on which we're moving (e.g. 3 along, 1 down).

We then use a `case` statement to look at the current square.
If it's `:out-of-bounds` we just return our count, and otherwise recursively call the function with our modified values. That way we build up the x, y and number of trees as it loops.

At this point we're pretty much done!

```clojure
(def answer-1 (run column 3 1))
```

## Part 2
Part 2 adds very little: we just try it with varying slopes and find the product of the answers. Easy!

```clojure
(def answer-2 (* (run column 1 1)
                 (run column 3 1)
                 (run column 5 1)
                 (run column 7 1)
                 (run column 1 2)))
```

See you tomorrow!
