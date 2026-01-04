+++
title = "Advent of Code 2025 Day 2 in under 200us"
date = "2025-12-02T15:02:55+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

# Optimizing advent of code 2025 day 2

## Disclaimer
- [Here](https://adventofcode.com/2025/day/2) is the link to the full Advent of Code 2025 Day 2 puzzle. To make it short, and also because the event creator requested in the [About page](https://adventofcode.com/2025/about) to not paste significant parts of the problem statement, the focus is going to be on the invalid ID generation part.

## Problem description
Given a list of integer ranges `[start, end]`, obtain all the distinct *invalid IDs* contained in each range.

- Part 1 defines "invalid ID" as any number constructed by a sequence of digits repeated exactly twice. For example `123123, 11, 4004` are invalid IDs, but `111, 12321` aren't.
- Part 2 redefines "invalid ID" as any number constructed by a sequence of digits repeated any number of times. For example `121212, 333, 456745674567` are invalid IDs, but `12321, 9889` aren't. Part 1 then becomes a special case of Part 2 where `repetitions=2`

## Solution
The solution is to generate **only** invalid IDs instead of iterating over the entire range and check whether the number is an invalid ID. The algorithm is as follows:
1. For a given `[start, end]` range and `repetitions` number, find the first `seed` number that generates an invalid ID `>= start`.
2. If the generated invalid ID is `<= end`, add it to the invalid IDs list.
3. Set `seed = seed + 1` and generate the next invalid ID.
4. Repeat 2 and 3.

### Generating the seed
Ideally you want to make an initial `guess` for the `seed` value close to the correct one, and increase it as few times as possible until you find the first invalid ID. How do you get the initial invalid ID value given `start` and `repetitions`? Looking at some values from the problem statement example:

| start | repetitions | first invalid ID |
| --- | --- | --- |
| 11 | 2 | 11 |
| 998 | 2 | 1010 |
| 998 | 3 | 999 |

If you pay close attention, there are two different cases
- When the number of digits of `start` is divisible by `repetitions`, you can set the initial `guess` to the first `digits_start/repetitions` digits. For example, in `start=1256, repetitions=2` you set `guess=12`.
- When the number of digits of `start` isn't divisible by `repetitions`, you can set `guess = 10 ** (digits_start / repetitions)`. For example, in `start=998, repetitions=2` you set `guess=10` because `3` isn't divisible by `2`.

### Generating an invalid ID
In order to generate an invalid ID for the given seed you have two options.

#### Using strings
This is the easiest option, you have to convert `seed` to a string, repeat it `repetitions` times and convert it back to an integer.

#### Using integers only
You can generate the invalid ID without strings, using integer arithmetic.

Guessing the formula from two examples:
- `seed=12, repetitions=3`, the solution is`121212`, which is possible to express as `seed + seed * (10 ** 2) + seed * (10 ** 4)`.
- `seed=123, repetitions=2`, the solution is `123123`, which is possible to express as `seed + seed * (10**3)`.

You want `repetitions` terms, and each exponent should shift `seed` as many digits as `seed` contains. You can obtain the digits of a number via `log10(num) + 1`, then you can define the number of digits as `step`, and finally the formula is `seed * (10 **(step*0) + 10**(step*1) + ... + 10**(step * (repetitions-1))`  

## Results
You can find the complete solution [here](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day2.rs).
