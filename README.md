# house robber python: solve LeetCode 198 three ways, from 2ⁿ recursion to O(1) space

You open LeetCode 198, read the paragraph about a robber and connected alarm systems, and think: obviously I just take every other house. Then `[2, 7, 9, 3, 1]` returns 10 instead of 12 and the whole thing falls apart. Most people searching for a house robber Python solution land in one of two spots: they have a loop that fails on some hidden test, or they copied a four-line solution they can't explain out loud. This walks through both.

## What LeetCode 198 is actually asking

Restated in plain terms: given a list of non-negative integers, pick a subset of indices such that no two picked indices are adjacent, and maximize the sum.

- Input: `nums = [1,2,3,1]` → output `4` (rob house 0 and house 2)
- Input: `nums = [2,7,9,3,1]` → output `12` (rob houses 0, 2, 4: 2 + 9 + 1)
- Constraints on LeetCode: `1 <= nums.length <= 100`, `0 <= nums[i] <= 400`

The keyword in that constraint list is "adjacent." Not "sorted," not "contiguous." Order of houses stays fixed; you just can't take two neighbors.

## Why the obvious greedy answer is wrong

Two greedy instincts both fail here.

The first is "rob every other house starting from index 0, then starting from index 1, take the larger." Try `[2, 12, 9, 3, 4]`. Odd-index houses give 12 + 3 = 15, even-index give 2 + 9 + 4 = 15. Both 15. The real answer is 16 (12 + 4), because you can skip a house you don't want to skip. Alternating parity isn't a constraint anywhere in the problem.

The second is "grab the biggest house first, then work around it." Largest is 9, so you're forced to skip 7 and 3, and you can take 1 → total 10. The optimum is 2 + 9 + 1 = 12. Taking the locally biggest value never pays to extend globally.

That failure is the entire reason this problem is filed under dynamic programming. It also shows up in AlgoMonster's own discussion thread for House Robber, where a reader posted an odd/even summing solution that passed their test cases, and other readers pointed out LeetCode's tests kill it immediately. Thin test coverage will make a wrong solution look correct, which is worth remembering before you trust a green checkmark.

## The state and recurrence that make house robber click

Stop thinking about which houses to rob. Think about one house at a time and what your best total is if the street ends there.

Define `dp[i]` as the maximum amount you can steal considering houses `0..i`. At house `i` you have exactly two moves:

1. Rob house `i`. Then house `i-1` is off limits, so you add `nums[i]` to the best result from `0..i-2`.
2. Skip house `i`. Your total stays whatever it was for `0..i-1`.

So:


dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])


Base cases: `dp[0] = nums[0]`, `dp[1] = max(nums[0], nums[1])`.

That dependence on the previous two entries is the same shape as climbing stairs or Fibonacci, just with a `max` instead of a `+`. If you've internalized one, you've internalized the other.

## Solution 1: plain recursion

Start here to prove you understand the decision, then throw it away.

python
class Solution:
    def rob(self, nums: list[int]) -> int:
        def dfs(i: int) -> int:
            if i >= len(nums):
                return 0
            take = nums[i] + dfs(i + 2)
            skip = dfs(i + 1)
            return max(take, skip)

        return dfs(0)


Time is O(2ⁿ) because each house spawns two branches, and `dfs(i)` gets recomputed from both `i-1` and `i-2`. Space is O(n) for the call stack. LeetCode's `n <= 100` will time out on the worst cases.

## Solution 2: top-down DP with memoization

Same recursion, plus a cache. `functools.lru_cache` turns exponential into linear in one decorator.

python
from functools import lru_cache

class Solution:
    def rob(self, nums: list[int]) -> int:
        @lru_cache(maxsize=None)
        def dfs(i: int) -> int:
            if i >= len(nums):
                return 0
            return max(nums[i] + dfs(i + 2), dfs(i + 1))

        return dfs(0)


Two things bite people here.

- `lru_cache` requires hashable arguments. Memoize on the index, not on the list. `dfs(nums[i:])` looks elegant and raises `TypeError: unhashable type: 'list'`.
- The default cache size is 128 entries. At `n <= 100` you're fine either way, but `maxsize=None` removes the question entirely.

Time is O(n), space is O(n) for the cache plus recursion stack.

## Solution 3: bottom-up array

This is the version to write on a whiteboard if an interviewer asks you to walk through the table.

