# Specter [![Build Status](https://travis-ci.org/nathanmarz/specter.svg?branch=master)](http://travis-ci.org/nathanmarz/specter)

Most of Clojure programming involves creating, manipulating, and transforming immutable values. However, as soon as your values become more complicated than a simple map or list – like a list of maps of maps – transforming these data structures becomes extremely cumbersome. 

Specter is a library (for both Clojure and ClojureScript) for doing these queries and transformations concisely, elegantly, and efficiently. These kinds of manipulations are so common when using Clojure – and so cumbersome without Specter – that Specter is in many ways Clojure's missing piece.

Specter is fully extensible. At its core, its just a protocol for how to navigate within a data structure. By extending this protocol, you can use Specter to navigate any data structure or object you have.

Specter does not sacrifice performance to achieve its elegance. Actually, Specter is faster than the limited facilities Clojure provides for doing nested operations. For example: the Specter equivalent to get-in runs 30% faster than get-in, and the Specter equivalent to update-in runs 5x faster than update-in. In each case the Specter code is equally as convenient. 

# Latest Version

The latest release version of Specter is hosted on [Clojars](https://clojars.org):

[![Current Version](https://clojars.org/com.rpl/specter/latest-version.svg)](https://clojars.org/com.rpl/specter)

# Learn Specter

- Introductory blog post: [Functional-navigational programming in Clojure(Script) with Specter](http://nathanmarz.com/blog/functional-navigational-programming-in-clojurescript-with-sp.html)
- Presentation about Specter: [Specter: Powerful and Simple Data Structure Manipulation](https://www.youtube.com/watch?v=VTCy_DkAJGk)

Specter's API is contained in a single, well-documented file: [specter.cljx](https://github.com/nathanmarz/specter/blob/master/src/clj/com/rpl/specter.cljx)

# Questions?

You can ask questions about Specter by [opening an issue](https://github.com/nathanmarz/specter/issues?utf8=%E2%9C%93&q=is%3Aissue+label%3Aquestion+) on Github. 

# Guide

## Basics

Specter's query and transformation tools, when used at the most basic level, feel similar to Clojure's `get-in`, `assoc-in`, and `update-in`.

```clojure
user> (use 'com.rpl.specter)
user> (select [:a :b] {:a {:b 1}})
[1]
user> (setval [:a :b] "hello world" {:a {:b 1}})
{:a {:b "hello world"}}
user> (transform [:a :b] inc {:a {:b 1}})
{:a {:b 2}}
```

In each example, we are passing in a data structure and a query that finds a value deep inside that structure. In the case of `setval` and `transform`, we are also specifying how to do something to that nested value and keep the rest of the data structure intact. This is pretty simple so far.

That query is also known as a *path* which contains one or more *navigators*. A navigator doesn't just have to be a keyword. For example, the navigator `ALL` will return or update all values inside a list or a vector.

```clojure
user> (def bob {:name "Bob Jones",
                :pets [{:type :dog, :name "Rover"}
                       {:type :cat, :name "Coco"}
                       {:type :dog, :name "Max"}]})
user> (select [:pets ALL :name] bob)
["Rover" "Coco" "Max"]
user> (transform [:pets ALL :name] #(str % " Jones") bob)
{:name "Bob Jones",
 :pets [{:type :dog, :name "Rover Jones"}
        {:type :cat, :name "Coco Jones"}
        {:type :dog, :name "Max Jones"}]}
```

In this example, we navigate to the collection under the key `:pets`, then traverse that sequence with `ALL`, and for each of those values (which are pet maps) we navigate to its `:name`.

Navigators can not only expand a query but also pare it down. Let's find the names of all of Bob's dogs.

```clojure
user> (select [:pets ALL (pred #(= (:type %) :dog)) :name] bob)
["Rover" "Max"]
```

Finally, let's add the last name Jones (like the previous example) but only to the dogs:

```clojure
user> (transform [:pets ALL (pred #(= (:type %) :dog)) :name]
                 #(str % " Jones")
                 bob)
{:name "Bob Jones",
 :pets [{:type :dog, :name "Rover Jones"}
        {:type :cat, :name "Coco"}
        {:type :dog, :name "Max Jones"}]}
```

A Specter-less implementation of that problem (left as an exercise to the reader) would likely take more thought and end up a more complex solution than necessary. Specter's power over Clojure's built-in data functions comes from its vast selection of navigators that allow more succinct and creative ways to manipulate immutable data. And as we will see later, Specter is also more performant.

## Collecting

When doing more involved transformations, you often find you lose context when navigating deep within a data structure and need information "up" the data structure to perform the transformation. Specter solves this problem by allowing you to collect values during navigation to use in the transform function. Here's an example which transforms a sequence of maps by adding the value of the :b key to the value of the :a key, but only if the :a key is even:

```clojure
user> (transform [ALL (collect-one :b) :a even?]
              +
              [{:a 1 :b 3} {:a 2 :b -10} {:a 4 :b 10} {:a 3}])
[{:b 3, :a 1} {:b -10, :a -8} {:b 10, :a 14} {:a 3}]
```

The transform function receives as arguments all the collected values followed by the navigated to value. So in this case `+` receives the value of the :b key followed by the value of the :a key, and the transform is performed to :a's value. 

The four built-in ways for collecting values are `VAL`, `collect`, `collect-one`, and `putval`. `VAL` just adds whatever element it's currently on to the value list, while `collect` and `collect-one` take in a selector to navigate to the desired value. `collect` works just like `select` by finding a sequence of values, while `collect-one` expects to only navigate to a single value. Finally, `putval` adds an external value into the collected values list.


Here's how to increment the value for :a key by 10:
```clojure
user> (transform [:a (putval 10)]
              +
              {:a 1 :b 3})
{:b 3 :a 11}
```


For every map in a sequence, increment every number in :c's value if :a is even or increment :d if :a is odd:

```clojure
user> (transform [ALL (if-path [:a even?] [:c ALL] :d)]
              inc
              [{:a 2 :c [1 2] :d 4} {:a 4 :c [0 10 -1]} {:a -1 :c [1 1 1] :d 1}])
[{:c [2 3], :d 4, :a 2} {:c [1 11 0], :a 4} {:c [1 1 1], :d 2, :a -1}]
```

"Protocol paths" can be used to navigate on polymorphic data. For example, if you have two ways of storing "account" information:

```clojure
(defrecord Account [funds])
(defrecord User [account])
(defrecord Family [accounts-list])
```

You can make an "AccountPath" that dynamically chooses its path based on the type of element it is currently navigated to:


```clojure
(use 'com.rpl.specter.macros)
(defprotocolpath AccountPath [])
(extend-protocolpath AccountPath
  User :account
  Family [:accounts-list ALL])
```

Then, here is how to select all the funds out of a list of `User` and `Family`:

```clojure
user> (select [ALL AccountPath :funds]
        [(->User (->Account 50))
         (->User (->Account 51))
         (->Family [(->Account 1)
                    (->Account 2)])
         ])
[50 51 1 2]
```

The next examples demonstrate recursive navigation. Here's how to double all the even numbers in a tree:

```clojure
(defprotocolpath TreeWalker [])

(extend-protocolpath TreeWalker
  Object nil
  clojure.lang.PersistentVector [ALL TreeWalker])

(transform [TreeWalker number? even?] #(* 2 %) [:a 1 [2 [[[3]]] :e] [4 5 [6 7]]])
;; => [:a 1 [4 [[[3]]] :e] [8 5 [12 7]]]
```

Here's how to reverse the positions of all even numbers in a tree (with order based on a depth first search). This example uses conditional navigation instead of protocol paths to do the walk:

```clojure
(declarepath TreeValues)

(providepath TreeValues
  (if-path vector?
    [ALL TreeValues]
    STAY
    ))


(transform (subselect TreeValues even?)
  reverse
  [1 2 [3 [[4]] 5] [6 [7 8] 9 [[10]]]]
  )

;; => [1 10 [3 [[8]] 5] [6 [7 4] 9 [[2]]]]
```

You can make `select` and `transform` work much faster by precompiling your selectors using the `comp-paths` function. There's about a 3x speed difference between the following two invocations of transform:

```clojure
(def precompiled (comp-paths ALL :a even?))

(transform [ALL :a even?] inc structure)
(compiled-transform precompiled inc structure)
```

Depending on the details of the selector and the data being transformed, precompiling can sometimes provide more than a 10x speedup. Using Specter with precompilation generally gets the speed within a few percentage points of hand-optimized code. 

You can even precompile selectors that require parameters! For example, `keypath` can be used to navigate into a map by any arbitrary key, such as numbers, strings, or your own types. One way to use `keypath` would be to parameterize it at the time you use it, like so:

```clojure
(defn foo [data k]
  (select [(keypath k) ALL odd?] data))
```

It seems difficult to precompile the entire path because it is dependent on the argument `k` of `foo`. Specter gets around this by allowing you to precompile a path without its parameters and bind the parameters to the selector later, like so:

```clojure
(def foo-path (comp-paths keypath ALL odd?))
(defn foo [data k]
  (compiled-select (foo-path k) data))
```

This code will execute extremely efficiently. 

When `comp-paths` is used on selectors that require parameters, the result of `comp-paths` will require parameters equal to the sum of the number of parameters required by each selector. It expects to receive those parameters in the order in which the selectors were declared. This feature, called "late-bound parameterization", also works on selectors which themselves take in selector paths, such as `selected?`, `filterer`, and `transformed`.


# Future work
- Integrate Specter with other kinds of data structures, such as graphs. Desired navigations include: reduction in topological order, navigate to outgoing/incoming nodes, to a subgraph (with metadata indicating how to attach external edges on transformation), to node attributes, to node values, to specific nodes.
- Make it possible to parallelize selects/transforms
- Any connection to transducers?

# License

Copyright 2015-2016 Red Planet Labs, Inc. Specter is licensed under Apache License v2.0.
