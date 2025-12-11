+++
title = "Advent of Code 2025 Day 10"
date = "2025-12-11T15:50:42+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

# Solving Advent of Code 2025 Day 10 with Z3

## Problem Description
We are given an array of length `n` of non-negative integers called `joltages`, and an array of buttons where each button is an array of unique integer values `x` where `0 <= x < n` i.e. each button is an array of indexes of `joltages`. Each time a button is pressed, it increases the value of the joltages indexes it contains by 1. We have to find the lowest total button presses such that starting from an array of `0` of length `n` we reach `joltages`.

Example from the input:
- `joltages = [3,5,4,7]`
- `buttons = [[3], [1,3], [2], [2,3], [0,2], [0,1]]`
- Combination: press `[3]` once, `[1,3]` three times, `[2,3]` three times, `[0,2]` once and `[0,1]` twice for a total of 1 + 3 + 3 + 1 + 2 = **10**

## Solution
We can use the [Z3](https://en.wikipedia.org/wiki/Z3_Theorem_Prover) solver, which also supports [Integer Linear Programming](https://en.wikipedia.org/wiki/Linear_programming#Integer_unknowns), via the [Rust Z3 crate](https://docs.rs/z3/latest/z3/index.html). The approach is to define the set of constraints, the value we want to maximize/minimze, and let the solver do the work.

### Defining variables
We have to define the integer variable `total_presses`, which is what we want to minimize, and also we have to define the integer variables button presses `button_${i}` for each button. We can do it as follows in Rust:
```rust
// Imports
// use z3::ast::Int;

// Variables in scope: buttons, joltages
let total_presses = Int::fresh_const("total_presses");

let button_presses: Vec<Int> = (0..buttons.len())
    .map(|i| Int::fresh_const(&format!("button_{i}")))
    .collect();
```

### Setting constraints
We must specify what we want to optimize for and set the constraints. In our case we want to minimize `total_presses`. We need all `button_${i} >=0`, this will ensure that `total_presses` is also `>=0` without having to explicitly define it.
```rust
// Imports
// use z3::Optimize;
let opt = Optimize::new();

opt.minimize(&total_presses);

// All buttons >= 0
button_presses.iter().for_each(|b| opt.assert(&b.ge(0)));
```

Here is the tricky part, for each `joltages[i] = x`, we must specify that the sum of `buttons_${idx}` that contain `i` is equal to `x`.
```rust
for (i, &x) in joltages.iter().enumerate() {
    // Construct terms as a vector with all the `buttons_${idx}` that
    // contain the index
    let mut terms = Vec::new();

    for (idx, btn) in buttons.iter().enumerate() {
        if btn.contains(&i) {
            terms.push(button_presses[idx].clone());
        }
    }
    let sum = Int::add(&terms.iter().collect::<Vec<&Int>>());
    opt.assert(&sum.eq(Int::from_u64(x as u64)));
}
```

And finally, `total_presses` must be equal to the sum of button presses
```rust
opt.assert(&total_presses.eq(Int::add(&button_presses)));
```

### Evaluating the model
Now that all the constraints have been defined, we can check the model to ensure there exists at least one solution and evaluate `total_presses` since it is the variable we care about
```rust
// use z3::SatResult;

match opt.check(&[]) {
    SatResult::Sat => opt
        .get_model()
        .unwrap()
        .eval(&total_presses, true)
        .and_then(|t| t.as_u64())
        .unwrap(),
    _ => panic!("No solution found"),
}
```

Full code can be found [here](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day10.rs)