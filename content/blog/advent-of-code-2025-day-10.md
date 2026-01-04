+++
title = "Advent of Code 2025 Day 10"
date = "2025-12-11T15:50:42+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

# Solving advent of code 2025 day 10 with Z3

## Problem description
The inputs are an array of length `n` of non-negative integers called `joltages`, and an array of buttons where each button is an array of distinct `joltages` indexes. Each time a you press a button, it increases the value its `joltages` indexes by 1. You have to find the lowest total button presses such that starting from an array of `0` of length `n`, it reaches `joltages` values.

Example from the input:
- `joltages = [3,5,4,7]`
- `buttons = [[3], [1,3], [2], [2,3], [0,2], [0,1]]`
- Combination: press `[3]` once, `[1,3]` three times, `[2,3]` three times, `[0,2]` once and `[0,1]` twice for a total of 1 + 3 + 3 + 1 + 2 = **10**

## Solution
The solution uses the [Z3](https://en.wikipedia.org/wiki/Z3_Theorem_Prover) solver, which also supports [Integer Linear Programming](https://en.wikipedia.org/wiki/Linear_programming#Integer_unknowns), via the [Rust Z3 crate](https://docs.rs/z3/latest/z3/index.html). The approach is to define the set of constraints, the variable or equation to maximize/minimze, and let the solver do the work.

### Defining variables
The integer variable to minimize is `total_presses`, and also each button press needs its own variable `button_${i}`
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
You must specify the optimization equation and set the constraints. In this case, the goal is to minimize `total_presses`. The constraints require that all `button_${i} >=0`, this also ensures that `total_presses` is also `>=0` without having to explicitly define it.
```rust
// Imports
// use z3::Optimize;
let opt = Optimize::new();

opt.minimize(&total_presses);

// All buttons >= 0
button_presses.iter().for_each(|b| opt.assert(&b.ge(0)));
```

Here is the tricky part, for each `joltages[i] = x`, you must specify that the sum of `buttons_${idx}` that contain `i` is equal to `x`.
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
After defining all the constraints, you can check the model to ensure that there exists at least one solution and evaluate the `total_presses` variable.
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

You can find the full code [here](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day10.rs)