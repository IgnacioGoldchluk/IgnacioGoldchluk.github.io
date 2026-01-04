+++
title = "Advent of Code 2025 Day 9"
date = "2025-12-09T17:10:25+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

## Problem description
Paraphrasing the problem to more formal terms: given a list of contiguous vertices `(x, y)`, where `x` and `y` are non-negative integers, of a *simple* rectilinear polygon, find the area of the largest rectangle contained inside the polygon where 2 of the opposite corners are also vertices of the polygon.

### Observations
- The input contains 496 vertices, and therefore 122760 (496C2) possible rectangles.
- The vertices are in order, meaning that they always share either the `x` or `y` coordinate.
- The area of a point is `1`, therefore the formula for the area of the rectangle between opposite vertices `v1 = (x1, y1); v2 = (x2, y2)` is `(|x2 - x1| + 1) * (|y2 - y1| + 1)`. This means that contiguous vertices form rectangles with area != 0 and you can't skip them.
- The vertices of a rectangle formed by `v1 = (x1, y1); v2 = (x2, y2)`, being `xmin = min(x1,x2); xmax = max(x1,x2); ymin = min(y1,y2); ymax = max(y1,y2);` are `(xmin, ymin), (min, ymax), (xmax, ymin), (xmax, ymax)`

## Solution
The first part is trivial, which is to generate all possible candidate rectangles sorted by area in descending order. The second part is trickier, because you have to determine whether the rectangle formed is fully contained inside the polygon.

The "eureka moment" is to realize that you don't have to check that all the points are inside the candidate rectangle, but rather that no side of the polygon fully or partially crosses the candidate rectangle. The sides of the polygon are every pair of two contiguous vertices from the input, plus the pair formed by the first and last vertex to close the loop. There are two cases to determine whether a side crosses the rectangle:
- Fixed `x` value for an horizontal line spanning from `ylmin` to `ylmax` in the `y` axes. If `xmin < x < xmax` and not `ylmin > ymax || ylmax < ylmin` then the line crosses the rectangle and you can discard the candidate.
- Fixed `y` value for a vertical line spanning from `xlmin` to `xlmax` in the `x` axes. If `ymin < y < ymax` and not `xlmin > xmax || xlmax < xlmin` then the line crosses the rectangle and you can discard the candidate.

You can find the full code [here](https://github.com/IgnacioGoldchluk/advent-of-code-2025/blob/main/src/solutions/day9.rs)