+++
title = "Understanding Recursion"
date = "2025-12-28T19:37:22+02:00"

#
# description is optional
#
# description = "How to code in a language that doesn't support loops"

tags = []
+++

# Writing code in a functional language, or, understanding recursion

**In summary: tail call recursion**

Have you ever wondered how to write production code in Elixir or any other functional languages? Languages that don't support `while` and `for`[1] loops, `break`, `continue`, early `return` nor mutable variables. More than once, developers familiar with other paradigms, such as object-oriented or procedural, don't understand how to structure programs in a language with such limitations. This post explains how to do "real life" stuff using a functional language.

**[1]** The `for` macro in Elixir behaves differently from a `for` loop in most languages, see [the official docs](https://hexdocs.pm/elixir/comprehensions.html)

## Recursion
The definition of recursion according to Wikipedia [recursion](#recursion) is *a method of solving a computational problem where the solution depends on solutions to smaller instances of the same problem*. Mathematical *Induction* follows the same approach: you first prove the solution for the base case, and then express the general case from the base case.

To better understand how to do it in practice, this blog presents basic examples in Elixir using functional style and compares them to a procedural version implemented in Python. Note that the code might not be idiomatic Elixir because the goal is to make it readable for those unfamiliar with the language, but it serves the purpose for the explanation.

## Elixir crash course
The only things you need to know to understand the Elixir examples:
- `[head | tail]` extracts the first element and the rest of the list. For example, for `[1,2,3]` binds to `head = 1, tail = [1,2]`.
- The return value of a function is its last expression, and everything is an expression.
- You can define the same function multiple times. Elixir executes the first function clause that matches the input arguments.

For example
```ex
def foo([]), do: nil
def foo([x]), do: x
def foo(other), do: ...
```
First tries to match an empty list, then a list with a single element, and then anything else. The order in which you define the functions is important, since they evaluate from top to bottom. If you wrote the clauses in this order
```ex
def foo(other), do: ...
def foo([x]), do: x
```
the second case would never execute, because `other` matches everything.

Now, to the examples.

## First example: sum of an array
Perhaps the simplest example of a function that you can express recursively. Given an array of `elements` that support addition, write a function that returns the sum of all the elements.

The imperative-style solution in Python would look like this
```py
def sum(elements):
    acc = 0
    for el in elements:
        acc += el
    return acc
```
NdR: you can do `sum(elements)` in Python, same as `Enum.sum(elements)` in Elixir, but that's not the point of this post

For the recursive solution you need a base case and a general case. The base case occurs when `elements = []`, in that case the sum is `0`. For the general case of an array `[e1, e2, e3, ..., eN]`, you can express the sum as `e1 + sum([e2, e3, ..., eN])`. Given that the list gets 1 element shorter on each function call, it always reaches the base case.

The functional code in Elixir
```ex
def sum([]), do: 0
def sum([e1 | rest]), do: e1 + sum(rest)
```
Only two lines😎

There is one small problem with the preceding recursive solution, which is also why most non-functional languages discourage the use of recursion: if the array is too long, that code can cause a stack overflow. Thankfully you can fix this by switching to a version that uses **tail call optimization** and can call itself infinitely without exhausting the stack.

## Tail call optimization
In simple terms, *tail call* is when the last expression of a function is another function call. In simple terms, this allows to drop the stack allocations for the current function and allocate the values for the new call. Unfortunately, not all languages support tail call optimization. You should prefer the loop version over the recursive version if your language doesn't support tail-call optimization.

The `sum` function in Elixir isn't tail call optimized. For the general case, the last expression is a `+` that needs to evaluate the right-side expression. For example the array `[3,4,2]` executes to
```
sum([3, 4, 2]), do: 3 + sum([4, 2])
                        sum([4 | 2]), do: 4 + sum([2])
                                              sum([2 | []]), do: 2 + sum([])
                                                                     0
-> 3 + (4 + (2 + (0))) = 9
```

Changing the simple example to a tail-call optimized version
```ex
def sum(elements), do: sum(elements, 0)
def sum([], acc), do: acc
def sum([e1 | rest], acc), do: sum(rest, e1 + acc)
```

You can see a common approach for tail-call optimized versions, which is to add an intermediate accumulator for the general case, and returning it in the base case.

Now the previous example `[3, 4, 2]` executes to
```
sum([3, 4, 2]), do: sum([3, 4, 2], 0)
sum([3 | [4, 2]], 0), do: sum([4, 2], 3)
sum([4 | [2]], 3), do: sum([2], 7)
sum([2 | []], 7), do: sum([], 9)
sum([], 9), do: 9
```

### More complex example
Given a list of `item` containing `sku` and `price` called `whishlist` and a positive `budget`, the function must return a list of all the items' `sku` that the `budget` can afford, in order of appearance.

Python, imperative style
```py
def affordable(whishlist, budget):
    skus = []
    for item in whishlist:
        if budget < item["price"]:
            break
        budget -= item["price"]
        skus.append(item["sku"])
    return skus
```

In Elixir
```ex
def affordable(whishlist, budget), do: affordable(whishlist, budget, [])
def affordable([], _budget, skus), do: skus

# We can also destructure hash maps in Elixir by key
def affordable([%{"price" => price, "sku" => sku} | rest], budget, skus) when budget >= price, do: affordable(rest, budget - price, [sku | skus])

def affordable(_whishlist, _budget, skus), do: skus
```
1. The first clause is the entry point of the function, its only purpose is to set the `skus` accumulator to its initial value.
2. The second clause is the case where there are no more items left, returning the accumulated `skus`.
3. The third clause uses a `when` guard with the reversed logic as the `if` statement in Python. When the item price is lower than the current budget, the code sets the new budget to `budget - price`, adds the `sku` to `skus` and continues to the next item.
4. The final clause is when nothing matched, meaning the next item's `price` is higher than the current budget, thus returning the accumulated `skus`.

### Final words
Hopefully you gained some understanding on how to think recursively. It does require some initial effort to change the way you think, but after some practice it becomes easy. In case you still have issues understanding recursion you can read [this post](#) which does a better job explaining the topic.

Again, note that production Elixir code is rarely written this way. The [Enum module](https://hexdocs.pm/elixir/Enum.html) from the standard library contains many functions for dealing with enumerables (lists, hashmaps, etc.) that handle common operations such as `Enum.filter`, `Enum.count` and take the enumerable and a filter/map/reduce function.
