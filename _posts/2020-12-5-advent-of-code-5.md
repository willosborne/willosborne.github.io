---
layout: post
title: "Advent of Code 2020 Day 5: Binary Boarding"
tags: adventofcode coding
categories: coding
---

Day 5 presented another fun puzzle. The problem: you've been given instructions to find your seat on a plane, but the instructions use Binary Space Partitioning. By repeatedly dividing the cabin into halves and choosing a half each time, you can go from a series of `:front` and `:back`s to a seat. You then have to do the same on each row.

As always, you can find my solution [on my GitHub.](https://github.com/willosborne/aoc-2020/blob/master/src/advent_2020/day-6.clj) This one could have been much more concise, but as always I like to do it clearly without hacks, and with proper parsing. I also missed a very elegant solution, which I'll describe at the end.

## Input
Since we have two directions to search in, it makes sense to re-use the code. As such, we need Front-Back and Left-Right to be translated into the same format. I decided to use `:left` and `:right` and work in an abstract context, on generic numbers.

We also need to extract the two parts of the input string - as it contains both. I do this using a regular expression, which will return only the parts of the string that match the relevant directions. I made a generic `match` function that took a regex and a case statement matching letters to directions.

```clojure
(defn match [func pattern]
  (fn [input]
    (->> input
         (re-find pattern)
         (map str)
         (map func))))

(def match-row (match #(case %
                         "F" :left
                         "B" :right
                         (throw (Exception. "bad input")))
                      #"[FB]+"))

(def match-col (match #(case %
                         "L" :left
                         "R" :right
                         (throw (Exception. "bad input")))
                      #"[LR]+"))

(defn parse-line [line]
  {:row (match-row line)
   :col (match-col line)})

(def input (->> (io/resource "day5.txt")
                (slurp)
                (str/split-lines)
                (map parse-line)))
```

Couple of things:
* Clojure case statements - very nifty
* Mapping to string before I match - personal preference, I don't like Clojure's notation for char literals
* re-find over re-match - we don't want to match the whole string, only find the part that is relevant to us

This produces structures like this:

```clojure
{:row [:left :left :right :left :left :right :right]
 :col [:left :right :left]}
```

## Binary searching
Now we need to run our binary space partition over that input. We need to partition the range of possible numbers at each step, incrementally narrowing it down to a single value. We start by defining a good function to get the midpoint of two numbers:

```clojure
(defn midpoint [left right]
  (+ left (int (/ (- right left) 2))))
```

Note the addition of the left value - that tripped me up at first and I was getting negative outputs!

The `binary-search` function (poorly named, but binary space partition is a pain) is a typical recursive function, written in the style used so often by the Bible - that is, [Structure and Interpretation of Computer Programs](https://mitpress.mit.edu/sites/default/files/sicp/index.html).

```clojure
(defn binary-search [left right sequence]
  (let [[x & xs] sequence]
    (case x
      nil right
      :left (recur left (midpoint left right) xs)
      :right (recur (+ 1 (midpoint left right)) right xs))))
```

* Pattern match on the head and tail of the sequence - this is essentially `(x:xs)` for Haskellers
* If we're out of input, return the upper value
* Based on the input, either recurse into the left side of the range or the right side

Note the +1 for the right hand side.

## Running the thing
We need to get the 'seat ID' for each key - that is (row * 8 + col).
Here we simply run it twice - making sure to pass in the correct range values for the binary search.

```clojure
(defn run-line [parsed-line rows cols]
  (let [row (binary-search 0 rows (:row parsed-line))
        col (binary-search 0 cols (:col parsed-line))]
    {:row row
     :col col
     :id (+ (* row 8) col)}))
```

Now we can get an answer easily - just find the largest `:id` key in the list of output.

```clojure
(def answer-1 (apply max-key :id seats))
```
Job's a good-un.

## Part 2
In Part 2 it's revealed to us that our seat isn't in the list - and we need to find it by examining the missing seats. However, seats at the start and end are missing in a contiguous block, so we can identify ours by finding a single isolated missing seat.

I'm going to do this with set operations - specifically the set difference. Find *all* the seats, then take away the seats we've found, leaving only the missing ones.

First we write a function to get all the filled seat IDs as a set:

```clojure
(defn filled-seats [seats]
  (reduce conj #{} (map :id seats)))
```

This is ugly - realise now I could have just used `(set (map :id seats))`. Reduce works just fine, though.

Next we do the set difference with the set of all seats:
```clojure
(def max-id (+ (* 127 8) 7))

(defn missing-seats [filled-seats]
  (set/difference (set (for [id (range max-id)]
                         id))
                  filled-seats))
```
Here I'm using `clojure.set/difference`, as well as a nice `for` comprehension to get all the IDs into a set.

Finally, we filter out the missing seats that are adjacent to other missing seats. For each seat, if the seats set contains another seat with an ID +/- 1, reject it.

```clojure
(defn filter-front-back [missing-seats]
  (filter (fn [seat]
            (not (or (contains? missing-seats (+ seat 1))
                     (contains? missing-seats (- seat 1)))))
          missing-seats))
```

And then run it!

```clojure
(def answer-2 (-> seats
                  filled-seats
                  missing-seats
                  filter-front-back))
```

Another day down. This one was actually a lot easier than Day 4, which is a sentiment I've found shared by a lot of friends.

## The clever solution
Matt, a close friend of mine, pointed out a solution a few hours after I happily sat back with mine that was so blindingly obvious with hindsight I felt stupid for the rest of the day.

The binary direction instructions (left, right, forward, back etc) can be converted into a binary number - e.g. `:left` is 0, `:right` is 1. When converted back to decimal this number gives the right position!
