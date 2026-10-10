# House Robber: The DP Recurrence, the Greedy Trap, and How to Get It Down to O(1) Space

People arrive at this problem in two very different moods. Some read the statement, think "obviously I just take every other house," and get a rejection a minute later. Others get the right answer on the sample cases and still can't explain why their loop works. Both groups usually trip over the same thing: the recurrence, not the code.

So here's the whole problem, the reason the intuitive answer is wrong, and the version of the solution you should be able to write from scratch in an interview.

## The problem, in one paragraph

You're given `nums`, an array where `nums[i]` is the money sitting in house `i`. Two adjacent houses can't both be robbed on the same night, or the alarm triggers. Return the maximum total you can take.

The classic inputs:

text
nums = [1,2,3,1]     -> 4   (houses 0 and 2: 1 + 3)
nums = [2,7,9,3,1]   -> 12  (houses 0, 2, 4: 2 + 9 + 1)


On LeetCode (problem 198) the constraints are small and friendly: the array has between 1 and 100 elements, and each value is between 0 and 400. The platform currently labels it Medium. The size limits mean even a slow recursive solution will technically pass on the judge — which is exactly why this problem is worth studying properly instead of getting away with brute force.

## Why "take every other house" is wrong

The tempting shortcut is to sum all the even-indexed houses, sum all the odd-indexed ones, and take the larger of the two. It hits both sample cases. It's still wrong.

Try `nums = [2, 12, 9, 3, 4]`.

- Even indices (0, 2, 4): 2 + 9 + 4 = 15
- Odd indices (1, 3): 12 + 3 = 15

Both parities land on 15. The real answer is 16, from houses 1 and 4 (12 + 4). Those two houses are three positions apart, so nothing is adjacent and the alarm stays quiet — but the parity trick can never produce that combination, because it forces you to commit to one alignment for the entire street.

This is the same mistake you'll see in the wild. In AlgoMonster's own discussion forum for the problem, a reader reports that their odd/even parity solution passed every test case on the practice page and then got rejected immediately by LeetCode. A separate comment thread on the same page points at a base-case snippet that reads ambiguously. The lesson generalizes: sample cases don't validate a DP solution. A counterexample does.

## The recurrence, which is the entire problem

Stop thinking about the street as a whole and think about one house.

At house `i` you have exactly two choices:

1. **Skip it.** Your best total is whatever you had through house `i - 1`.
2. **Rob it.** You take `nums[i]`, and the previous house is now off limits, so you add your best total through house `i - 2`.

That's it:

text
dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])


The greedy failure now makes mechanical sense. The parity trick assumes the decision is fixed in advance; the recurrence lets house `i` depend on two previous answers, so taking a cheap house can be worth it if it unlocks an expensive one two doors down.

## Three implementations of the same idea

They differ in speed and in how much memory they burn, not in logic.

### 1. Plain recursion — O(2^n)

python
def rob(nums, i=0):
    if i >= len(nums):
        return 0
    return max(nums[i] + rob(nums, i + 2), rob(nums, i + 1))


Correct, and unusable past roughly 30 houses, because `rob(nums, i + 1)` keeps re-answering questions that `rob(nums, i + 2)` already answered. The recursion tree branches into overlapping subproblems — that overlap is the whole reason dynamic programming applies here.

### 2. Top-down with memoization — O(n) time, O(n) space

python
from functools import lru_cache

def rob(nums):
    @lru_cache(maxsize=None)
    def dp(i):
        if i < 0:
            return 0
        return max(dp(i - 1), dp(i - 2) + nums[i])
    return dp(len(nums) - 1)


This is the recurrence written literally. The `i < 0` guard handles the two base cases at once, which is why it's less error-prone than hand-writing `if i == 0` and `if i == 1` branches.

### 3. Bottom-up with two variables — O(n) time, O(1) space

Because `dp[i]` only ever looks at `dp[i - 1]` and `dp[i - 2]`, the array is unnecessary:

python
def rob(nums):
    prev2 = prev1 = 0
    for money in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + money)
    return prev1


Trace it on `[2, 7, 9, 3, 1]` and the two variables walk up to 12 without any array at all. In an interview, this is the version worth leading with, provided you can explain the array version first. Writing the O(1) form without being able to justify it looks like memorization.

Complexity summary:

| Approach | Time | Space | Notes |
| --- | --- | --- | --- |
| Recursion | O(2^n) | O(n) stack | Fine for n ≤ ~25 |
| Memoized top-down | O(n) | O(n) | Closest to the recurrence |
| Iterative two-variable | O(n) | O(1) | The version to memorize for pattern recognition |

## The bugs that actually show up

- **Skipping the single-house case.** `nums = [5]` should return 5. Recursive solutions that reference `nums[1]` before checking the length crash on the first test case.
- **Updating the two variables out of order.** `prev2` has to receive the *old* `prev1`. In Python the tuple assignment handles it; in C++ or Java, writing `prev2 = prev1; prev1 = max(prev1, prev2 + nums[i]);` silently corrupts the second line.
- **Treating non-negative values as a gift you must take.** All values are `≥ 0` per the constraints, so there's no penalty for robbing a house — but a solution that assumes "always rob if it helps" collapses the moment the same template is applied to a variant with costs or with negative contributions.
- **Accepting a test suite as proof.** As the forum complaint above shows, a wrong approach can pass every provided case. Write your own counterexample before you trust anything.

