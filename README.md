# kth largest element in an array: how min-heap, quickselect, and sorting compare, and which one to write in an interview

The problem statement is two sentences long. It sits in LeetCode's top interview question set with tags for heap and divide-and-conquer, and it also shows up in NeetCode's 250 list. Almost everyone who reads it can produce *an* answer. The gap between candidates is rarely correctness — it's which approach they commit to, and how well they defend it when the interviewer asks "can you do better?"

That follow-up is written into the problem itself on some platforms: solve it without sorting. So let's take the problem apart properly, then look at the cheapest way to actually get good at this family of questions.

## What the problem is really asking

Given an unsorted integer array `nums` and an integer `k`, return the kth largest element. The statement adds a clause that people skim past: it's the kth largest element **in sorted order, not the kth distinct element**. Duplicates count.

Two examples cover it:

- `nums = [3,2,1,5,6,4]`, `k = 2` → `5`
- `nums = [3,2,3,1,2,4,5,5,6]`, `k = 4` → `4`

In the second one there are repeated values and the answer is still just a position in sorted order. If your solution deduplicates first, you fail that test case.

The single most useful reframing for the whole problem: after sorting ascending, the kth largest sits at index `n - k`. For `[3,2,1,5,6,4]`, `n - k = 4`, and the sorted array is `[1,2,3,4,5,6]`, so `nums[4] = 5`. Every efficient solution below is an attempt to get at that one index without paying for the other `n - 1` positions.

One constraint detail that most English-language write-ups skip: the values in this problem are bounded to a small range. The Chinese-language editorial on the problem explicitly points at that ~10,000-wide value range and solves it with a counting-sort-style array, offsetting for negative numbers. That's a legitimate third answer if your interviewer mentions bounded values, and it's a nice thing to have in your pocket because it signals you read the constraints instead of the title.

## Approach 1: sort it

python
def find_kth_largest(nums, k):
    nums.sort()
    return nums[len(nums) - k]


Two lines, obviously correct, `O(n log n)` time and `O(1)` extra space in the in-place Python case (languages that return a new list will cost `O(n)` for the copy).

This passes. It also wastes work: you computed the full ordering to read one element. Sorting is the right answer when `n` is small, when you actually need the full order later, or when you're writing the baseline version of the solution in the first two minutes so you have something on the board. Say that out loud. "I'll start with sorting, which is `O(n log n)`, and then replace it" reads as a plan. Silently writing `.sort()` and stopping reads as not knowing anything better exists.

## Approach 2: a min-heap of size k

This is the answer most interviewers are fishing for, and the one NeetCode's platform lists as its recommended target — `O(n log k)` time, `O(k)` space.

The counterintuitive part, and the part worth explaining rather than reciting: you use a **min**-heap, not a max-heap. The kth largest element is the *smallest* member of the top-k club. A min-heap keeps its minimum at the root, so the root is always exactly the element you're going to be asked for. The heap acts as a sieve with k slots — anything smaller than the current root gets flushed when the sieve overflows.

python
import heapq

def find_kth_largest(nums, k):
    heap = []
    for num in nums:
        heapq.heappush(heap, num)
        if len(heap) > k:
            heapq.heappop(heap)
    return heap[0]


Running it on `nums = [3,2,1,5,6,4]`, `k = 2`:

| Step | num | Heap after push | Overflow? | Heap after pop | Root |
| --- | --- | --- | --- | --- | --- |
| 1 | 3 | [3] | no | [3] | 3 |
| 2 | 2 | [2,3] | no | [2,3] | 2 |
| 3 | 1 | [1,2,3] | yes, pop 1 | [2,3] | 2 |
| 4 | 5 | [2,3,5] | yes, pop 2 | [3,5] | 3 |
| 5 | 6 | [3,5,6] | yes, pop 3 | [5,6] | 5 |
| 6 | 4 | [4,5,6] | yes, pop 4 | [5,6] | **5** |

Each push and pop costs `O(log k)` because the heap never holds more than `k + 1` elements, and you do that twice per element in the worst case, giving `O(n log k)`. Space is `O(k)`.

A useful number to have ready when someone asks whether the log factor matters: with `k = 8` and `n = 1,000,000`, `log k = 3` while `log n ≈ 20`. The heap is roughly seven times less work than sorting. Flip that around and you get the honest caveat — if `k` is close to `n`, `log k` is barely smaller than `log n`, and the heap is sorting with extra bookkeeping plus a heap allocation.

