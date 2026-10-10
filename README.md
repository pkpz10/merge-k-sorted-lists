# merge k sorted lists: heap and divide-and-conquer solutions, complexity math, and how to pick one under interview pressure

You can merge two sorted lists in your sleep. Then the interviewer turns "two" into "k" and the problem stops being about pointers and starts being about choosing a strategy. That's why this question keeps showing up: it's less a coding exercise than a decision-making test.

Here's what the problem actually asks, the three solutions interviewers expect, the complexity table you need to be able to derive out loud, and the edge cases that trip people up — plus where a structure-based prep platform like AlgoMonster fits if you're building this skill rather than memorizing one answer.

## What the problem is asking

You get an array of `k` linked lists, each already sorted in ascending order. Merge them into one sorted linked list and return its head. The classic example:


Input:  lists = [[1,4,5],[1,3,4],[2,6]]
Output: 1->1->2->3->4->4->5->6


The constraints matter more than most people realize. On the standard version of the problem, `k` goes up to 10⁴, each individual list holds at most 500 nodes, values can be negative, and the total number of nodes across all lists stays at or under 10⁴. So `k` can be far larger than the number of nodes you're merging. That single fact changes which solution is the "right" one, and it's a detail plenty of candidates skip past while skimming.

Two edge cases come up constantly: an empty array (`lists = []` → return `None`) and an array holding a single empty list (`lists = [[]]` → also `None`, not an error).

## The three solutions interviewers expect

### 1. Collect everything and sort

Dump every node's value into one array, sort it, rebuild a linked list.

This is `O(N log N)` time and `O(N)` space, where `N` is the total node count. It's correct, it's three lines, and given that `N` is capped at 10⁴ here, it will pass. It also throws away the one thing you were handed for free: the lists are already sorted. Say that out loud in an interview. Interviewers usually accept the brute force as a starting point, but only if you acknowledge what you're ignoring.

### 2. Min-heap k-way merge

The key observation: at any moment, the next node in your output is the smallest unprocessed node across all `k` lists. Each list exposes exactly one candidate — its current head. Track those candidates in a min-heap of size `k`, and pulling the global minimum drops from `O(k)` to `O(log k)`.

python
import heapq

def mergeKLists(lists):
    cursors = list(lists)
    heap = [(cursors[i].val, i) for i in range(len(cursors)) if cursors[i]]
    heapq.heapify(heap)

    dummy = tail = ListNode()
    while heap:
        val, i = heapq.heappop(heap)
        tail.next = cursors[i]
        tail = cursors[i]
        cursors[i] = cursors[i].next
        if cursors[i]:
            heapq.heappush(heap, (cursors[i].val, i))

    return dummy.next


The `(value, index)` tuple isn't decoration. Python's heap needs comparable items, and `ListNode` objects aren't comparable. Storing the list index as the tiebreaker solves that — and since only one node per list lives in the heap at a time, `(value, index)` pairs are always unique, so the heap never tries to compare two nodes directly.

Complexity: `O(N log k)` time, `O(k)` extra space. There are `N` pops, each followed by at most one push, each costing `O(log k)`. Building the initial heap from the `k` heads is `O(k)` and gets absorbed.

One detail worth mentioning in an interview: this is the textbook k-way merge, and it's the approach that degrades most gracefully when the lists have wildly different lengths. The heap doesn't care that one list has 500 nodes and another has 2.

### 3. Divide and conquer

Instead of merging everything at once, merge pairs and repeat until one list remains — the same shape as merge sort's merge phase, applied at the list level.

python
def mergeKLists(lists):
    if not lists:
        return None
    while len(lists) > 1:
        merged = []
        for i in range(0, len(lists), 2):
            a = lists[i]
            b = lists[i + 1] if i + 1 < len(lists) else None
            merged.append(mergeTwo(a, b))
        lists = merged
    return lists[0]

def mergeTwo(a, b):
    dummy = tail = ListNode()
    while a and b:
        if a.val <= b.val:
            tail.next, a = a, a.next
        else:
            tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next


There are `log k` rounds. Each round touches every node exactly once, so the total work is `O(N)` per round → `O(N log k)` overall. The recursive version costs `O(log k)` stack space; the iterative version above keeps an array of `k/2` list heads at the widest point, which is `O(k)` of pointers but zero new nodes.

python
def mergeKLists(lists):
    if not lists:
        return None
    while len(lists) > 1:
        merged = []
        for i in range(0, len(lists), 2):
            a = lists[i]
            b = lists[i + 1] if i + 1 < len(lists) else None
            merged.append(mergeTwo(a, b))
        lists = merged
    return lists[0]

def mergeTwo(a, b):
    dummy = tail = ListNode()
    while a and b:
        if a.val <= b.val:
            tail.next, a = a, a.next
        else:
            tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next


Be careful about the `<=` in `mergeTwo`. Using `<` instead won't break correctness here because the output is a linked list of distinct node objects, not a stable sort of values — but if an interviewer asks about maintaining relative order among equal values from different lists, that's exactly where the comparison operator matters.

## Complexity comparison

| Approach | Time | Space | Good for |
| --- | --- | --- | --- |
| Collect all values, sort | O(N log N) | O(N) | Tiny inputs, quick warm-up answer |
| Merge lists one by one | O(N·k) worst case | O(1) extra | Very small k; earlier merges get re-walked repeatedly |
| Min-heap k-way merge | O(N log k) | O(k) | Uneven list lengths, large k, streaming-friendly |
| Divide and conquer | O(N log k) | O(log k) recursion or O(k) pointers | Equal-length lists, when you want no heap |

