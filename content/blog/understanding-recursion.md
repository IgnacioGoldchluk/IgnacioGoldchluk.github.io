+++
title = "Understanding Recursion"
date = "2025-12-28T19:37:22+02:00"

#
# description is optional
#
# description = "How to code in a language that does not support loops"

tags = []
+++

# How we write code in a functional language (or, understanding recursion)

**TLDR: tail call recursion**

I program primarily using a functional language (Elixir). More than once, other developer has asked me about my job, and they were intrigued and confused about the fact that Elixir lacks several features that are *essential* to write any useful program. Elixir (as most functional languages) does not have `while` and `for`[1] loops, neither `break`, `continue`, nor early `return`, oh and also all variables are immutable!?. So how do you write code in such a *limited* language? Easy (well, simple), using [**recursion**](#recursion)!

**[1]** The `for` macro in Elixir behaves differently from a `for` loop in most languages, see [the official docs](https://hexdocs.pm/elixir/comprehensions.html)

## Recursion
According to Wikipedia [recursion](#recursion) is defined as *a method of solving a computational problem where the solution depends on solutions to smaller instances of the same problem*. It is the same reasoning as *Induction*, we first define the solution for the base case, and then express the general case from the base case.

To better understand how it is done in practice, let's do some very basic examples in Elixir and compare them to the imperative solution in Python. Note that the code might not be idiomatic Elixir because the goal is to make it readable for those unfamiliar with the language, but it serves the purpose for the explanation.

## Elixir crash course
The only things you need to know to understand the Elixir examples:
- `[head | tail]` extracts the "head" (first element) and "tail" (rest) of a list. For example if we have `[1,2,3]` then it assigns `head = 1, tail = [1,2]`
- The return value of a function is its last expression, and everything is an expression.
- Functions can be defined multiple times, and the first "matching" function signature from top to bottom is executed.

For example
```ex
def foo([]), do: nil
def foo([x]), do: x
def foo(other), do: ...
```
First tries to match an empty list, then a list with a single element, and then anything else. The order in which the function cases are defined is important, since it goes from top to bottom. If we had defined as
```ex
def foo(other), do: ...
def foo([x]), do: x
```
The second case would never execute, because `other` matches everything.

Now, to the examples.

## First Example: sum of an array
This is the most basic example I can think of that some people still struggle to think about recursively. We are given an array of `elements` that support addition, and we have to return its sum.

The imperative-style solution would look like this
```py
def sum(elements):
    acc = 0
    for el in elements:
        acc += el
    return acc
```
(Yes, I know you can do `sum(elements)` in Python, you can also do `Enum.sum(elements)` in Elixir, that's not the point of this post)

Now let's think about the recursive solution. The base case occurs when `elements = []`, in that case the sum is `0`. For the general case of an array `[e1, e2, e3, ..., eN]` we can say that the sum is `e1 + sum([e2, e3, ..., eN])`. Given that the list gets 1 element shorter on each function call, the base case is guaranteed to be reached.

The functional code in Elixir
```ex
def sum([]), do: 0
def sum([e1 | rest]), do: e1 + sum(rest)
```
Only two lines😎

There is one small problem with that recursive solution, which is also why recursion is discouraged in most languages: if the array is too long, that code can cause a stack overflow. Thankfully we can switch to a version that can call itself indefinitely with **tail call optimization**

## Tail Call Optimization
In simple terms, *tail call* is when the last expression of a function is another function call. This allows the CPU (simplifying a lot) to drop the stack allocations for the current function and allocate the values for the new call. Unfortunately, not all languages support tail call optimization, hence why loops are preferred over recursion in most languages.

Our `sum` function in Elixir is not tail call optimized because for the general case, the last expression is a `+` that needs to evaluate the right-side expression. If we take for example the array `[3,4,2]` the code would be evaluated as follows
```
sum([3, 4, 2]), do: 3 + sum([4, 2])
                        sum([4 | 2]), do: 4 + sum([2])
                                              sum([2 | []]), do: 2 + sum([])
                                                                     0
-> 3 + (4 + (2 + (0))) = 9
```

Let's change our simple example to be tail call optimized instead.

```ex
def sum(elements), do: sum(elements, 0)
def sum([], acc), do: acc
def sum([e1 | rest], acc), do: sum(rest, e1 + acc)
```
We had to slightly tweak our logic by adding an intermediate accumulator and returning it for the base case, which is a common approach when doing tail call optimization.

Now the previous example `[3, 4, 2]` would be evaluated as follows, without increasing the stack allocations
```
sum([3, 4, 2]), do: sum([3, 4, 2], 0)
sum([3 | [4, 2]], 0), do: sum([4, 2], 3)
sum([4 | [2]], 3), do: sum([2], 7)
sum([2 | []], 7), do: sum([], 9)
sum([], 9), do: 9
```

### A More Complex Example
Let's now do a more complex case: we have `item` maps with `sku` and `price`, an array of such items called `whishlist`, and a `budget`. We need to return a list of all the `sku` that we can buy with our `budget`.

In Python, imperative style
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
1. The first definition is the entry point of the function, its only purpose is to set the `skus` accumulator to its initial value.
2. The second definition is the case where we ran out of items, which is implicitly covered in Python. There are no more items to check, hence we return the `skus`.
3. The third case uses a `when` guard with the reversed logic as the `if` statement in Python. If adding the item's `price` does not go over `budget` then we set the new `budget`, prepend the item's `sku` to `skus` and move to the next item.
4. The final case is when nothing else matched, i.e. the next item would be above budget and we cannot add it, returning the `skus`.

### Final Words
Hopefully you gained some understanding on how to think recursively. It does require some initial effort to change the way you think, but after some practice it becomes easy. In case you still have issues understanding recursion I highly suggest you read [This excellent post](#) which does a better job explaining the topic.

Again, note that production Elixir code is rarely written this way. The [Enum module](https://hexdocs.pm/elixir/Enum.html) from the standard library contains many functions for dealing with enumerables (lists, hashmaps, etc.) and one does not have to reinvent the wheel every time.