python
class Solution:
    def rob(self, nums: list[int]) -> int:
        n = len(nums)
        if n == 1:
            return nums[0]

        dp = [0] * n
        dp[0] = nums[0]
        dp[1] = max(nums[0], nums[1])

        for i in range(2, n):
            dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])

        return dp[-1]


O(n) time, O(n) space. Slower to write than the memoized version, but every intermediate value is visible, which makes "why is this O(n) and not O(2ⁿ)" a two-second answer.

## Solution 4: two variables, O(1) space

`dp[i]` only ever reads `dp[i-1]` and `dp[i-2]`, so the whole array is unnecessary. Keep two scalars.

python
class Solution:
    def rob(self, nums: list[int]) -> int:
        prev2 = prev1 = 0   # best total up to i-2 and i-1
        for money in nums:
            prev2, prev1 = prev1, max(prev1, prev2 + money)
        return prev1


Read the tuple assignment carefully, because it's where people misread the code. The right side is fully evaluated first, so `prev2` receives the old `prev1` and `prev1` receives the new maximum in the same statement. Writing it as two separate lines in the wrong order silently corrupts both states.

A variation you'll see in LeetCode's Python editorials is functionally identical:

python
def rob(self, nums):
    dp1 = dp2 = 0
    for num in nums:
        dp1, dp2 = dp2, max(dp1 + num, dp2)
    return dp2


Two variables, O(n) time, O(1) space, and it handles an empty input by returning 0. That last property matters in the next section.

## Tracing [2, 7, 9, 3, 1] by hand

| i | nums[i] | dp[i-1] | dp[i-2] + nums[i] | dp[i] |
| --- | --- | --- | --- | --- |
| 0 | 2 | — | — | 2 |
| 1 | 7 | 2 | — | 7 |
| 2 | 9 | 7 | 2 + 9 = 11 | 11 |
| 3 | 3 | 11 | 7 + 3 = 10 | 11 |
| 4 | 1 | 11 | 11 + 1 = 12 | 12 |

Notice row 3: a house with 3 dollars is skipped even though 3 > 0, because taking it forces you to give up the 9. The `max` in the recurrence is what encodes that trade, and tracking it out loud in an interview is usually what earns the "they actually understand their solution" credit.

## Complexity summary

| Approach | Time | Space | When to use |
| --- | --- | --- | --- |
| Plain recursion | O(2ⁿ) | O(n) stack | Explaining the decision, never submitting |
| Memoization | O(n) | O(n) | Fast to write under time pressure |
| Bottom-up array | O(n) | O(n) | Whiteboard explanations, House Robber II helpers |
| Two variables | O(n) | O(1) | The answer most interviewers want |

## Edge cases and the traps that show up in tests

- **Single house.** `nums = [5]` must return 5, not 0. The bottom-up version needs the explicit `if n == 1` guard; the two-variable version doesn't.
- **Two houses.** `[4, 9]` returns 9. No third house exists to change the decision.
- **All zeros.** `[0, 0, 0, 0]` returns 0. The recurrence handles it without special-casing.
- **Empty list.** LeetCode guarantees at least one house in 198, but the empty case appears the moment you write a shared helper for House Robber II. Make that helper return 0 for an empty slice.
- **In-place mutation.** None of these solutions modify `nums`. If you find yourself doing `nums[i-1] = 0` to "mark" a robbed house, you've turned a DP problem into a state-tracking problem and broken the invariant.
- **Value ranges.** `nums[i] <= 400` and `n <= 100` cap the answer around 20,000, so nothing overflows in any language. Python's arbitrary-precision ints make it a non-issue regardless.

## House Robber II and III reuse this exact code

House Robber II (LeetCode 213) puts the houses in a circle, so the first and last are now adjacent. The trick is that you can't rob both ends, so the answer is the better of two linear runs:

python
class Solution:
    def rob(self, nums: list[int]) -> int:
        if len(nums) == 1:
            return nums[0]

        def linear(houses: list[int]) -> int:
            prev2 = prev1 = 0
            for money in houses:
                prev2, prev1 = prev1, max(prev1, prev2 + money)
            return prev1

        return max(linear(nums[:-1]), linear(nums[1:]))


Same helper, two calls, and the `len(nums) == 1` guard exists because `nums[:-1]` and `nums[1:]` are both empty otherwise.

House Robber III (LeetCode 337) moves the same take-or-skip decision onto a binary tree. The state becomes a pair `(rob_current, skip_current)` returned from each recursive call, with children's values combed out on the way up. If the 1D recurrence feels obvious by now, the tree version is mostly bookkeeping.

