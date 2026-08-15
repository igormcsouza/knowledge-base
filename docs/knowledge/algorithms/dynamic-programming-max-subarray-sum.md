---
tags:

- algorithms
- dynamic-programming
- computer-science

---

# Dynamic Programming: Maximum Subarray Sum

Dynamic Programming (DP) is a technique for solving problems by breaking them
into smaller overlapping subproblems, solving each subproblem once, and
reusing those results instead of recomputing them. It applies when a problem
has two properties:

- **Optimal substructure** — the optimal solution to the problem can be built
  from optimal solutions to its subproblems.
- **Overlapping subproblems** — naively solving it recursively would solve
  the same subproblem many times.

Two ways to implement DP: **memoization** (recursive, cache results of calls
you've already made) and **tabulation** (iterative, build a table of results
bottom-up). The problem below is a good first DP problem to fully internalize
because the tabulated version collapses into a single-pass, constant-space
algorithm.

## The Problem

Given an array of integers (which can include negative numbers), find the
**contiguous subarray** with the largest sum, and return that sum.

```text
nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
answer = 6   # from the subarray [4, -1, 2, 1]
```

Note this is about a *contiguous* subarray — not to be confused with the
"maximum sum (non-contiguous) subsequence" problem, which is trivial (sum of
all positive numbers).

## Brute Force

Try every possible subarray, sum each one, keep the max:

```python
def max_subarray_brute(nums: list[int]) -> int:
    best = float("-inf")
    for i in range(len(nums)):
        for j in range(i, len(nums)):
            best = max(best, sum(nums[i : j + 1]))
    return best
```

This is O(n³) as written (O(n²) subarrays × O(n) to sum each), or O(n²) if you
accumulate the sum incrementally instead of re-summing each slice. Either
way, it recomputes overlapping work: the sum of `nums[i:j]` and
`nums[i:j+1]` share almost everything.

## The DP Formulation

Define `dp[i]` = the maximum subarray sum **ending exactly at index i** (not
just anywhere in `nums[0..i]`). At each position, there are only two options
for the best subarray ending there:

1. Extend the best subarray that ended at `i - 1` by including `nums[i]`.
1. Start fresh at `i` — abandon whatever came before, because it was net
   negative and would only drag the sum down.

```text
dp[i] = max(nums[i], dp[i - 1] + nums[i])
```

The answer to the whole problem is `max(dp)` — the best subarray could end
anywhere, so we take the best over all ending points.

## Kadane's Algorithm

Because `dp[i]` only ever depends on `dp[i - 1]`, there's no need to keep the
whole `dp` array — just carry the previous value forward. This reduction from
an O(n)-space table to O(1) space is exactly what "tabulation" collapses into
once you notice the recurrence only looks one step back. This specific
algorithm is known as **Kadane's Algorithm**:

```python
def max_subarray(nums: list[int]) -> int:
    best_ending_here = best_so_far = nums[0]

    for num in nums[1:]:
        best_ending_here = max(num, best_ending_here + num)
        best_so_far = max(best_so_far, best_ending_here)

    return best_so_far
```

**Complexity**: O(n) time, O(1) space — a single pass, no extra array.

## Walkthrough

For `nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`:

| i | nums[i] | best_ending_here | best_so_far |
|---|---|---|---|
| 0 | -2 | -2 | -2 |
| 1 | 1 | max(1, -2+1)=1 | 1 |
| 2 | -3 | max(-3, 1-3)=-2 | 1 |
| 3 | 4 | max(4, -2+4)=4 | 4 |
| 4 | -1 | max(-1, 4-1)=3 | 4 |
| 5 | 2 | max(2, 3+2)=5 | 5 |
| 6 | 1 | max(1, 5+1)=6 | 6 |
| 7 | -5 | max(-5, 6-5)=1 | 6 |
| 8 | 4 | max(4, 1+4)=5 | 6 |

Final answer: **6**, matching the subarray `[4, -1, 2, 1]` (indices 3–6).

!!! tip "Pro Tip"
    Whenever a recurrence like `dp[i] = f(dp[i-1], ...)` only looks a fixed,
    small number of steps back, that's a strong signal you can drop the full
    DP array and just keep a handful of rolling variables — turning an O(n)
    space solution into O(1). Worth checking on every DP problem before
    settling for the array-based version.

## Summary

- DP applies when a problem has optimal substructure + overlapping
  subproblems.
- Define the subproblem carefully: here, "best subarray *ending at* i,"
  not "best subarray in the first i elements" — that distinction is what
  makes the recurrence work.
- The recurrence `dp[i] = max(nums[i], dp[i-1] + nums[i])` encodes "extend or
  restart" at every position.
- Kadane's Algorithm is this DP solved in O(n) time, O(1) space by only
  carrying the previous state forward.

## Related Articles

- [Python Tips & Tricks](../python/python-tips.md) — general Python
  patterns useful while implementing algorithms like this.
