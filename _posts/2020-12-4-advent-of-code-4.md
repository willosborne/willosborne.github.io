---
layout: post
title: "Advent of Code 2020 Day 4: Passport Processing"
tags: adventofcode coding
---

Day 4 is when things start to get interesting. This puzzle is all data validation, this time checking passports.
The first challenge asks you for a set of fields that must be present in a passport; part 2 adds more sophisticated validation to the field formats.

You can view my solution [on my GitHub.](https://github.com/willosborne/aoc-2020/blob/master/src/advent_2020/day-4.clj)

## Parsing
You know the drill: parse the input file into something we can work with. Again, I'm using Instaparse, so here's my grammar:

```clojure
(def parser (insta/parser
              "
file = passport (<'\\n\\n'> passport)* <'\\n'>
passport = field (<whitespace> field)*
whitespace = #'[ \\n]'
field = key <':'> value
key = #'[a-z]+'
value = #'[a-z0-9#]+'
"))
```

At this point it's starting to pay off - this is potentially quite complex to parse, but Instaparse makes it much easier!
Passports are sets of key-value pairs. I used regular expressions to match the values, and was careful to eliminate unnecessary output with `<`angle brackets`>`.

This data needs transforming: straight out of the parser it looks like this:

```clojure
(:file
 [:passport
  [:field [:key "byr"] [:value "1971"]]
  [:field [:key "eyr"] [:value "2039"]]
  [:field [:key "hgt"] [:value "172in"]]
  [:field [:key "pid"] [:value "170cm"]]
  [:field [:key "hcl"] [:value "17106b"]]
  [:field [:key "iyr"] [:value "2012"]]
  [:field [:key "ecl"] [:value "gry"]]
  [:field [:key "cid"] [:value "339"]]]
 [:passport
  [:field [:key "hgt"] [:value "161cm"]]
  [:field [:key "eyr"] [:value "2027"]]
  [:field [:key "ecl"] [:value "grn"]]
  [:field [:key "iyr"] [:value "2011"]]
  [:field [:key "hcl"] [:value "#a97842"]]
  [:field [:key "byr"] [:value "1977"]]
  [:field [:key "pid"] [:value "910468396"]]]
```

Usable, but not pretty. At this point I figured out how to use Instaparse's transformer - a utility that lets you provide a function for each non-terminal, describing how to transform it.
You provide this via a map, with keys matching the non-terminal names.

For example, if you had a non-terminal called "age" and were matching numbers, providing the following would automatically parse the number string into an int:

```clojure
{:age read-string}
```

My transform map looks like this:

```clojure
(def transform-options
  {:key identity
   :value identity
   :field (fn [key val]
            {:key key :value val})
   :passport (comp vec list)
   :file (comp vec list)
   })
```

Couple of tricks here:
* Providing `identity` does nothing to the key, but unwraps it - e.g. `[:key "ecl"] -> "ecl"`
* Providing `(comp vec list)` does the same thing, but with a list, by using the function composition of `vec` and `list`
* The value for `:field` takes a key and value and parses it into a nice map.

The output after transformation looks like this:

```clojure
(def data (->> (io/resource "day4.txt")
               (slurp)
               (parser)
               (insta/transform transform-options)))

[{:key "byr", :value "1971"}
  {:key "eyr", :value "2039"}
  {:key "hgt", :value "172in"}
  {:key "pid", :value "170cm"}
  {:key "hcl", :value "17106b"}
  {:key "iyr", :value "2012"}
  {:key "ecl", :value "gry"}
  {:key "cid", :value "339"}]
 [{:key "hgt", :value "161cm"}
  {:key "eyr", :value "2027"}
  {:key "ecl", :value "grn"}
  {:key "iyr", :value "2011"}
  {:key "hcl", :value "#a97842"}
  {:key "byr", :value "1977"}
  {:key "pid", :value "910468396"}]
```

Much nicer. Now we can define a list of required field, a predicate to check if a passport contains a given key, and then an overall predicate to check the validity of a passport.

```clojure
(def required-fields ["byr"
                      "iyr"
                      "eyr"
                      "hgt"
                      "hcl"
                      "ecl"
                      "pid"])

(defn passport-contains-key [pass key]
  (some #(= key %)(map :key pass)))

(defn passport-valid-1
  [passport]
  (every? #(passport-contains-key passport %) required-fields))
```

Things to notice here:
* `some` is basically a trick to check if one of the items in the list of keys satisfies the predicate - so we get all the keys with `map` and see if the given one is contained in the resulting list
* `every?` takes a list and a predicate, and returns true if the predicate is satisfied by every item in the list. Here, our list is the required keys.

That's enough for an answer:

```clojure
(def answer-1 (count (filter passport-valid-1 data)))
```

## Part 2
Part 1 was easy enough, but there's a big step up for the second part.
Now we have a variety of requirements for each field. Copying from the Advent of Code website:


* byr (Birth Year) - four digits; at least 1920 and at most 2002.
* iyr (Issue Year) - four digits; at least 2010 and at most 2020.
* eyr (Expiration Year) - four digits; at least 2020 and at most 2030.
* hgt (Height) - a number followed by either cm or in:
  * If cm, the number must be at least 150 and at most 193.
  * If in, the number must be at least 59 and at most 76.
* hcl (Hair Color) - a # followed by exactly six characters 0-9 or a-f.
* ecl (Eye Color) - exactly one of: amb blu brn gry grn hzl oth.
* pid (Passport ID) - a nine-digit number, including leading zeroes.
* cid (Country ID) - ignored, missing or not.


I decided to go about this using a map of predicates: for each key, we have a function that lets you check the format of the corresponding value.

```clojure
(def key-preds
  {"byr" (check-date 1920 2002)
   "iyr" (check-date 2010 2020)
   "eyr" (check-date 2020 2030)
   "hgt" check-height
   "hcl" check-hair
   "ecl" check-eye
   "pid" check-pid})
```

### Checking years
I noticed that 3 of the fields were almost identical, and so to reduce duplication I wrote a generator function that would produce a checker given a date range:

```clojure
(defn check-date [min max]
  (fn [value] ;; return another function that takes a value
    (and (= (count value) 4)
         (let [num (read-string value)]
           (and (>= num min)
                (<= num max))))))
```
The main thing to notice here is that we're returning a `(fn [value] ...)` - i.e. a function taking a value and checking it. In Clojure, there's no return statement - the last value in the function is returned. This is the case with all Lisps.

### Checking height
This bit was by far the most complicated. We need to parse the input, check the format is valid, and then check the range of the number based on the format.

I did this with a quick Instaparser:

```clojure
(def height-parser
  (insta/parser "height = val ('cm'|'in')
val = #'[0-9]+'"))

(def height-transform {:height (comp vec list)
                       :val read-string})
```

This takes a measurement (e.g. `150cm`) and spits out a `[height unit]` vector for easy processing.
We then check it:

```clojure
(defn check-height [input]
  (try
    (let [[height unit] (->> input
                             (height-parser)
                             (insta/transform height-transform))]
      (or (and (= unit "cm")
               (>= height 150)
               (<= height 193))
          (and (= unit "in")
               (>= height 59)
               (<= height 76))))
    (catch Exception e false)))
```

Main takeaway is to notice the `(insta/transform)` call, and the destructuring in the let statement `(let [[height unit]])`. We love syntactic sugar, right?

### Other fields
The other fields are less interesting. Either we use a simple regex, or we check from an explicit list of values.

```clojure
(defn check-hair [hair]
  (re-matches #"#[0-9a-f]{6}" hair))

(def valid-eyes ["amb"
                 "blu"
                 "brn"
                 "gry"
                 "grn"
                 "hzl"
                 "oth"])
(defn check-eye [eye]
  (some #(= eye %) valid-eyes))

(defn check-pid [pid]
  (re-matches #"[0-9]{9}" pid))
```

### Final validity check
Having got all this set up, we need to check our passports at last.

* The passport must be valid according to part 1
* For every key in the passport must have a value that returns true when passed to the corresponding predicate in `key-preds`
  * Important note: the 'cid' value is explicitly ignored in the problem statement. To cover this, we simply return true if there's no predicate for the value in the predicate map

This took a few tries - kept matching the wrong number due to minor checks (I missed the important note!) and because my regex was accepting things it shouldn't.

```clojure
(defn passport-valid-2 [passport]
  (and (every? (fn [{key :key
                     value :value}]
                 (let [pred (get key-preds key)]
                   (if pred
                     (pred value)
                     true)))
               passport)
       (passport-valid-1 passport)))
```

At this point we can shoot for an answer:

```clojure
(def answer-2 (count (filter passport-valid-2 data)))
```

Getting this one right was very satisfying! Things are beginning to heat up...
