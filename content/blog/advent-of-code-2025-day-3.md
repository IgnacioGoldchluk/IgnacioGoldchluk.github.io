+++
title = "Advent of Code 2025 Day 3 using recursion"
date = "2025-12-03T12:32:22+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

# Problem description
Given an array of digits 1-9 `bank`, and a number of batteries `n`, where `length(bank) >= n`, find the maximum number that you can construct by selecting `n` digits from `bank` in order.

Examples
| bank | batteries | value |
| --- | --- | --- |
| 5635 | 2 | 65 |
| 333 | 2 | 33 |
| 9819 | 3 | 989 |

## Recursive solution
A good approach to find a recursive solution is to think of the base case and some concrete cases:
- `n = 0` the result is `0`, because you can't choose any battery
- `n = 1` you can simply find the maximum digit in the entire array
- `n = 2` gets a bit tricky. You have to find the maximum digit but you can't go through the entire array. For example, when `bank = 689, n = 2` the maximum value is `89`, but if you select the maximum digit, you would start with `9`. You have to find the maximum digit leaving at least 1 digit left for the second battery, and then select the maximum digit to the right.
- `n = 3` is similar to `n = 2`, you have to find the maximum value and leave at least 2 digits left for the other 2 batteries.

Another thing to keep in mind is which maximum value to choose from. Consider `bank = 8781, n = 2`, where the result should be `88`. On the first iteration the maximum digit is `8` and you have two options:
- If you choose the first `8` then you move to `bank = 781, n = 1`, obtaining `88`. 
- If you choose the second `8` then you move to `bank = 1, n = 1`, obtaining `81`.

Therefore, you always have to choose the first occurrence of the maximum digit.

The algorithm then would be the following
1. If `n = 0` return `0`.
2. If `n = 1` return the maximum digit.
3. If `n > 1` select the maximum digit from the array excluding the last `n - 1` digits.
4. Call the function again with the remainder of the `bank` array and `n = n - 1`.

For accumulating the result there are three choices:
- Create an array and convert all the digits to an integer once you reach the base case
- Multiply the current accumulator by 10 and pass it to the function call
- **Non Tail-Call Optimized**: multiply the current max digit by `10 ** (n - 1)` and add the result of the next function call

Since [the solution is in Rust](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day3.rs), which doesn't have stable tail-call optimization yet, it uses the third option, however it's trivial to convert it to a tail-call optimized case by adding an accumulator as an argument.