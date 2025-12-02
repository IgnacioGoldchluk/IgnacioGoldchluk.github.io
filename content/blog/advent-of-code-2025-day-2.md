+++
title = "Advent of Code 2025 Day 2 in under 200us"
date = "2025-12-02T15:02:55+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

# Optimizing Advent of Code 2025 Day 2

## Disclaimer
- [Here](https://adventofcode.com/2025/day/2) is the link to the full Advent of Code 2025 Day 2 puzzle. To make it short, and also because the event creator requested in the [About page](https://adventofcode.com/2025/about) to not paste significant parts of the problem statement, the focus is going to be on the invalid ID generation part.
- Also, **SPOILER ALERT**.

## Problem Description
We are given a list of integer ranges `[start, end]` and we have to obtain all the distinct *invalid IDs* contained in each range.

- Part 1 defines "invalid ID" as any number constructed by a sequence of digits repeated exactly twice. For example `123123, 11, 4004` are invalid IDs, but `111, 12321` are not.
- Part 2 redefines "invalid ID" as any number constructed by a sequence of digits repeated any number of times. For example `121212, 333, 456745674567` are invalid IDs, but `12321, 9889` are not. Part 1 then becomes a special case of Part 2 where `repetitions=2`

## Solution

We can generate **only** invalid IDs instead of iterating over the entire range and check whether the number is an invalid ID. The algorithm is as follows:
1. For a given `[start, end]` range and `repetitions` number, find the first `seed` number that generates an invalid ID `>= start`.
2. If the generated invalid ID is `<= end`, add it to the invalid IDs list.
3. Set `seed = seed + 1` and generate the next invalid ID.
4. Repeat 2 and 3.

### Generating the seed
We want to make an initial `guess` for the `seed` value close to the correct one, and increase it as few times as possible until we get our first invalid ID, but how can we get the initial invalid ID value given `start` and `repetitions`? Let's look at some values from the problem statement example:

| start | repetitions | first invalid ID |
| --- | --- | --- |
| 11 | 2 | 11 |
| 998 | 2 | 1010 |
| 998 | 3 | 999 |

If you pay close attention we have two different cases
- When the number of digits of `start` is divisible by `repetitions`, we can set our initial `guess` to the first `digits_start/repetitions` digits. For example in `start=1256, repetitions=2` we would take `12`.
- When the number of digits of `start` is not divisible by `repetitions`, we can set `guess = 10 ** (digits_start / repetitions)`. For example in `start=998, repetitions=2` we set `guess=10` because `3` is not divisible by `2`.

### Generating an invalid ID
In order to generate an invalid ID for the given seed we have two options.

#### Using strings
The easiest option, we have to convert `seed` to a string, repeat it `repetitions` times and convert it back to an integer. While this approach is fast, when I measured my solution in Rust took between 10ms and 15ms.

#### Using integers only
We can generate the invalid ID without strings, using only integer arithmetic.

Let's try to guess the formula with some examples:
- `seed=12, repetitions=3`, we want to get `121212`, which can be represented as `seed + seed * (10 ** 2) + seed * (10 ** 4)`.
- `seed=123, repetitions=2`, we want to get `123123`, which can be represented as `seed + seed * (10**3)`.

It is clear that we want `repetitions` terms, and each exponent should shift `seed` as many digits as `seed` contains. The digits of a number can be obtained via `log10(num) + 1`, we can define the number of digits as `step`, and the formula is `seed * (10 **(step*0) + 10**(step*1) + ... + 10**(step * (repetitions-1))`  

## Results
The complete solution is [here](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day2.rs). It takes ~200us to run *on my machine* using Rust release build.