## How to recognize this pattern before writing code

The trigger for this whole family is two things appearing together: a sequence, and a constraint that says you can't pick two adjacent elements while maximizing a sum. If both are present, the answer is almost always a 1D `dp` with a `max` recurrence, and the base cases are the first one or two elements.

If only one of those is present, slow down. "Maximize sum with no adjacent picks" on a circular array adds one extra observation. On a tree, it changes the state shape. On a problem where you can also skip arbitrary elements, the adjacency constraint may be a red herring and you probably want a different pattern.

## Where AlgoMonster fits, honestly

AlgoMonster is a pattern-based interview prep platform built by ex-Google engineers, and its House Robber page sits inside the dynamic programming section rather than being a standalone answer key. The teaching format is text plus interactive diagrams, with an in-browser editor and templates per pattern, and Python is one of the supported languages alongside Java, JavaScript, C++, Go, and Haskell.

Two caveats worth knowing before you pay for it, both from AlgoMonster's own forum rather than from marketing copy.

First, DP is treated as lower return on investment than other patterns. In the House Robber discussion thread, an AlgoMonster team member states plainly that DP is lower ROI and not worth heavy time investment unless you're targeting companies that lean on it, and notes Google's interview format specifically as the exception. That's a defensible stance for general prep and an inconvenient one if DP is exactly what you came for.

Second, the same thread has multiple readers asking for a deeper DP treatment, with one pointing at a well-known LeetCode editorial as a better model for explaining it, and the team acknowledging the section was being reworked. So if your goal is pattern recognition for the adjacent-constraint family, the templates are solid. If your goal is a from-scratch DP education, the section is thin in places and you'll want a second source.

If pattern templates look like the gap you're trying to close, 👉 [see what's inside the current AlgoMonster plan](https://algo.monster?fpr=li69) before deciding.

## AlgoMonster pricing: every plan on the page

AlgoMonster keeps this short. There's no team or enterprise tier, and no separate purchase for the coding curriculum.

| Plan | What you get | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems | $0 | — | [Start on the free plan](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems, unlimited AI assistant usage, unlimited Speedrun, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, company question banks updated monthly, member discount on live bootcamps | $99/year (shown as $8.25/mo billed annually) | Annual, cancel or upgrade anytime | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the yearly plan plus lifetime updates, one-time payment | $189 | One time | [Get Pro Lifetime access](https://algo.monster?fpr=li69) |

Two honest notes on this table.

The pricing page currently presents Yearly and Lifetime with a sale price applied, listing the yearly plan at $99 (against a $198 reference price) and lifetime at $189 (against $378). An older capture of the same page also showed a Pro – Monthly tier at $18.99/month, and third-party reviews disagree about whether monthly billing still exists, with some citing $459 for lifetime from an earlier pricing cycle. Check the live checkout for the number you'll actually be charged rather than trusting any review, including this one.

Live bootcamp discounts are included, but specialized partner courses such as Quant Interview Prep are sold separately.

### Is it worth it for a DP problem?

If you're stuck on House Robber, don't buy anything yet. The four solutions above are the whole problem, and they'll cost you twenty minutes. Where a paid curriculum earns its place is the part after that: knowing that the same take-or-skip recurrence resurfaces in dozens of problems, and having a decision flowchart that gets you from problem statement to pattern without a fifteen-minute detour. The yearly plan runs $99 and the lifetime plan $189, so the lifetime option pays for itself if you expect to be back in interview prep in a second cycle, which most engineers are within a few years. If you have a single interview in three weeks, the yearly plan covers it and the lifetime upgrade is money parked for later.

## Quick FAQ

**Should I write the memoized version or the O(1) version in an interview?**
Write the memoized version first because it's four lines and clearly correct, then optimize to two variables when asked. Announcing the optimization before writing it is usually worth more than the optimization itself.

**Is `@lru_cache` acceptable in an interview?**
Yes, with one sentence of explanation: it caches on the recursion index, which makes each state compute once, converting O(2ⁿ) into O(n) time and O(n) space. Be ready to explain what the cache would look like without the decorator.

**Does the O(1) space version work on an empty array?**
Yes, it returns 0. That's why it's the safer choice inside a House Robber II helper, where you'll be calling it on slices that can be empty.

**Why not sort the houses?**
Because sorting destroys the adjacency relationship, which is the only constraint in the problem. Sorting would turn it into a different (and easier) problem that you weren't asked to solve.