Two implementation traps:

- Python's `heapq` is a min-heap; Java's `PriorityQueue` also defaults to a min-heap. If you reach for a max-heap and pop `k` times, you're back to `O(n)` memory and a different complexity story. It works, it's just a worse answer.
- Watch the ordering of the push/pop. Popping first and pushing the new element only when it beats the root is a common micro-optimization; pushing then popping unconditionally is simpler and does more heap operations than necessary.

## Approach 3: quickselect

Quickselect is quicksort with the second half of the recursion deleted. Quicksort partitions around a pivot, which lands in its final sorted position, then recurses into both sides. You only need one index, so you recurse into one side and throw the other away.

That single change moves the recurrence from `T(n) = 2T(n/2) + O(n)` to `T(n) = T(n/2) + O(n)`, which solves to `O(n)` on average.

python
def find_kth_largest(nums, k):
    target = len(nums) - k  # kth largest = (n-k)th element in ascending order

    def partition(left, right):
        pivot = nums[right]
        store = left
        for i in range(left, right):
            if nums[i] < pivot:
                nums[store], nums[i] = nums[i], nums[store]
                store += 1
        nums[store], nums[right] = nums[right], nums[store]
        return store

    left, right = 0, len(nums) - 1
    while True:
        p = partition(left, right)
        if p == target:
            return nums[p]
        if p < target:
            left = p + 1
        else:
            right = p - 1


Three things you must volunteer before the interviewer asks:

1. **Worst case is `O(n²)`.** If the pivot is consistently the smallest or largest element, each partition shrinks the problem by one: `T(n) = T(n-1) + O(n)`. Randomizing the pivot (shuffle first, or pick `random.randint(left, right)` and swap it to the end) makes that astronomically unlikely but not impossible.
2. **It mutates the input.** In an in-place implementation you're reordering the caller's array. In a real system with a read-only buffer or a shared array across threads, that's disqualifying, and saying so unprompted is the kind of thing that reads as production experience.
3. **Duplicates degrade a two-way partition.** The classic pathological input here isn't sorted data, it's all-equal data. A three-way partition (Dutch national flag: less than / equal to / greater than the pivot) groups equal elements in one pass and kills that failure mode.

Guaranteed linear time exists — median-of-medians gives you `O(n)` worst case — but it's a lot of code and interviewers almost never expect it. Knowing it exists and being able to say why it isn't worth writing is worth more than writing it.

## Side by side

| Approach | Time (average) | Time (worst) | Extra space | Mutates input | Works on a stream | Pick it when |
| --- | --- | --- | --- | --- | --- | --- |
| Sort | `O(n log n)` | `O(n log n)` | `O(1)` or `O(n)` | yes | no | Small `n`, or you need the full order |
| Min-heap of size k | `O(n log k)` | `O(n log k)` | `O(k)` | no | yes | Default interview answer; `k` small; input is streaming or read-only |
| Max-heap of size n | `O(n + k log n)` | `O(n + k log n)` | `O(n)` | no | yes | Almost never — same space as sorting, worse than the size-k heap |
| Quickselect | `O(n)` | `O(n²)` | `O(1)` | yes | no | You need the theoretical best and can mutate the array |
| Counting / bucket | `O(n + V)` | `O(n + V)` | `O(V)` | no | yes | Values are bounded to a small range (`V` = range size) |

## Edge cases and off-by-ones people actually lose points on

- `k = 1` should return the maximum; `k = n` should return the minimum. Both fall out of `nums[n - k]` for free, which is a good reason to prefer ascending sort plus `n - k` over descending sort plus `k - 1`.
- The "kth distinct" misreading. Deduping breaks the second sample case.
- Negative values. Counting sort needs an offset to map the value range onto array indices; forgetting it is an index-out-of-range crash, not a wrong answer.
- Empty or `null` input. Constraints say `1 ≤ k ≤ n`, but a production version of this function still needs the guard.
- Java comparator overflow: `(a, b) -> b - a` on large values overflows int and produces a silently wrong ordering. Use `Integer.compare`.
- The `target == left == right` termination. In a `while` loop version, make sure you have a return path when `left == right`, or you'll hang on a single-element subarray.

## The follow-ups, which is where this problem actually lives

LeetCode 215 is the doorway to a family. Interviewers use it as a warmup and then push in one of these directions:

