# Purity

An important concept in functional programming is _purity_. This is the idea that a function, when called, has no effects other than transforming its input to an output. If you call the function twice, you get the same output, and it shouldn't matter how many times it's called—whether you call it once, a hundred times, or none at all, there should be no observable difference.

## Immutability

Variables, along with the values in them, should be immutable in functional languages. When a value is mutable, it means that its value depends on where in a function's evaluation you are. When variables are known to be immutable, it doesn't matter where you refer to each variable (as long as it's defined, of course), since it'll have the same value at any point. This may seem like a big restriction, but most programs don't need to mutate their values a lot, at least not after an initialization. Pretty much any case other than this can be written as a recursive function, which, once you're used to them, is often clearer.

It's not just humans who benefit from this change, but the computer does as well. It's easier to track how a value is depended on in a program if it's not changed, and operations can be safely duplicated and reordered if the result is known to not be observable. It's probably never going to reach the level of perfectly written C, but for people who like modern amenities while programming, this is a pretty nice benefit.

## Laziness

Doing work is hard. The simplest way to make your program run faster is to make it do less work. From that, a brilliant idea was born: what if we didn't do work that no one checked if we did? The answer to that question is, it makes your program run faster, and that's called laziness. Laziness allows a program to not do unnecessary work, which can be guaranteed to be safe because any observable effect must be passed as a value, which means that the value has to be evaluated, and the lazy code is guaranteed to run.

### Infinite Lazy Lists

In an eager language, if you tried to make a list of all natural numbers, you'd crash your computer (unless you had 16 GiB of RAM free to hold it or made a special container for this usage). In a lazy language, it's quite easy to do, you can just write `iterate (+1) 0` and get a list of `[0, 1, 2, 3, ...]`. The reason that this doesn't allocate an infinite amount is because if you never ask what the *n*th element is, it'll never compute it. If you try to use it in a function that expects a finite list, like `show`, `print`, or `length`, _then_ you'll have problems, but if just want to get a finite number of elements from the front of it, the rest just won't be computed.

You can also use the laziness to create self-referential lists, like with `let a = 0:a in ...`, which creates a list where the tail of the list is the list. This is fine, as long as you don't rely on the value of `a` in its definition. The `:` operator prepends the value on the left to the list on the right without actually caring what it is, so this works fine and creates an infinite list.

While eager languages need some kind of iterator to define their sequences, there's no need for a special iterator type in a lazy language. Instead, operations are implemented for lists, and any type of sequence should be convertible to a list.

### Thunks

A lazy value is referred to as a thunk, and it's implemented as a function that takes zero arguments. When the contained value of a thunk is needed, it calls that functino and replaces its value with the result, so that any future uses of it don't have to recompute the value. There are various functions that force evaluation of a thunk even if it's not used, which can help reduce the memory usage in some cases.

Take two standard functions: `foldl` and `foldl'`. Both of these are left-associative folds, but the former is lazy while the latter is eager.

```hs
-- add the numbers from 1 to 1,000,000 (inclusive)
stackOverflow = foldl (+) 0 [1..1000000] -- this creates a massive thunk that will cause a stack overflow when evaluated
worksFine = foldl' (+) 0 [1..1000000] -- this eagerly evaluates and folds it arguments and is tail recursive, so it has a constant memory usage

lazyMul :: Int -> Int -> Int
lazyMul _ 0 = 0 -- this makes our multiplication lazy in our first parameter if the second is 0
lazyMul x y = x * y

-- undefined is a value of any type that gives a runtime exception if evaluated
zeroProduct = foldl lazyMul 1 [1, 2, undefined, 0] -- this builds a thunk of lazyMul (lazyMul (lazyMul 1 2) undefined) 0, which evaluates to 0 without the inner multiplications being evaluated
runtimeError = foldl' lazyMul 1 [1, 2, undefined, 0] -- this gives a runtime exception because we have to check the undefined value when evaluating lazyMul 2 undefined
```

## Why?

Purity like an odd concept; after all, people write programs to have some effect, so trying to eliminate them seems counterintuitive. Purity has a strong advantage, though: you know exactly what can happen as a result of calling a function from the structure of your code, which also helps to enforce correct usage. Take this C code:

```c
int add(int a, int b) {
  printf("This is a side effect that you weren't thinking about!\n");
  printf("Or maybe you want to log: we called add(%d, %d)\n", a, b);
  return a + b;
}
```

From your perspective, you have no way of knowing that this was going to print, and maybe you didn't want it to do that. Let's try this in Haskell:

```hs
import Text.Printf;

add :: Int -> Int -> Int -- don't worry about the syntax too much, but this is a declaration with the same signature as what we saw in C
add a b =
  let _ignored = putStrLn "This print statement won't do anything on its own!" in
  let msg = printf "If we want to log, we could say we called add %d %d" a b in
  let _ign2 = putStrLn msg in
  a + b

-- using it
main :: IO ()
main = putStrLn $ "The result of our addition is " ++ (show $ add 2 3)
-- with parentheses instead of the $ opeartor
main = putStrLn ("The result of our addition is " ++ show (add 2 3))
```

This doesn't really look very good, and those ignored values should be a sign that something's wrong. Since functions are pure, there's rarely a use for ignoring their value, so the code for it isn't very clean. The `let ... in` syntax is used to define a variable, one that you're expected to use. Also... this code won't print anything. Each of our printing functions return an `IO ()`. Glossing over the exact details, `IO t` means an operation that _will_ return `t` (so `IO ()` is an operation that will return `()`). `()` is the unit type, meaning that there's only one valid value for it. This is similar to `None` in Python or `undefined` in JavaScript. In order to actually run an operation, you have to eventually return it from your `main` function, and anything that has effects has to return `IO` or a type containing it in order to be used.

So what if we wanted to have our print statements, and we wanted them to actually run? Well in that case, we could do this:

```hs
import Text.Printf;

add :: Int -> Int -> IO Int -- This time, we're returning an IO Int, meaning that we have some effectful operation that'll give an Int after it runs
add a b =
  putStrLn "This print statement is being returned, so it will have an effect if it makes it to main" >> -- the >> operator takes two IO values and composes them into an operation that runs the first, then the second, and returns the second's value. We'd write its signature as IO a -> IO b -> IO b
  let msg = printf "If we want to log, we could say we called add %d %d" a b in
  putStrLn msg >>
  return (a + b) -- return takes a value and wraps it in an IO, which is what the >> operator needs

main :: IO ()
main = add 2 3 >>= \res -> putStrLn $ "The result of our operation is " ++ show res -- the bind (>>=) operator lets you run one operation, then use its result for another

-- alternatively, using do-block syntax
add a b = do
  putStrLn "This print statement is being returned, so it will have an effect if it makes it to main" -- the do-block desugars these "statements" to composition
  let msg = printf "If we want to log, we could say we called add %d %d" a b
  putStrLn msg
  return (a + b)

main :: IO ()
main = do
  res <- add 2 3
  putStrLn $ "The result of our operation is " ++ show res
```

This second form almost looks like procedural code! Haskell's `do` notation can clean up a lot of the messiness that comes from monadic combinators, but still keeps our purity intact. It's clear from the signature that this function has some form of intended effect now, and that _can't_ be misused.

### What if we reeeeally wanted to have IO?

This is a bad idea, since most languages that are built around purity expect you to not try to circumvent it and therefore perform optimizations based on it. GHC, the most popular Haskell compiler, does optimizations with inlining and common subexpression elimination (CSE) that mean that your function could be called more or fewer times than what you wrote. However, there's still an escape hatch provided:

```hs
import System.IO.Unsafe;

add :: Int -> Int -> Int
add a b = (unsafePerformIO $ putStrLn "please don't do this") `seq` (a + b) -- Haskell is lazily evaluated, so you need to force evaluation with seq
```

## Exceptions

While purity is a nice tool to reason about code, it's not necessary for all functional principles. The ML family of languages, namely SML and Ocaml, only encourage purity, rather than forcing it. They use eager evaluation and procedural IO, along with having escape hatches for mutability that make them impure, but the overall design of the language still guides you toward functional code. In my opinion, this makes them more useful languages to learn, but this tutorial will be sticking with Haskell because it better expresses purely functional solutions.

---

[Functional Root](index.md) | [Previous Article (So what is functional programming, anyway?)](purity.md)
