---
layout: post
title: "Advent of Code 2020 Day 1: Report repair"
tags: adventofcode, coding
---

Every year, [Advent of Code](adventofcode.com) runs a series of coding challenges in the form of an advent calendar: 25 puzzles, getting progressively harder as the month goes on.
This year, I'm having a crack at it in Clojure, because Clojure is cool.

Day 1 presents a relatively simple task to warm people up.
The task is to examine lists of numbers in a given file, find the numbers that add to 2020 and return their product.
Part 2 simply extends it to triples of numbers, so I'll do that here as the implementations are the same.

I start by loading in the source file, splitting it into lines and parsing an integer from each.

```clojure
(def input (->> (io/resource "day1.txt") ;; get filename
                (slurp)                  ;; read into string
                (str/split-lines)        ;; split by lines
                (map edn/read-string)))  ;; use edn/read-string to parse int
```

Here, I use the `->>` threading macro to thread the file through the functions.

Next, I get all the combinations of the input numbers by using a `for` comprehension:

```clojure
(def combos (for [x input
                  y input
                  z input]
              [x y z]))
```

This is essentially the same as `[(x, y, z) | x <- input, y <- input, z <- input]` in Haskell, or `[(x, y, z) for x in input for y in input for z in input]` in Python.

Next I write a predicate to check if a combo is valid, and then filter the combinations to match:

```clojure
(defn valid? [[x y z]]
  (= (+ x y z) 2020))

(def valid (filter valid? combos))
```
Here you can see the use of double brackets in the function arguments `[[x y z]]` - here I am using some syntactic sugar to destructure the combo vector in place. This binds the first, second and third arguments to x, y and z respectively.

This returns a list of all valid combinations; I then need to extract the triple and multiply them together to get my answer.

```clojure
(def answer (let [[x y z] (first valid)]
              (* x y z)))
```

Day 1: sorted. Relatively straightforwards, but adjusting to Clojure took me a while!


