# Closures & Currying

A function is a pretty familiar concept to most programmers—being able to reuse code is pretty important for all paradigms of programming. Where functional programs differ is that functions are treated as data rather than solely for instructions. The concept of functions being able to be passed around as values is called _first-class functions_, and functions that take and return other functions are called _higher-order functions_ (HOFs). Higher-order functions are incredibly useful for defining common operations like "structure-preserving map" or "left-associative fold." Operations like these are functional, and they sneak their way into many languages because _they're really useful_. In order to have easily usable functions, we need a few extensions to the typical imperative functions.

## Closures

In old languages like C, a function can only be defined at the top level, and it can only reference other global symbols. This makes sense from the perspective of the compiled output, since a function isn't going to be defined on the fly in a local scope (unless you're JIT compiling, but that's not the solution to our problem). From a programmer's perspective, this is inconvenient, especially for the one-off operations we expect to be using with HOFs. It's not very ergonomic to define a bunch of out-of-line functions, and either pass a bunch of arguments or pack your state into a fixed-shape parameter, that requires jumping around, unclear code that's easy to mess up, and is generally uglier.

Closures solve this problem by simply letting you "capture" variables from the surrounding scope to use in your function. This typically generates a structure containing all of the referenced local variables and a function implicitly takes a pointer to that structure, but the exact layout is language-dependent and typically not the programmer's concern.

An associated pattern often (mistakenly) called a closure is a lambda expression—an anonymous inline function. Rather than having to write a bulky definition externally, like this:

```hs
isEven :: Int -> Bool
isEven v = v % 2 == 0

-- somewhere in another function
filter isEven [1, 2, 3, 4, 5]
```

It can just be defined inline:

```hs
filter (\v -> v % 2 == 0) [1, 2, 3, 4, 5]
```

## Currying

With the convenience of closures and functions being able to take and return them, we can change functions to only have one input and one output. With tuples, this was always possible—a function could take a tuple of everything it needs as an input. Curried functions, however, are far more powerful than that.

In the Haskell examples so far, you'll notice that there haven't been parentheses around any function calls. That's because two values next to each other are a function call—instead of `f(a)`, we have `f a`. In a language with functions with more than one parameter, there's an ambiguity: `f a b` could be `f(a, b)` or `f(a)(b)`. With this functional syntax, we take it to mean the latter, but also rewrite all cases of the former to be the latter.

### Example: Addition

Let's go back to a simple `add` function. You'd write it like this:

```hs
add :: Int -> Int -> Int
add a b = a + b
```

This arrow syntax might seem confusing at first, but it's really simple: `A -> B` means a function that takes an `A` and returns a `B`, like `B(A)` in C-style languages or `fn(A) -> B` in Rust. This operator is _right_-associative, so `A -> B -> C` is the same as `A -> (B -> C)`. Our two-argument function becomes a function that takes `A` and returns a function that takes `B`. To explain what happens when this is called, let's rewrite it with lambdas:

```hs
add = \a -> \b -> a + b

add 2 3 -- let's use it like this
```

Adding in parentheses for clarity:

```hs
add = \a -> (\b -> (a + b))

(add 2) 3
```

We can inline our variable:

```hs
((\a -> (\b -> (a + b))) 2) 3
```

In order to evaluate our lambda, we replace all instances of the parameter on the left side of the arrow in the expression on the right side of the arrow with the argument we pass in. Doing that, we get:

```hs
(\b -> 2 + b) 3 -- replace the outer lambda first
2 + 3 -- then the inner one
```

## Partial application

In our addition example, there was an intermediate function: `add 2`. We just glossed over it there, but we had a function that adds 2 to _any_ number passed into it. You could do this:

```hs
let f = add 2 in
  (f 1) * (f 3)
```

`f` has the type `Int -> Int`, and it's usable just like any other function. We call this a partially applied function, and with clever use of these, we can write in what's called _pointfree_ style. While lambdas are a powerful tool, getting the function you need just by composing other functions can write solutions that are more elegant and are often easier to optimize.

Going back to the [Closures](#closures) section, we had two ways of writing a check to see if a value is even, but with partial application and function composition, we can write a pointfree solution:

```hs
filter ((== 0) . (% 2)) [1, 2, 3, 4, 5]
```

Binary operators can be partially applied like this to create functions that take this missing side, e.g. `(% 2)` is the same as `\x -> x % 2`. Likewise, `(== 0)` is the same as `\x -> x == 0`. The `.` in the middle is a compositional operator, which creates a function which applies the first function to the result of the second function.

As a more complex example, take this function:

```hs
import Data.List
import Data.Char

process :: String -> [String]
process str =
  let
    strs       = words str
    trimmed    = map (\s -> dropWhileEnd isSpace $ dropWhile isSpace s) strs
    nonEmpty   = filter (\s -> not (null s)) trimmed
    uppercased = map (\s -> map toUpper s) nonEmpty
    sorted     = sortBy (\a b -> compare (length a) (length b)) uppercased
  in sorted
```

This takes a string, breaks it into words, converts them all to uppercase, and sorts by length. Rewritten in pointfree form using some standard combinators, it looks like this:

```hs
import Data.List
import Data.Char
import Data.Function

process :: String -> [String]
process xs = words xs
  & map (dropWhileEnd isSpace . dropWhile isSpace)
  & filter (not . null)
  & map (map toUpper)
  & sortBy (compare `on` length)
```

The `&` operator acts somewhat like a "pipeline," taking the value on the left and using it as the argument for the value on the right. These `filter` and `map` functions aren't explcitly meant to be used like this, but they don't need to be; their last parameter is the list they operate on, which gets passed to the function by `&`. Because of currying, neither function needs to be written specially to support the other—`filter` is just written like it would take all of its arguments inline, and `&` takes an arbitrary function.

---

[Functional Root](index.md) | [Previous Article (Purity)](purity.md)