The sequential-merge row deserves a sentence. Merging list 0 into list 1, then that result into list 2, and so on means the nodes you merged early get traversed again at every later step. With `k` lists, the first list's nodes participate in up to `k` passes. `N` nodes times roughly `k` passes is where the `O(N·k)` comes from, and it's why sequential merging blows up while the other two don't.

## Which one to write

If you only prepare one, prepare the heap. It's `O(N log k)`, it handles ragged list lengths without special-casing, and it generalizes directly to "merge k sorted arrays" and to streaming data where you can't hold everything in memory.

Divide and conquer is the one to reach for when the interviewer nudges you toward "no extra data structure" or when the lists are all the same length, in which case D&C avoids the heap's constant-factor overhead from `log k` sift operations per node. Same asymptotic time, slightly different constants, and interviewers do ask why you'd pick one.

The honest tiebreaker: if `k` is small (say under 10) and the lists are equal length, sequential merging is fine and the code is shorter. The asymptotics bite when `k` is large, and the constraints on this problem explicitly allow `k` up to 10⁴. That's the number to point at when justifying your choice.

## Edge cases and the follow-ups that decide the interview

Getting the right output is the entry fee. The follow-ups are where offers get decided:

- **Empty input and `[[]]`** — both return `None`. Handle the null head before pushing into a heap, or you'll push `None` and crash on the first comparison.
- **Negative values** — no adjustment needed for either approach, but don't assume values are non-negative when writing your base cases.
- **`k = 1`** — return the single list unchanged. Not a special case you need to code, but worth naming.
- **Nothing fits in memory** — this is the classic escalation. The answer is the same k-way merge, applied across disk chunks instead of linked lists: external merge sort, where the heap keeps one buffer per chunk and you write out blocks as you consume them.
- **Streaming input** — you can't sort what you haven't seen, which is exactly why the "collect and sort" answer fails here and the heap survives.

If you want to internalize the pattern rather than four separate solutions, this is the moment the problem stops being about linked lists. "Merge k sorted things" and "find the k-th smallest across k sorted things" and "keep the top k elements from a stream" are all the same idea with different wrappers: maintain a bounded heap over one candidate per source, pop the extreme, refill from the source you just drained.

That reframing is the entire pitch of AlgoMonster, and it's worth checking whether the pitch matches how you actually learn. 👉 [See how AlgoMonster teaches the heap pattern rather than the one problem](https://algo.monster?fpr=li69)

## Where this fits into structured prep

AlgoMonster is an interview-prep platform built by ex-Google engineers. Instead of a problem bank you browse at random, it teaches reusable patterns — the two pointers, sliding window, BFS/DFS, and dynamic programming families — with code templates you adapt to variations. Reviews of the platform consistently describe the curriculum as organized around roughly 48 patterns, with each pattern paired to a handful of practice problems rather than an endless list.

What the platform's own pricing page advertises: 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, unlimited use of an AI assistant that explains any selected text or debugs submitted code, an unlimited Speedrun mode for timed review, and company-specific question banks refreshed monthly.

The relevant caveat for a problem like this one: AlgoMonster is entirely text and diagrams. There's no video. If you learn by watching someone else type, this format will feel dry, and that's a real limitation rather than a marketing footnote. The other gaps reviewers consistently point out are thin system design coverage for senior loops and the modest problem volume — roughly 230 lessons and problems, against LeetCode's 3,000+. The usual recommendation in those reviews is to learn patterns on AlgoMonster and grind volume on LeetCode, not to treat either one as complete.

## Plans and current prices

AlgoMonster has no monthly option. It's free tier, annual, or a one-time lifetime payment.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems — enough to judge the teaching style | $0 | Free, ongoing | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks updated monthly; live bootcamp member discount | $8.25/mo, billed as $99/year | Recurring, cancel or upgrade anytime | [Check the yearly plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates | $189 | One-time payment | [Check lifetime access](https://algo.monster?fpr=li69) |

Two things worth knowing before you click. First, AlgoMonster runs its paid plans at a discount as a standing practice — the pricing page shows sale prices with the struck-through originals alongside, and the same page has displayed lower promotional numbers at other times, so treat the checkout total as the number that counts, not whatever a blog post (including this one) quoted. Second, there's no formal free trial beyond the free tier, so the free content is your only preview. Use it to test whether reading diagrams beats watching a video for you. If the free tier feels like homework you have to force yourself through, the paid plans won't fix that. 👉 [Run the free tier against a pattern you already know](https://algo.monster?fpr=li69)

## Quick answers

**Is there a dedicated merge k sorted lists lesson?** I can't confirm a specific lesson page from the public material, and I'm not going to guess at one. What the platform sells is the pattern layer — heaps, priority queues, and the k-way merge idea — which is what actually transfers when the interviewer changes the problem on you.

**Should I buy annual or lifetime?** Do the arithmetic on your own interviewing habits. One interview cycle in the next couple of months means annual. Expecting to interview again in two or three years, which is normal for engineers who switch companies, and lifetime at $189 pays for itself against a second annual purchase.

**Do I still need LeetCode?** Yes, if the companies you're targeting pull from a deep question pool. Pattern recognition tells you which approach to try; volume practice makes you fast at writing it. The heap solution above is maybe 20 lines, and 20 lines written cold under a timer is a different skill from 20 lines read and understood.

**Is a heap really necessary for this problem?** No. With the total node count capped around 10⁴, collect-and-sort passes, and if you're interviewing somewhere that values clear code over optimal constants, saying so is a legitimate answer — as long as you also say why the heap exists.