**Kth largest in a stream (LeetCode 703).** You get a constructor with `k` and a sorted-ish initial list, then repeated `add(val)` calls that must each return the current kth largest. Same size-k min-heap, with the heap maintained across calls. Each `add` is `O(log k)`. Building the initial heap costs `O(n log k)` naively, or `O(n + (n-k) log k)` if you heapify then trim.

**Top K frequent elements (347).** Same selection idea, different key: count frequencies, then select the top k counts — with a heap, or with bucket sort over the frequencies for `O(n)`.

**Find median from a data stream (295).** Two heaps, one max and one min, kept balanced. If you understand why the size-k heap works here, you understand why the two-heap balance invariant works there.

**"What if the data doesn't fit in memory?"** This is the question that separates people who memorized quickselect from people who understand it. Quickselect needs random access into the whole array, so it's out. The size-k heap is a single pass with `O(k)` memory, so it's in. That's the honest answer, and it takes ten seconds to give.

**"What's the worst case and how do you avoid it?"** Answer with the `O(n²)` scenario and random pivots, not with a shrug.

## Hitting this pattern once instead of 200 times

Here's the thing about `kth largest element in an array`: you will meet it again as "k closest points to origin", "kth smallest element in a sorted matrix", "top k frequent words", and "median of a stream". These are the same selection problem wearing different clothes. Practicing the family once, with the template and the complexity trade-off written down, is roughly an order of magnitude cheaper than solving each instance cold.

That's the pitch behind [👉 AlgoMonster's pattern-based curriculum](https://algo.monster?fpr=li69), which is built by ex-Google engineers and organized around reusable templates rather than a problem list. The relevant part for this problem is the heap and priority-queue pattern content: lesson, template, then a handful of problems that each bend the pattern slightly. The platform also ships a decision flowchart that walks you from a problem description to a candidate pattern through a series of questions — is the input sorted, are you selecting by rank, is the data streaming — which is precisely the reasoning an interviewer is trying to hear from you, in checklist form. Whether you learn better from that or from raw volume is a real question; more on that below.

The practice environment is an inline IDE in the page with an AI assistant woven through it: select any paragraph and ask for a plain-language explanation, or submit a solution and get the failing case pointed at. There's a Speedrun mode for timed pattern review before an onsite, and a set of templates written to be adapted rather than retyped.

## Pricing

AlgoMonster has no monthly plan. The site shows a free tier plus two paid options, and the paid prices below are what the pricing page displayed at the time of writing — they're sale prices applied against a list price, so check the checkout before you commit.

| Plan | What you get | Price | Billing | Fits who |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems — enough to judge the teaching style | $0 | Free forever | Anyone deciding whether the text-and-diagram format works for them |
| Pro — Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; company questions refreshed monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; bonus Foundation and Build-an-LLM courses; member discount on live bootcamps | $99/year ($8.25/month billed annually; list price shown as $198) | 12 months, cancel or upgrade anytime | A prep window measured in weeks, with interviews already scheduled |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates on all current and future content | $189 one-time (list price shown as $378) | Paid once, yours permanently | Anyone who expects to interview again in 2-3 years |

👉 [Check the current yearly price and start the free tier](https://algo.monster/subscribe?fpr=li69)

👉 [See what the lifetime plan includes](https://algo.monster/subscribe?fpr=li69)

Three things worth knowing before you pay, because no pricing page advertises its own gaps:

- **There is no video anywhere.** Every explanation is text with interactive diagrams. Reviews from several independent write-ups converge on the same conclusion: efficient if you read well, actively worse if you learn from watching someone else code.
- **The problem bank is small.** AlgoMonster's selling point is a bounded curriculum with an endpoint, not a 3,000-problem bank. If you already recognize patterns and just need reps under pressure, you're paying for the part you don't need.
- **System design coverage is introductory.** Fine for a `k`-selection drill, not enough for a senior loop where design is a full round.

## What to actually say in the interview

Open by restating the two things that matter: duplicates count, and `k` is always valid so you don't need to defend against `k > n`. Write the sort baseline in a minute so something correct exists on the board. Then offer the size-k min-heap as your main solution, explaining *why* a min-heap and not a max-heap — that's the moment where the interviewer finds out whether you understand the data structure or just remember the answer. Then mention quickselect as the asymptotic upgrade, along with its `O(n²)` worst case and its mutation of the input.

If you have spare time at the end, drop the bounded-value-range observation: with a small value range, counting sort gives `O(n + V)` and needs no comparison at all. It isn't the answer anyone expects. It is the answer that makes you look like you read the problem instead of the title.
