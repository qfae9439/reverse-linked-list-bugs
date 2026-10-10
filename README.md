# reverse a linked list: the two working solutions, the edge cases that break them, and how to say it out loud in an interview

Ask ten engineers to name the problem they've been asked most often in a coding screen and "reverse a linked list" shows up on nearly every list. It's LeetCode 206, it's rated Easy, and it still filters people out — not because the idea is hard, but because the implementation has almost no room for fuzzy thinking. You either track three pointers correctly or you produce a cycle and a wrong answer.

Here's the whole problem, the way it's usually stated: given the head of a singly linked list, reverse the list, and return the reversed list. Input `[1,2,3,4,5]`, output `[5,4,3,2,1]`. LeetCode's constraints allow between 0 and 5,000 nodes with values in the range −5000 to 5000, and the official follow-up asks you to do it both iteratively and recursively. That follow-up is not decoration. Expect it.

## What the question is really grading

Nobody cares whether you can produce a reversed list on paper. An interviewer is watching four things:

- Whether you have a clear invariant — a statement like "everything to the left of `prev` is already reversed" — that you can say out loud while your hands keep moving
- Whether you handle the boundaries: an empty list, a single node, two nodes
- Whether the complexity claim you make is accurate and whether you know *why* the recursive version costs more memory than the iterative one
- Whether you can adapt when the interviewer says "now reverse only the nodes between positions 2 and 4"

The pointer manipulation is a symptom. The invariant is the answer.

## The iterative solution: three pointers, one link flipped per step

The core observation is that you're not moving nodes. You're re-pointing each node's `next` field at the node that used to come before it. To do that without losing the rest of the list, you need to remember where you came from and where you're going.

python
def reverseList(head):
    prev = None
    curr = head
    while curr:
        nxt = curr.next      # save the rest of the list
        curr.next = prev     # flip the link
        prev = curr          # advance prev
        curr = nxt           # advance curr
    return prev


The order of those four lines inside the loop is not a style choice. If you write `curr.next = prev` before saving `curr.next` into `nxt`, the remainder of the list becomes unreachable — you've overwritten your only handle on it. That single line moved one position too early is the most common way this problem is failed.

Trace it on `1 → 2 → 3 → 4 → 5`:

| Step | prev | curr | nxt | List state |
| --- | --- | --- | --- | --- |
| start | `None` | `1` | — | `1→2→3→4→5→None` |
| 1 | `1` | `2` | `2` | `None←1  2→3→4→5→None` |
| 2 | `2` | `3` | `3` | `None←1←2  3→4→5→None` |
| 3 | `3` | `4` | `4` | `None←1←2←3  4→5→None` |
| 4 | `4` | `5` | `5` | `None←1←2←3←4  5→None` |
| 5 | `5` | `None` | `None` | `None←1←2←3←4←5` |

When `curr` falls off the end, `prev` is sitting on the old tail, which is the new head. That's what you return.

The same thing in Java, since a lot of screens are Java or C++:

java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode curr = head;
    while (curr != null) {
        ListNode nxt = curr.next;
        curr.next = prev;
        prev = curr;
        curr = nxt;
    }
    return prev;
}


Inititializing `prev = null` rather than to `head` matters. The original head becomes the new tail, and its `next` must end up `null`. Starting `prev` at `null` makes that happen for free.

## The recursive solution: reverse the tail, then fix one link

The recursive version works back to front. Assume the sublist after the current node is already reversed; then you only need to attach the current node to the far end of that result.

python
def reverseList(head):
    if not head or not head.next:   # empty list or single node
        return head
    new_head = reverseList(head.next)
    head.next.next = head           # the node after me points back at me
    head.next = None                # and I stop pointing forward
    return new_head


The line people drop is `head.next = None`. Walk through `1 → 2` and you'll see why it matters: `head.next.next = head` makes node 2 point back at node 1, and node 1 is *still* pointing at node 2. That's a two-node cycle. Your reversal "worked" at every call, and the final list is broken. LeetCode's own editorial points at exactly this, and recommends testing with a list of size two specifically because that's the smallest input where the bug surfaces.

## Iterative or recursive? Pick based on the constraint, not the aesthetics

|  | Iterative | Recursive |
| --- | --- | --- |
| Time | O(n), each node visited once | O(n), each node visited once |
| Extra space | O(1) — three pointers | O(n) — one stack frame per node |
| Reads as | Explicit invariant, easy to trace | Compact, easier to get subtly wrong |
| Stack risk | None | Real on long lists in Python |
| Best used when | Memory matters, or the interviewer wants reliability | The interviewer asks for elegance, or input is short |

That stack risk isn't hypothetical. Python's default recursion limit is 1,000, and this problem permits 5,000 nodes. A clean recursive solution on LeetCode's maximum input raises a `RecursionError` unless you raise the limit with `sys.setrecursionlimit`. In Java or C++ you'll typically clear 5,000 frames on a default stack, but the auxiliary memory is still O(n) — the problem asks for O(1), so the honest answer is that only the iterative version meets it.

If an interviewer asks which one to ship, say the iterative one and give the reason: constant extra space, no depth limit, and the loop can be stepped through by hand during code review.

## The four bugs that show up over and over

1. **Lost remainder.** You flipped `curr.next` before storing it. The list after `curr` is now garbage.
2. **Cycle at the old head.** You forgot `head.next = None` in the recursive version, so the new tail points into the list.
3. **Returning the wrong node.** Returning `head` instead of `prev` from the iterative version. It compiles, runs, and hands back the old head — which is now the tail of a single wrong node.
4. **No edge-case guard.** Empty list (`head = []`, valid per the constraints) or one node. The iterative version survives both without special-casing; the recursive version's base case covers them only if you check `not head.next` and `not head`, in that order of safety (`head or head.next` short-circuits correctly).

