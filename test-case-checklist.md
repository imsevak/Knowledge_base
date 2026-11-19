# Test-case checklist

Works for any data structure or function. Write **at least one case per line** before writing code.

1. **Empty**: operations on nothing.
2. **One**: a single element.
3. **Many**: several elements, and the normal case.
4. **Boundaries**: exactly at a limit, and one past it (capacity, size 0, max values).
5. **Invalid input**: NULL, negative, zero size, out of range.
6. **Duplicates**: the same key or value again.
7. **Ordering**: sorted, reverse-sorted, the same element at the front, middle and end.
8. **State transitions**: every operation in every state (empty, partial, full).
9. **The defining rule**: the one thing the structure promises (e.g. LRU: eviction order).
10. **Sequences**: mixed operations in a row, not just one at a time.
11. **Invariants as the oracle**: check them after every step.
12. **Error paths**: failed allocation, and what state is left behind.
13. **Overflow and wraparound**: integer limits, and indexes that wrap.
14. **Resource cleanup**: no leaks on free, on error and on reuse (run ASan).
15. **Stress**: large input and long random runs, checked against a simple reference.

**Memory aid:** start with **Empty, One, Many**, then ask **"where are the edges, and what could go wrong?"**

Not every line applies to every problem. If you skip one, say why in one phrase (e.g. "no duplicates:
keys are unique by spec").