## The variants are the same problem in a different costume

This is why House Robber is worth more than the five minutes it takes to accept: it's the entry point to a family of problems that interviewers rotate through.

**House Robber II (circular street).** House 0 and house `n-1` are now adjacent, so you can't take both. Run the same function twice — once on `nums[0:n-1]`, once on `nums[1:n]` — and return the larger result. Two linear passes, no new logic.

**House Robber III (binary tree).** Houses hang off a tree, so adjacent means parent-child. Each recursive call returns a pair: the best total *including* this node, and the best total *excluding* it. Same take/skip decision, different container.

**Delete and Earn.** An array of values where picking `x` deletes every `x - 1` and `x + 1`. Bucket the values into a frequency-weighted array and it becomes House Robber on the sorted distinct values. If you don't see the reduction, you'll write something much worse.

**House Robber IV (LeetCode 2560).** Different animal — the adjacency rule stays, but the question becomes "what's the minimum capability needed to rob at least `k` houses?" The answer is binary search on the capability, with a greedy left-to-right count as the feasibility check. AlgoMonster publishes a walkthrough for it that does exactly that: binary search over the value range, and a helper that walks the array counting houses it can take while skipping the neighbor of the last one taken.

Notice what the family has in common: **a sequence or structure where you choose a subset under a "no two adjacent" constraint and optimize a total.** Once you can name that condition on sight, House Robber stops being a problem you remember and becomes a pattern you apply.

That's the pitch behind pattern-based prep generally, and the reason a problem this small shows up in structured curricula rather than in a random grind list. If you want to see how the take/skip decision is taught as a reusable template instead of a one-off trick, 👉 [it's worth looking at how AlgoMonster organizes the optional-skip DP pattern](https://algo.monster?fpr=li69) alongside the other sequence problems.

## If you're using a platform to drill this

AlgoMonster is built by former Google engineers and organizes its material around a fixed set of coding patterns rather than a large problem bank. House Robber sits in its dynamic programming / sequence material, and the platform also covers House Robber IV, so the variants aren't left as an exercise.

One honest caveat, from the platform's own community: in the discussion threads under the DP section, several users argue the dynamic programming chapter is thinner than the graph, DFS and BFS chapters, and a maintainer replies that they're in the process of reworking it. If DP is your weak spot, read a few of the free lessons before you pay — the teaching quality isn't uniform across chapters.

### Current plans and pricing

AlgoMonster has three tiers, and no monthly billing. The official checkout page currently shows the annual plan at $8.25/month billed annually ($99/year) and lifetime access at $189 billed once.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; enough to judge the teaching style | $0 | None | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited Speedrun; unlimited AI Assistant; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; monthly-updated questions from top tech companies; 12 months of access; member discount on live bootcamps | $99/year ($8.25/mo) | Annual, cancel anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates; one payment | $189 | One-time | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A few things worth knowing before you buy:

- **Some content is sold separately.** The checkout page states that specialized courses developed with external partners, such as Quant Interview Prep, aren't included in the Pro subscription.
- **Third-party price listings disagree with each other.** Review sites published as recently as this year quote everything from $89/year to $299/year and lifetime figures between $99 and $459. Treat the official checkout page as the only number that matters; sale pricing moves, and older articles don't get updated.
- **There's no month-to-month option and no formal free trial.** The free tier is your preview. If you need a plan you can cancel in 30 days, this isn't it.
- **Discount codes exist but the sale is already applied.** Coupon-hunting sites track partner codes — one widely-cited code, `SCHELLEY`, is advertised as an extra 10% off on top of the existing 50% sale — but codes expire without notice, and the checkout page already reflects the reduced pricing. Verify anything you find before entering card details.

For someone working through a DP problem set like House Robber and its variants over a few weeks, the annual plan is the sensible one. Lifetime access only pays off if you expect to come back to this material across multiple job searches.

## Short answers to what people search most

**Is House Robber hard?** Not conceptually — the state is one index and the recurrence fits on one line. It's Medium on LeetCode because of how many people commit to the parity shortcut and how many others can't derive the base cases cleanly.

**Should I use O(n) or O(1) space in an interview?** Write the O(n) array version first so the interviewer can follow the state, then collapse it to two variables and say why the array is unnecessary. Handing over the two-variable loop with no explanation reads as memorized.

**Do I need dynamic programming at all for 198?** For the actual constraints, a memoized recursion is enough and arguably clearer. The transferable skill is recognizing the "no two adjacent, maximize the sum" shape, because that shape recurs in harder problems where brute force genuinely won't pass.

**Where should I practice the variants?** Do House Robber II and III immediately after 198, in the same sitting. If you write II as two calls to the same function instead of a new solution, the pattern has landed. Delete and Earn is the real test — it doesn't mention houses at all, and you're expected to see the reduction yourself.

The interviewers who ask this aren't checking whether you can write eight lines of Python. They're checking whether you decompose a constraint into a state, and whether you can tell a locally optimal choice from a globally optimal one. The greedy trap is the point of the problem. Solve it once by hand, write the counterexample down, and you'll recognize it the next time it's wearing a different title — which, in this family, is about four problems out of the next twenty you'll see.