There's a fifth, subtler one: in Python, writing the loop body as a single tuple assignment — `curr.next, prev, curr = prev, curr, curr.next` — is technically correct because the right side evaluates fully first. It's also harder to defend out loud, and interviews are conversations. Write the four-line version.

## Follow-ups you should expect, and why they're the same problem

Every one of these is the base reversal plus bookkeeping. Learn the base case properly and they stop being new problems.

- **Reverse nodes between positions m and n** (LeetCode 92). Add a dummy node before the head, walk `prev` to position m−1, then run the standard reversal loop for `n − m + 1` steps and stitch the three parts back together.
- **Reverse nodes in k-groups** (LeetCode 25). Same loop, run in blocks, with a check that each block actually has k nodes before you reverse it.
- **Swap nodes in pairs** (LeetCode 24). This is k-group reversal with k = 2, usually written with the dummy-node pattern.
- **Palindrome linked list** (LeetCode 234). Find the middle with slow/fast pointers, reverse the second half, compare the two halves, then reverse the second half back so you don't leave the caller's list mutated. That last step is where candidates lose points.
- **Reverse a doubly linked list.** Same traversal, but each node has two pointers and you swap them both. No `prev` variable needed — the node already remembers it, which is a nice thing to point out.

Notice the recurring shape: a dummy node, a `prev` pointer, a `curr` pointer, and reversal as a subroutine. That's the pattern worth internalizing — not the eleven-line function.

## If you're drilling this for a real interview loop

One problem isn't a strategy. Reversal is really a lesson in pointer bookkeeping — keeping a reference to the node you're about to orphan — and that skill is what the harder linked-list questions keep reusing: merge two sorted lists, remove the Nth node from the end, detect a cycle, reorder a list. All of them punish the same mistake in different clothing.

AlgoMonster, built by ex-Google engineers and competitive programmers, organizes its curriculum around that idea rather than around a problem count. Its [Blind 75 practice list](https://algo.monster?fpr=li69) flags reversing a linked list explicitly as part of the linked-list entry alongside slow-fast pointers and merging. The teaching format is text plus interactive diagrams — there are no video walkthroughs anywhere on the platform — and the subscribe page currently advertises 325 lessons and problems, 678 illustrations, and 3,000+ editorials across 48 patterns, with questions from major tech companies refreshed monthly.

Three features are the ones that actually matter for a linked-list problem:

- **Interactive code editor in the browser.** You write the reversal, submit, and get feedback without a separate IDE or a LeetCode tab. If your output is wrong, you can see which nodes ended up where.
- **Decision flowcharts.** When you're staring at an unfamiliar list problem and can't tell whether you need a dummy node, slow-fast pointers, or a straight traversal, the flowchart walks you through the questions that narrow it down.
- **Speedrun mode.** Timed passes through problems you've already learned. For a pattern this short, speedrun is the useful part — repetition until `nxt = curr.next` comes out of your fingers without thinking, which is what you actually need under a 45-minute clock.

The AI assistant is woven through the platform rather than bolted on: select any passage and it explains it, and it will debug a submission you've already written. That's a real difference from reading a static editorial, though it won't replace a human telling you your invariant statement was unclear.

Independent reviews are consistent about where the platform stops being enough. AlgoMonster is entirely text and diagrams, which is efficient if you learn by reading and painful if you learn by watching. The problem set is deliberately compact rather than a 3,000-question bank, so volume practice still means LeetCode. System design coverage is introductory, and there is no live mock interview environment — the platform builds pattern recognition, it doesn't simulate someone watching your screen. Several reviewers also mention a job guarantee attached to completing the curriculum; check the current terms on the site rather than taking a blog's word for it.

## What it costs

AlgoMonster has no monthly plan. You either pay for a year or buy lifetime access, and there's a free tier with limited access that's genuinely useful for judging whether the text-and-diagram format suits you.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems — enough to test the format | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems, unlimited AI assistant and Speedrun, company questions updated monthly, 12 months of full access | $119/year (shown as $9.92/mo billed annually) | Annual, cancel or upgrade anytime | [Check the Pro Yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan plus lifetime updates, one 1:1 coaching session with the founder, and an internal referral to a top tech company for qualified candidates | $459 one-time | One payment | [Check the Pro Lifetime plan](https://algo.monster?fpr=li69) |

Two things worth knowing before you click. First, AlgoMonster runs promotions frequently and the gap is large — checkout pages have shown an annual price near $99 and a lifetime price near $189 during sale periods, and the $459 lifetime figure above is itself already discounted from a listed full price. Second, because reviewers reporting prices for the same product land anywhere from $89 to $459 depending on when they looked, treat the number on the checkout page as the only current one. Third-party promo codes circulate widely; if you use one, confirm it's applied before paying, since an expired code changes nothing.

Which plan is right is mostly arithmetic. If you have an interview in six to eight weeks, the yearly plan covers the window the curriculum is designed for and costs less than one LeetCode Premium year. If you expect to search for jobs again in two or three years — which is the normal pattern for engineers rather than the exception — the lifetime plan stops being a gamble around the second cycle.

## The short version

Reversal is four lines of code and three pointers, and the entire difficulty lives in the order of those lines and the two edge cases. Write the iterative version until you can explain the invariant while typing it. Write the recursive version until you remember `head.next = None` without being told. Then go do the k-group and m-to-n versions, because those are what get asked after you've solved the easy one.

If structure is the thing missing from your prep rather than effort, a pattern-based platform will get you further than another hundred random problems. 👉 [Compare the AlgoMonster plans and current pricing](https://algo.monster?fpr=li69) — the free tier is enough to find out whether the format works for how you think.
