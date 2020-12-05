---
layout: post
title: "Advent of Code 2020 Day 2: Password Philosophy"
tags: adventofcode coding
categories: coding
---

Day 2 of Advent of Code introduced some challenges: namely, parsing.
The exercise was to read in a series of passwords from an input file (sensing a theme here) and validate them, returning the number of valid passwords.

Passwords were structured like this:

```
1-3 a: abcde
1-3 b: cdefg
2-9 c: ccccccccc
```

In line 1, the letter a must appear between 1 and 3 times in the password. 

You can find my solution [on my GitHub.](https://github.com/willosborne/aoc-2020/blob/master/src/advent_2020/day-2.clj)

## Initial parsing attempt: Kern
My philosophy when working with input data is to **parse the input into something you can work with easily.** In OOP this means building objects, in Haskell it means Algebraic Data Types, and in Clojure it usually means vectors and maps, with key information extracted in the form of `:keywords` for easy access. You shouldn't have to do any data wrangling in the core logic - separate the input processing into a different layer of abstraction. As such, I decided to go all in on parsing this year and do it *properly* - which can be quite time-consuming.

Part of the reason I chose a functional language this year was the excellent parsing support.
My initial thought was to use parser combinators, a powerful concept allowing you to define your parsers by combining simple parsers together.
For example, one might make a parser for a hexadecimal colour code by writing a parser that accepted a hash symbol, a parser that accepted a hex digit, and then combining them with combinators:
In Haskell:

```haskell
hexParser :: Parser HexCode
hexParser = do
  parseHash
  digits <- many1 parseHexDigit
  return (HexCode digits)
```

Clojure has its own parser combinator libraries, the most popular of which seems to be [kern](https://github.com/blancas/kern). However, kern just wouldn't work - I couldn't seem to find the right parts of the project when I imported it - and since I was doing this over lunch I just gave up.

## Parser magic: Instaparse

Continuing my search I found a new, extremely cool library that more than made up for the lack of combinators: [Instaparse](https://github.com/Engelberg/instaparse) by Mark Engleberg.
What Instaparse does is extremely cool: it takes a standard context-free grammar in BNF, and generates a fully functional parser, entirely from within Clojure.

For example, consider the following grammar:

```
pair = key ':' value
key = #'[a-z]+'
value = #'[0-9]+'
```

This grammar matches simple key-value pairs: `eggs:123` would be matched.

You can dump this almost verbatim into Instaparse and get a working parser out:

```clojure
(def parser (insta/parser
             "
pair = key ':' value
key = #'[a-z]+'
value = #'[0-9]+'
"))
```

Running the returned parser produces this output:

```clojure
advent-2020.day-2> (parser "egg:123")
[:pair [:key "egg"] ":" [:value "123"]]
```

How cool is that? It turns out there are way more features - you can exclude items from the output, for example, as well as using a simple but powerful transformation system to reshape that output vector into something useful within your program, such as a map.

## Onwards
Having got my parser working, I added the infrastructure to load, parse and format the input data as I wanted.

```clojure
(def password-policy-parser
  (insta/parser
   "
FILE = (LINE<'\\n'>)*
LINE = REQS<': '>PASS
<REQS> = RANGE<' '>CHAR
<RANGE> = LOWER<'-'>HIGHER
<LOWER> = #'[0-9]+'
<HIGHER> = #'[0-9]+'
<CHAR> = #'[a-z]'
<PASS> = #'[a-z]+'
"))

(defn parse-line [line]
  (let [[_ lower upper target password] line]
    {:lower (edn/read-string lower)
     :upper (edn/read-string upper)
     :target (first target)
     :password password}))

(def data (->> (io/resource "day2.txt")
              (slurp)
              (password-policy-parser)
              (rest)
              (map parse-line)))
```

The parse-line function is messy. On Day 4 I figured out how to clean this up - but it does the job for now. Note the grammar for parsing the password file, and the use of threading macros to nicely pass the data through the program.

This spits out data in a useful form that's easy to work with:

```clojure
advent-2020.day-2> (take 10 data)
({:lower 5, :upper 6, :target \s, :password "zssmssbsms"}
 {:lower 3, :upper 6, :target \j, :password "jjjjjrrj"}
 {:lower 4, :upper 7, :target \k, :password "kfkgkkkkk"}
 {:lower 2, :upper 3, :target \n, :password "nkbgfnn"}
 {:lower 7, :upper 12, :target \h, :password "hhhhhhdhhhhhfhhhh"}
 {:lower 1, :upper 4, :target \v, :password "nvvv"}
 {:lower 6, :upper 9, :target \h, :password "hhthplhgmpzsmhhxhh"}
 {:lower 6, :upper 7, :target \r, :password "rrtrrrgrgcc"}
 {:lower 10, :upper 15, :target \h, :password "sdbhvbhfjhwllmrpdv"}
 {:lower 3, :upper 4, :target \s, :password "bsss"})
```

Next, I need to count the number of occurrences of the target character in the password string. This is a bit awkward in Clojure weirdly - so I wrote my own helper function using the `frequencies` function.

```clojure
(defn count-occurrences [x xs]
  (or (get (frequencies xs) x)
      0))

(defn password-valid-1? [password]
  (let [count (count-occurrences (:target password) (:password password))]
    (and (>= count (:lower password))
         (< count (:upper password)))))
```

At this point, a solution is easy; we just filter out the invalid data items and count what's left.

```clojure
(def answer-1 (count (filter password-valid-1? data)))
```

Great success! Onwards...

## Part 2
Part 2 threw a minor spanner in the works by changing the validation criteria.
Now, instead of describing the number of characters, the requirement is that the lower and upper values represent *positions in the password where the target must appear* - the character must appear at either the lower value or the upper value, but not both. Think XOR.

This required only a new validation function. This function gets the characters at each position, checks if they match, and then does the XOR. It's messy, but I don't care.

```clojure
(defn password-valid-2? [password]
  (let [target (:target password)
        lower (get (:password password) (- (:lower password) 1))
        upper (get (:password password) (- (:upper password) 1))
        match-lower (= lower target)
        match-upper (= upper target)]
    (or (and match-lower (not match-upper))
        (and (not match-lower) match-upper))))

(def answer-2 (count (filter password-valid-2? data)))
```

And with that Day 2 is complete! Still relatively simple, but it gave me time to get to grips with parsing - an investment of time that will surely pay off as the complexity of inputs increases.
