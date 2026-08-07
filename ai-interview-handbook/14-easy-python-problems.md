# Easy Python Problems (Warm-up Bank)

The 100 classic beginner Python problems that appear in screening rounds, phone screens, and the first ten minutes of a coding interview — Armstrong numbers, palindromes, Fibonacci, prime checks, FizzBuzz, and the rest. Each has a working one-or-two-line idiomatic solution, complexity, and the trap interviewers watch for.

**Problems:** 100

The companion spreadsheet [`easy-python-problems.csv`](easy-python-problems.csv) has the same 100 problems as a filterable sheet (category, frequency, complexity, trap) for tracking revision progress. This file has the code.

These are deliberately *not* formatted as Q1/Q2 interview questions — they are a drill bank. For the pattern-based interview problems (two pointers, sliding window, BFS/DFS, heaps) see [`02-dsa-problem-solving.md`](02-dsa-problem-solving.md).

**How to use this:** cover the solution, write it yourself, then compare. If you can do the Very High frequency rows cold in under two minutes each, you are ready for any screening round. The **traps** column is the real value — most of these problems are failed on an edge case, not on the core idea.

---

## Contents

| Category | Problems | Jump |
|---|---|---|
| Strings | 15 | [↓](#strings) |
| Numbers | 22 | [↓](#numbers) |
| Lists | 23 | [↓](#lists) |
| Dictionaries | 10 | [↓](#dictionaries) |
| Stacks, queues, search | 6 | [↓](#stacks-queues-and-search) |
| Sets | 4 | [↓](#sets) |
| Recursion | 6 | [↓](#recursion) |
| Sorting | 5 | [↓](#sorting) |
| Patterns and files | 7 | [↓](#patterns-files-and-bits) |
| Bit tricks | 2 | [↓](#patterns-files-and-bits) |

---

## The five things that fail these problems

Before the list — these account for most rejections on "easy" problems:

1. **Not normalising input.** Palindrome and anagram questions almost always intend case-insensitive, punctuation-stripped comparison. Ask, then handle it.
2. **Empty and single-element input.** `max([])` raises `ValueError`. Every solution should survive `""`, `[]`, and `0`.
3. **Negative numbers.** `-7 % 2` is `1` in Python, not `-1`, so `n % 2 == 1` is a working odd-check but `n % 2 == -1` never is. Digit-reversal and digit-sum problems need `abs()`.
4. **In-place methods returning `None`.** `lst.reverse()`, `lst.sort()`, `lst.append()` all return `None`. `x = lst.sort()` gives you `None`, and it is the single most common Python interview bug.
5. **Accidentally quadratic code.** `lst.count(x)` inside a loop over `lst`, or `if x in some_list` inside a loop, is O(n²). Reach for `Counter` and `set`.

---

## Strings

### 1. Reverse a string — *Very High*

```python
def reverse(s: str) -> str:
    return s[::-1]
```

**Time:** O(n) · **Space:** O(n) — strings are immutable, so a new one is built. There is no O(1)-space version in Python.

**Trap:** claiming you can reverse "in place." You cannot; if asked for in-place, the interviewer means a list of characters.

---

### 2. Check if a string is a palindrome — *Very High*

```python
def is_palindrome(s: str) -> bool:
    cleaned = [c.lower() for c in s if c.isalnum()]
    return cleaned == cleaned[::-1]

# Two-pointer version — O(1) extra space if you skip the cleaning step
def is_palindrome_two_ptr(s: str) -> bool:
    l, r = 0, len(s) - 1
    while l < r:
        while l < r and not s[l].isalnum(): l += 1
        while l < r and not s[r].isalnum(): r -= 1
        if s[l].lower() != s[r].lower():
            return False
        l, r = l + 1, r - 1
    return True
```

**Time:** O(n) · **Space:** O(n) for the slice version, O(1) for two pointers.

**Trap:** forgetting case-folding and non-alphanumeric stripping. `"A man, a plan, a canal: Panama"` is the canonical test and fails a naive `s == s[::-1]`.

---

### 3. Count vowels in a string — *High*

```python
def count_vowels(s: str) -> int:
    return sum(c in "aeiou" for c in s.lower())
```

**Time:** O(n) · **Space:** O(1) — `booleans` sum as 1/0, which is idiomatic Python.

**Trap:** not lowercasing, so `"AEIOU"` counts as zero.

---

### 4. Check if two strings are anagrams — *Very High*

```python
from collections import Counter

def is_anagram(a: str, b: str) -> bool:
    return Counter(a) == Counter(b)
```

**Time:** O(n) · **Space:** O(k) for k distinct characters.

`sorted(a) == sorted(b)` also works but is O(n log n) — say which you chose and why. For case- and space-insensitive matching, normalise first: `a.lower().replace(" ", "")`.

**Trap:** offering only the sorted version and not knowing it is slower.

---

### 5. Count character frequency — *High*

```python
from collections import Counter

freq = Counter("mississippi")   # {'i': 4, 's': 4, 'p': 2, 'm': 1}
```

**Time:** O(n) · **Space:** O(k)

**Trap:** hand-rolling `if c in d: d[c] += 1 else: d[c] = 1`. It works, but `Counter` (or `defaultdict(int)`) is what a Python engineer writes.

---

### 6. Remove duplicate characters, keeping order — *Medium*

```python
def dedupe(s: str) -> str:
    return "".join(dict.fromkeys(s))
```

**Time:** O(n) · **Space:** O(n)

Dicts preserve insertion order from Python 3.7, which makes this the standard order-preserving dedupe idiom for strings *and* lists.

**Trap:** `"".join(set(s))` — a set has no order, so the output is scrambled.

---

### 7. Find the first non-repeating character — *High*

```python
from collections import Counter

def first_unique(s: str) -> str | None:
    freq = Counter(s)
    return next((c for c in s if freq[c] == 1), None)
```

**Time:** O(n) — two passes, not nested · **Space:** O(k)

**Trap:** a nested loop counting occurrences per character, which is O(n²). Count once, then scan in original order — the ordering of the second pass is what makes "first" correct.

---

### 8. Capitalise the first letter of each word — *Medium*

```python
" ".join(w.capitalize() for w in s.split())
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `str.title()` breaks on apostrophes — `"o'brien".title()` gives `"O'Brien"`. Use `capitalize()` per word.

---

### 9. Count words in a sentence — *Medium*

```python
len(s.split())
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `s.split(" ")` splits on each single space, so `"a  b"` yields `['a', '', 'b']` — a phantom word. Bare `split()` collapses all runs of whitespace and handles tabs and newlines.

---

### 10. Reverse the word order in a sentence — *High*

```python
" ".join(s.split()[::-1])
```

**Time:** O(n) · **Space:** O(n)

**Trap:** reversing characters (`s[::-1]`) instead of words. Read the question twice.

---

### 11. Check if a string contains only digits — *Low*

```python
s.isdigit()          # "123" -> True,  "-1" -> False,  "1.5" -> False
```

**Time:** O(n) · **Space:** O(1)

**Trap:** `isdigit()` is `True` for superscripts like `"²"`. For "is this a valid number," use a `try: float(s)` block instead.

---

### 12. Longest word in a sentence — *Medium*

```python
max(s.split(), key=len, default="")
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `max()` on an empty string raises `ValueError` — pass `default=`.

---

### 13. Remove all whitespace — *Low*

```python
"".join(s.split())
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `s.strip()` only removes *leading and trailing* whitespace, not internal.

---

### 14. Check if a string is a pangram — *Low*

```python
from string import ascii_lowercase

def is_pangram(s: str) -> bool:
    return set(ascii_lowercase) <= set(s.lower())
```

**Time:** O(n) · **Space:** O(1) — the alphabet is a fixed 26.

`<=` on sets is the subset operator, which reads better than checking each letter.

---

### 15. Toggle the case of each character — *Low*

```python
s.swapcase()
```

**Time:** O(n) · **Space:** O(n)

---

## Numbers

### 16. Factorial of a number — *Very High*

```python
import math
math.factorial(n)

def factorial(n: int) -> int:      # if asked to implement it
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

**Time:** O(n) · **Space:** O(1) iteratively, O(n) recursively (call stack).

**Trap:** `0! == 1`, and the recursive version hits Python's ~1000-frame recursion limit. Mention both.

---

### 17. Nth Fibonacci number — *Very High*

```python
def fib(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

**Time:** O(n) · **Space:** O(1)

**Trap:** naive recursion is **O(2ⁿ)** — `fib(40)` takes seconds, `fib(50)` effectively never finishes, because it recomputes the same subproblems exponentially many times. Either iterate as above or memoise (problem 88). Being able to state the exponential blow-up is the point of this question.

---

### 18. Print the Fibonacci series up to n terms — *Very High*

```python
def fib_series(n: int) -> list[int]:
    out, a, b = [], 0, 1
    for _ in range(n):
        out.append(a)
        a, b = b, a + b
    return out
```

**Time:** O(n) · **Space:** O(n) for the output.

**Trap:** off-by-one — clarify whether "up to n" means n *terms* or values *below* n.

---

### 19. Check if a number is prime — *Very High*

```python
def is_prime(n: int) -> bool:
    if n < 2:
        return False
    if n % 2 == 0:
        return n == 2
    for i in range(3, int(n ** 0.5) + 1, 2):
        if n % i == 0:
            return False
    return True
```

**Time:** O(√n) · **Space:** O(1)

**Why √n is enough:** if `n = a × b` then one of the factors is ≤ √n, so a factor above √n would already have been found as the smaller partner. Being able to explain this is what separates a memorised answer from an understood one.

**Trap:** looping to `n` (too slow), and mishandling `n < 2` — 1 and 0 are not prime, and negatives are not either.

---

### 20. Print all primes in an interval — *High*

```python
def sieve(limit: int) -> list[int]:
    is_p = [True] * (limit + 1)
    is_p[0:2] = [False, False]
    for i in range(2, int(limit ** 0.5) + 1):
        if is_p[i]:
            is_p[i*i::i] = [False] * len(is_p[i*i::i])
    return [i for i, p in enumerate(is_p) if p]
```

**Time:** O(n log log n) · **Space:** O(n)

**Trap:** calling `is_prime()` on every number in the range — correct but O(n√n). Start crossing out at `i*i` because smaller multiples were already handled.

---

### 21. Check if a number is an Armstrong number — *Very High*

```python
def is_armstrong(n: int) -> bool:
    digits = str(n)
    power = len(digits)
    return n == sum(int(d) ** power for d in digits)
```

**Time:** O(d) for d digits · **Space:** O(1)

An Armstrong (narcissistic) number equals the sum of its digits each raised to the power of the **number of digits**: 153 = 1³ + 5³ + 3³, and 9474 = 9⁴ + 4⁴ + 7⁴ + 4⁴.

**Trap:** hardcoding `** 3`. That only works for 3-digit numbers and is the mistake this question exists to catch. The exponent must be the digit count.

---

### 22. Sum the digits of a number — *High*

```python
def digit_sum(n: int) -> int:
    return sum(int(d) for d in str(abs(n)))

def digit_sum_math(n: int) -> int:      # without string conversion
    n, total = abs(n), 0
    while n:
        n, r = divmod(n, 10)
        total += r
    return total
```

**Time:** O(d) · **Space:** O(1)

**Trap:** forgetting `abs()`, so a negative input crashes on `int('-')`.

---

### 23. Reverse the digits of a number — *High*

```python
def reverse_digits(n: int) -> int:
    sign = -1 if n < 0 else 1
    n, rev = abs(n), 0
    while n:
        n, d = divmod(n, 10)
        rev = rev * 10 + d
    return sign * rev
```

**Time:** O(d) · **Space:** O(1)

**Trap:** losing the sign, and not realising trailing zeros vanish (120 → 21) — which is correct behaviour, but worth stating so the interviewer knows you noticed.

---

### 24. Check if a number is a palindrome — *High*

```python
def is_num_palindrome(n: int) -> bool:
    return n >= 0 and str(n) == str(n)[::-1]
```

**Time:** O(d) · **Space:** O(d)

**Trap:** negatives — `-121` reversed is `121-`, so by convention no negative number is a palindrome. If the interviewer bans string conversion, reverse the digits arithmetically (problem 23) and compare.

---

### 25. Greatest common divisor — *High*

```python
import math
math.gcd(a, b)

def gcd(a: int, b: int) -> int:      # Euclid's algorithm
    while b:
        a, b = b, a % b
    return a
```

**Time:** O(log min(a,b)) · **Space:** O(1)

**Why it works:** any common divisor of `a` and `b` also divides `a % b`, so the pair can be shrunk without changing the answer until one side hits zero.

**Trap:** looping from `min(a,b)` downwards testing every candidate — O(n) instead of O(log n).

---

### 26. Least common multiple — *Medium*

```python
import math

def lcm(a: int, b: int) -> int:
    return abs(a * b) // math.gcd(a, b)
```

**Time:** O(log n) · **Space:** O(1)

**Trap:** using `/` instead of `//`, which returns a float and loses precision on large inputs. `math.lcm` exists in Python 3.9+.

---

### 27. Check if a number is a perfect number — *Medium*

```python
def is_perfect(n: int) -> bool:
    if n < 2:
        return False
    total = 1
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            total += i + (n // i if i != n // i else 0)
    return total == n
```

**Time:** O(√n) · **Space:** O(1)

A perfect number equals the sum of its **proper** divisors: 6 = 1+2+3, 28 = 1+2+4+7+14.

**Trap:** including `n` itself in the sum, and double-counting the square root when `n` is a perfect square.

---

### 28. Count the digits in a number — *Medium*

```python
len(str(abs(n)))
```

**Time:** O(d) · **Space:** O(d)

**Trap:** a `while n: n //= 10` loop returns 0 for input 0, but 0 has one digit. Guard it.

---

### 29. Swap two numbers without a temp variable — *Medium*

```python
a, b = b, a
```

**Time:** O(1) · **Space:** O(1)

Python builds a tuple on the right and unpacks it, so no explicit temporary is needed.

**Trap:** reaching for the XOR trick (`a ^= b; b ^= a; a ^= b`) — correct but unidiomatic, and it silently zeroes both if `a` and `b` are the same variable.

---

### 30. Check if a number is even or odd — *Very High*

```python
def is_even(n: int) -> bool:
    return n % 2 == 0
```

**Time:** O(1) · **Space:** O(1)

**Trap:** in Python `-7 % 2 == 1`, so `n % 2 == 1` *does* correctly detect odd negatives (unlike C, where it gives -1). Checking `n % 2 != 0` is the portable habit. Bitwise `n & 1` also works.

---

### 31. Find the largest of three numbers — *High*

```python
max(a, b, c)
```

**Time:** O(1) · **Space:** O(1)

**Trap:** writing a six-branch if/elif chain. If the interviewer bans `max`, nested comparison is fine — but say why you would normally use the builtin.

---

### 32. Multiplication table of a number — *Medium*

```python
for i in range(1, 11):
    print(f"{n} x {i} = {n * i}")
```

**Time:** O(1) — fixed 10 iterations · **Space:** O(1)

**Trap:** `range(1, 10)` stops at 9.

---

### 33. Check if a number is a power of two — *Medium*

```python
def is_power_of_two(n: int) -> bool:
    return n > 0 and n & (n - 1) == 0
```

**Time:** O(1) · **Space:** O(1)

A power of two has exactly one set bit, so `n - 1` flips it and all zeros below it — the AND is therefore 0.

**Trap:** omitting `n > 0`. Without it, `0 & -1 == 0` returns `True` for zero.

---

### 34. Sum of squares of the first n naturals — *Low*

```python
n * (n + 1) * (2 * n + 1) // 6      # O(1) closed form
sum(i * i for i in range(1, n + 1)) # O(n) loop
```

**Trap:** `/` gives a float. Use `//`.

---

### 35. Convert decimal to binary — *Medium*

```python
bin(n)[2:]          # strip the '0b' prefix
f"{n:b}"            # cleaner
f"{n:08b}"          # zero-padded to 8 bits
```

**Time:** O(log n) · **Space:** O(log n)

**Trap:** forgetting to strip `0b`. The f-string form avoids the issue.

---

### 36. Check if a year is a leap year — *Medium*

```python
def is_leap(y: int) -> bool:
    return y % 4 == 0 and (y % 100 != 0 or y % 400 == 0)
```

**Time:** O(1) · **Space:** O(1)

**Trap:** stopping at `y % 4`. 1900 is **not** a leap year (divisible by 100, not by 400); 2000 **is**. This exact edge case is why the question is asked.

---

### 37. Simple and compound interest — *Low*

```python
simple   = (p * r * t) / 100
compound = p * ((1 + r / 100) ** t) - p
```

**Trap:** integer division truncating the rate — use float literals.

---

## Lists

### 38. Sum all elements in a list — *Very High*

```python
sum(lst)
```

**Time:** O(n) · **Space:** O(1)

---

### 39. Find the largest and smallest in a list — *Very High*

```python
max(lst), min(lst)
max(lst, default=None)      # safe on empty input
```

**Time:** O(n) · **Space:** O(1)

**Trap:** `max([])` raises `ValueError`.

---

### 40. Find the second largest in a list — *Very High*

```python
def second_largest(lst: list[int]) -> int | None:
    first = second = float("-inf")
    for x in lst:
        if x > first:
            first, second = x, first
        elif first > x > second:
            second = x
    return None if second == float("-inf") else second
```

**Time:** O(n), single pass · **Space:** O(1)

**Trap:** `sorted(lst)[-2]` returns the same value as the largest when the maximum is duplicated — `[5, 5, 3]` gives 5, but the intended answer is usually 3. Clarify whether duplicates count, then use the distinct-tracking version above. `sorted(set(lst))[-2]` is an acceptable O(n log n) alternative.

---

### 41. Reverse a list — *High*

```python
rev = lst[::-1]      # new list, O(n) space
lst.reverse()        # in place, O(1) space, returns None
list(reversed(lst))  # iterator, lazy
```

**Time:** O(n)

**Trap:** `lst = lst.reverse()` sets `lst` to `None`. This is the most common Python interview bug there is — in-place methods return `None`.

---

### 42. Remove duplicates from a list — *Very High*

```python
list(dict.fromkeys(lst))   # preserves order  — preferred
list(set(lst))             # does NOT preserve order
```

**Time:** O(n) · **Space:** O(n)

**Trap:** using `set` when order matters, and `TypeError` when elements are unhashable (lists, dicts). For unhashable items you need an O(n²) scan or a key function.

---

### 43. Sort a list without using sort — *High*

```python
def bubble_sort(lst: list[int]) -> list[int]:
    a = lst[:]
    for i in range(len(a)):
        swapped = False
        for j in range(len(a) - i - 1):
            if a[j] > a[j + 1]:
                a[j], a[j + 1] = a[j + 1], a[j]
                swapped = True
        if not swapped:       # early exit on an already-sorted list
            break
    return a
```

**Time:** O(n²) worst and average, O(n) best with the early exit · **Space:** O(1) extra

**Trap:** omitting the `swapped` flag, and being unable to explain the O(n²): two nested loops each proportional to n.

---

### 44. Count element occurrences in a list — *High*

```python
from collections import Counter
Counter(lst)          # all counts at once
lst.count(x)          # a single element
```

**Time:** O(n) · **Space:** O(k)

**Trap:** `for x in lst: print(lst.count(x))` is O(n²) *and* prints duplicates. Use `Counter`.

---

### 45. Merge two sorted lists — *Very High*

```python
def merge(a: list[int], b: list[int]) -> list[int]:
    out, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            out.append(a[i]); i += 1
        else:
            out.append(b[j]); j += 1
    out.extend(a[i:]); out.extend(b[j:])
    return out
```

**Time:** O(n+m) · **Space:** O(n+m)

**Trap:** `sorted(a + b)` is O((n+m)log(n+m)) — it throws away the fact that the inputs are already sorted, which is the entire point of the question. Also: don't forget to append the leftover tail.

---

### 46. Sum of even and odd numbers in a list — *Medium*

```python
evens = sum(x for x in lst if x % 2 == 0)
odds  = sum(x for x in lst if x % 2)
```

**Time:** O(n) · **Space:** O(1)

---

### 47. Move all zeroes to the end — *High*

```python
def move_zeroes(lst: list[int]) -> None:
    pos = 0
    for i, x in enumerate(lst):
        if x != 0:
            lst[pos], lst[i] = lst[i], lst[pos]
            pos += 1
```

**Time:** O(n) · **Space:** O(1), in place

**Trap:** building a new list — usually the question requires in place. Relative order of non-zeros must be preserved, which the swap-forward approach guarantees.

---

### 48. Rotate a list by k positions — *High*

```python
def rotate(lst: list, k: int) -> list:
    if not lst:
        return lst
    k %= len(lst)
    return lst[k:] + lst[:k]      # left rotation
```

**Time:** O(n) · **Space:** O(n)

**Trap:** not reducing `k` modulo `n`, so `k > n` produces wrong output or an empty slice. Clarify left vs right rotation — right rotation is `lst[-k:] + lst[:-k]`.

---

### 49. Find the intersection of two lists — *High*

```python
list(set(a) & set(b))
[x for x in a if x in set(b)]     # preserves a's order
```

**Time:** O(n+m) · **Space:** O(n)

**Trap:** `[x for x in a if x in b]` with `b` as a *list* is O(n·m). Convert to a set first — and hoist the conversion out of the comprehension so it happens once.

---

### 50. Find missing numbers in a range — *High*

```python
missing = sorted(set(range(1, n + 1)) - set(lst))

def one_missing(lst: list[int], n: int) -> int:     # exactly one missing
    return n * (n + 1) // 2 - sum(lst)
```

**Time:** O(n) · **Space:** O(n), or O(1) for the sum trick.

**Trap:** not asking how many are missing. If exactly one, the Gauss sum formula is O(1) space and the answer they want.

---

### 51. Flatten a nested list — *High*

```python
flat = [x for sub in nested for x in sub]        # exactly one level

def deep_flatten(lst):                           # arbitrary depth
    for x in lst:
        if isinstance(x, (list, tuple)):
            yield from deep_flatten(x)
        else:
            yield x
```

**Time:** O(n) · **Space:** O(n)

**Trap:** offering only the comprehension when the input is arbitrarily nested — it flattens one level only. Ask about depth first.

---

### 52. Check if a list is sorted — *Medium*

```python
all(a <= b for a, b in zip(lst, lst[1:]))
```

**Time:** O(n), short-circuits on the first violation · **Space:** O(1) with `zip` on an iterator

**Trap:** `lst == sorted(lst)` is O(n log n) and allocates a copy.

---

### 53. Sum a matrix (list of lists) — *Medium*

```python
sum(map(sum, matrix))
```

**Time:** O(n) over all elements · **Space:** O(1)

---

### 54. Cumulative sum of a list — *Medium*

```python
from itertools import accumulate
list(accumulate(lst))
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `[sum(lst[:i+1]) for i in range(len(lst))]` is O(n²) — it re-adds the prefix every step.

---

### 55. Find all pairs summing to a target — *Very High*

```python
def pair_sum(lst: list[int], target: int) -> list[tuple[int, int]]:
    seen, out = set(), []
    for x in lst:
        if target - x in seen:
            out.append((target - x, x))
        seen.add(x)
    return out
```

**Time:** O(n) · **Space:** O(n)

**Trap:** nested loops (O(n²)); double-counting each pair; and letting an element pair with itself. Checking `seen` *before* inserting `x` prevents self-pairing for free. See `02-dsa-problem-solving.md` Q1 for the index-returning variant.

---

### 56. Split a list into equal chunks — *Medium*

```python
def chunks(lst: list, k: int) -> list[list]:
    return [lst[i:i + k] for i in range(0, len(lst), k)]
```

**Time:** O(n) · **Space:** O(n)

**Trap:** clarify what happens to a short final chunk — usually you keep it. Slicing past the end is safe in Python, which is why this is a one-liner.

---

### 57. Find the mode of a list — *Medium*

```python
from collections import Counter
Counter(lst).most_common(1)[0][0]
```

**Time:** O(n) · **Space:** O(k)

**Trap:** ties. `most_common` picks arbitrarily among equal counts — ask whether you should return all modes.

---

### 58. Interchange the first and last elements — *Low*

```python
if len(lst) >= 2:
    lst[0], lst[-1] = lst[-1], lst[0]
```

**Time:** O(1) · **Space:** O(1)

**Trap:** an empty list raises `IndexError`.

---

### 59. Multiply all numbers in a list — *Medium*

```python
import math
math.prod(lst)

from functools import reduce         # pre-3.8
reduce(lambda a, b: a * b, lst, 1)
```

**Time:** O(n) · **Space:** O(1)

**Trap:** initialising the accumulator to 0 — everything becomes 0.

---

### 60. Find array leaders (greater than everything to their right) — *Medium*

```python
def leaders(lst: list[int]) -> list[int]:
    out, running_max = [], float("-inf")
    for x in reversed(lst):
        if x > running_max:
            out.append(x)
            running_max = x
    return out[::-1]
```

**Time:** O(n) · **Space:** O(n) for output

**Trap:** the obvious nested loop is O(n²). Scanning right to left lets one running maximum answer every position.

---

## Dictionaries

### 61. Merge two dictionaries — *Very High*

```python
merged = a | b            # Python 3.9+
merged = {**a, **b}       # any version
a.update(b)               # in place, mutates a
```

**Time:** O(n) · **Space:** O(n)

**Trap:** keys in `b` silently overwrite `a`. If you need to detect collisions, check `a.keys() & b.keys()` first.

---

### 62. Sort a dictionary by value — *Very High*

```python
dict(sorted(d.items(), key=lambda kv: kv[1]))
dict(sorted(d.items(), key=lambda kv: kv[1], reverse=True))   # descending
```

**Time:** O(n log n) · **Space:** O(n)

**Trap:** sorting `d` directly sorts the *keys*. You must sort `d.items()` with a key function.

---

### 63. Invert a dictionary — *High*

```python
{v: k for k, v in d.items()}
```

**Time:** O(n) · **Space:** O(n)

**Trap:** duplicate values collapse — the last one wins and entries are silently lost. If values repeat, invert to lists:

```python
from collections import defaultdict
inv = defaultdict(list)
for k, v in d.items():
    inv[v].append(k)
```

---

### 64. Sum all values in a dictionary — *Medium*

```python
sum(d.values())
```

**Time:** O(n) · **Space:** O(1)

---

### 65. Find the key with the maximum value — *High*

```python
max(d, key=d.get)
```

**Time:** O(n) · **Space:** O(1)

**Trap:** `max(d)` returns the largest **key**, not the key of the largest value. The `key=d.get` is the whole answer.

---

### 66. Group items by a property — *High*

```python
from collections import defaultdict

groups = defaultdict(list)
for word in words:
    groups[len(word)].append(word)
```

**Time:** O(n) · **Space:** O(n)

**Trap:** a plain dict raises `KeyError` on first append. Use `defaultdict(list)` or `d.setdefault(k, []).append(x)`.

---

### 67. Count word frequency in a text — *Very High*

```python
import re
from collections import Counter

def word_freq(text: str) -> Counter:
    return Counter(re.findall(r"[a-z']+", text.lower()))
```

**Time:** O(n) · **Space:** O(k)

**Trap:** `text.split()` leaves punctuation attached, so `"cat."` and `"cat"` count separately — and not lowercasing splits `"The"` from `"the"`. Normalise first.

---

### 68. Check if two dictionaries are equal — *Low*

```python
a == b
```

**Time:** O(n) · **Space:** O(1)

Key insertion order does **not** affect equality.

---

### 69. Remove a key safely — *Medium*

```python
d.pop(key, None)      # no error if missing
```

**Time:** O(1) · **Space:** O(1)

**Trap:** `del d[key]` raises `KeyError` when absent.

---

### 70. Nested dictionary access with a default — *Medium*

```python
value = d.get("a", {}).get("b", "default")
```

**Time:** O(1) · **Space:** O(1)

**Trap:** `d["a"]["b"]` raises `KeyError`, and if `d["a"]` is `None` you get a `TypeError` instead. Chained `.get()` with `{}` fallbacks is the safe idiom.

---

## Stacks, queues and search

### 71. Check for balanced parentheses — *Very High*

```python
def is_balanced(s: str) -> bool:
    pairs = {")": "(", "]": "[", "}": "{"}
    stack = []
    for c in s:
        if c in "([{":
            stack.append(c)
        elif c in pairs:
            if not stack or stack.pop() != pairs[c]:
                return False
    return not stack
```

**Time:** O(n) · **Space:** O(n)

**Trap:** returning `True` without the final `not stack` check, so `"((("` passes. Also: popping from an empty stack when a closer arrives first.

---

### 72. Implement a stack using a list — *High*

```python
stack = []
stack.append(x)      # push — O(1)
top = stack.pop()    # pop  — O(1)
peek = stack[-1]     # peek — O(1)
```

**Trap:** `pop(0)` is O(n) and turns your stack into a queue. Push and pop from the **same** end.

---

### 73. Implement a queue — *High*

```python
from collections import deque

q = deque()
q.append(x)        # enqueue  — O(1)
first = q.popleft()  # dequeue — O(1)
```

**Trap:** `list.pop(0)` is O(n) because every remaining element shifts left — fine for 10 items, fatal for 100,000. `deque` is O(1) at both ends.

---

### 74. Binary search in a sorted list — *Very High*

```python
def binary_search(lst: list[int], target: int) -> int:
    lo, hi = 0, len(lst) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if lst[mid] == target:
            return mid
        if lst[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

**Time:** O(log n) — the range halves each iteration · **Space:** O(1)

**Trap:** `while lo < hi` (misses the single-element case), and forgetting the `± 1` so the loop never terminates. Python's `bisect` module does this in production.

---

### 75. Linear search — *Medium*

```python
def linear_search(lst: list, target) -> int:
    for i, x in enumerate(lst):
        if x == target:
            return i
    return -1
```

**Time:** O(n) · **Space:** O(1)

**Trap:** returning the value instead of the index, and forgetting `enumerate`.

---

### 76. FizzBuzz — *Very High*

```python
for i in range(1, 101):
    if i % 15 == 0:
        print("FizzBuzz")
    elif i % 3 == 0:
        print("Fizz")
    elif i % 5 == 0:
        print("Buzz")
    else:
        print(i)
```

**Time:** O(n) · **Space:** O(1)

**Trap:** checking 3 and 5 **before** 15, so 15 prints "Fizz" and exits the chain. Test the most specific condition first — that ordering is the entire point of the question.

---

## Sets

### 84. Check if a number appears twice in a list — *High*

```python
def has_duplicate(lst: list) -> bool:
    seen = set()
    for x in lst:
        if x in seen:
            return True
        seen.add(x)
    return False
```

**Time:** O(n) · **Space:** O(n)

**Trap:** nested loops give O(n²). Set membership is O(1) average.

---

### 85. Check if all list elements are unique — *High*

```python
len(set(lst)) == len(lst)
```

**Time:** O(n) · **Space:** O(n)

**Trap:** `TypeError` on unhashable elements. Note this version always scans everything, whereas problem 84 short-circuits on the first duplicate.

---

### 86. Union and difference of two lists — *Medium*

```python
set(a) | set(b)      # union
set(a) - set(b)      # in a but not b
set(a) ^ set(b)      # symmetric difference
```

**Time:** O(n+m) · **Space:** O(n)

**Trap:** sets do not preserve order, so wrap in `sorted()` if the output order matters.

---

### 87. Remove elements of one list from another — *Medium*

```python
b_set = set(b)
result = [x for x in a if x not in b_set]     # keeps a's order
```

**Time:** O(n+m) · **Space:** O(m)

**Trap:** mutating a list while iterating over it skips elements — build a new list instead.

---

## Recursion

### 88. Fibonacci with memoisation — *High*

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

**Time:** O(n) — each value computed once · **Space:** O(n) for the cache and the stack.

**Trap:** forgetting the decorator leaves it O(2ⁿ). This problem exists to test whether you can identify overlapping subproblems — say the words "memoisation turns the exponential recursion tree into a linear one."

---

### 89. Sum of digits recursively — *Medium*

```python
def digit_sum(n: int) -> int:
    n = abs(n)
    return n if n < 10 else n % 10 + digit_sum(n // 10)
```

**Time:** O(d) · **Space:** O(d) call stack

**Trap:** a missing or wrong base case gives infinite recursion and `RecursionError`.

---

### 90. Reverse a string recursively — *Medium*

```python
def rev(s: str) -> str:
    return s if len(s) <= 1 else rev(s[1:]) + s[0]
```

**Time:** **O(n²)** — each call slices a new string · **Space:** O(n²) total allocation, O(n) stack depth.

**Trap:** claiming O(n). Slicing copies, so the cost is quadratic — mentioning this unprompted is what the interviewer is listening for. `s[::-1]` is the real answer.

---

### 91. Tower of Hanoi — *Medium*

```python
def hanoi(n: int, src: str = "A", dst: str = "C", aux: str = "B") -> None:
    if n == 0:
        return
    hanoi(n - 1, src, aux, dst)
    print(f"Move disk {n} from {src} to {dst}")
    hanoi(n - 1, aux, dst, src)
```

**Time:** O(2ⁿ) — 2ⁿ−1 moves, which is provably minimal · **Space:** O(n) stack depth

**Trap:** swapping the auxiliary and destination pegs in the recursive calls. The structure is always: move n−1 out of the way, move the big disk, move n−1 back on top.

---

### 92. Fast exponentiation — *Medium*

```python
def power(base: float, exp: int) -> float:
    if exp < 0:
        return 1 / power(base, -exp)
    if exp == 0:
        return 1
    half = power(base, exp // 2)
    return half * half * (base if exp % 2 else 1)
```

**Time:** O(log n) — the exponent halves each call · **Space:** O(log n)

**Trap:** not handling negative or zero exponents, and recomputing `power(base, exp//2)` twice instead of storing it (which would make it O(n)).

---

### 93. Flatten a nested dictionary — *High*

```python
def flatten(d: dict, prefix: str = "", sep: str = ".") -> dict:
    out = {}
    for k, v in d.items():
        key = f"{prefix}{sep}{k}" if prefix else str(k)
        if isinstance(v, dict):
            out.update(flatten(v, key, sep))
        else:
            out[key] = v
    return out

# {"a": {"b": 1, "c": {"d": 2}}} -> {"a.b": 1, "a.c.d": 2}
```

**Time:** O(n) · **Space:** O(n)

**Trap:** not handling dicts nested inside **lists** — ask whether that case exists. This one comes up genuinely often in data-engineering and API-integration roles.

---

## Sorting

### 94. Bubble sort — *High*

See problem 43. **O(n²)** average, **O(n)** best with early exit, O(1) space, stable.

---

### 95. Insertion sort — *Medium*

```python
def insertion_sort(a: list[int]) -> list[int]:
    a = a[:]
    for i in range(1, len(a)):
        key, j = a[i], i - 1
        while j >= 0 and a[j] > key:
            a[j + 1] = a[j]
            j -= 1
        a[j + 1] = key
    return a
```

**Time:** O(n²) average, **O(n) best on nearly-sorted input** · **Space:** O(1) · **Stable:** yes

**Trap:** not knowing its best case is linear — that is exactly why real sort implementations (including Python's Timsort) use insertion sort for small or nearly-sorted runs.

---

### 96. Selection sort — *Medium*

```python
def selection_sort(a: list[int]) -> list[int]:
    a = a[:]
    for i in range(len(a)):
        m = min(range(i, len(a)), key=a.__getitem__)
        a[i], a[m] = a[m], a[i]
    return a
```

**Time:** O(n²) in all cases — no early exit possible · **Space:** O(1) · **Stable:** **no**

**Trap:** claiming it is stable; the long-distance swap can reorder equal elements. Its one virtue is making the minimum number of swaps (n−1).

---

### 97. Merge sort — *High*

```python
def merge_sort(a: list[int]) -> list[int]:
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    return merge(merge_sort(a[:mid]), merge_sort(a[mid:]))   # merge from problem 45
```

**Time:** O(n log n) guaranteed — log n levels, O(n) merging per level · **Space:** O(n) · **Stable:** yes

**Trap:** forgetting the O(n) auxiliary space, which is merge sort's main cost versus quicksort.

---

### 98. Quick sort — *High*

```python
def quick_sort(a: list[int]) -> list[int]:
    if len(a) <= 1:
        return a
    pivot = a[len(a) // 2]
    left  = [x for x in a if x < pivot]
    mid   = [x for x in a if x == pivot]
    right = [x for x in a if x > pivot]
    return quick_sort(left) + mid + quick_sort(right)
```

**Time:** O(n log n) average, **O(n²) worst** when the pivot is consistently extreme · **Space:** O(log n) stack for the in-place version, O(n) here.

**Trap:** not knowing the worst case, or that choosing the *first* element as pivot makes already-sorted input hit it. Random or median-of-three pivoting mitigates it.

---

## Patterns, files and bits

### 77–80. Star and number patterns — *High*

```python
n = 5
for i in range(1, n + 1):                    # 77: right triangle
    print("*" * i)

for i in range(1, n + 1):                    # 78: pyramid
    print(" " * (n - i) + "*" * (2 * i - 1))

for i in range(n, 0, -1):                    # 79: inverted
    print("*" * i)

row = [1]                                    # 80: Pascal's triangle
for _ in range(n):
    print(row)
    row = [1] + [row[j] + row[j + 1] for j in range(len(row) - 1)] + [1]
```

**Time:** O(n²) — total characters printed · **Space:** O(1) (O(n) for Pascal's row)

**Trap:** off-by-one in `range` bounds, and miscounting pyramid spaces. The pyramid formula is `n - i` spaces then `2i - 1` stars — derive it rather than memorising it.

---

### 81. Read a file and count lines and words — *High*

```python
with open("file.txt", encoding="utf-8") as f:
    lines = words = 0
    for line in f:                 # streams — does not load the whole file
        lines += 1
        words += len(line.split())
```

**Time:** O(n) · **Space:** O(1) — one line at a time

**Trap:** `f.readlines()` loads the entire file into memory, and omitting the `with` block leaks the handle. Iterating the file object directly is both idiomatic and memory-safe.

---

### 82. Most frequent word in a large file — *Medium*

```python
import re
from collections import Counter

freq = Counter()
with open("file.txt", encoding="utf-8") as f:
    for line in f:
        freq.update(re.findall(r"[a-z']+", line.lower()))
print(freq.most_common(5))
```

**Time:** O(n) · **Space:** O(k) distinct words

**Trap:** `f.read()` on a multi-gigabyte file. Stream it and update the `Counter` incrementally — the same reasoning as problem 81.

---

### 83. Write a list to a file line by line — *Medium*

```python
with open("out.txt", "w", encoding="utf-8") as f:
    f.write("\n".join(map(str, items)))
    # or: f.writelines(f"{x}\n" for x in items)
```

**Trap:** `writelines` does **not** add newlines — you must include them yourself.

---

### 99. Count set bits in a number — *Medium*

```python
bin(n).count("1")
n.bit_count()              # Python 3.10+

def popcount(n: int) -> int:      # Brian Kernighan's algorithm
    count = 0
    while n:
        n &= n - 1             # clears the lowest set bit
        count += 1
    return count
```

**Time:** O(log n), or O(number of set bits) for Kernighan · **Space:** O(1)

**Trap:** not knowing the `n & (n-1)` trick — it clears the lowest set bit, so the loop runs once per set bit rather than once per bit.

---

### 100. Find the single number in a list of pairs — *High*

```python
from functools import reduce
import operator

def single_number(lst: list[int]) -> int:
    return reduce(operator.xor, lst)
```

**Time:** O(n) · **Space:** **O(1)**

Every duplicate cancels itself because `x ^ x == 0`, and `0 ^ y == y`, so only the unpaired value survives. XOR is commutative and associative, so order does not matter.

**Trap:** reaching for a `Counter` — correct, but O(n) space. The XOR solution is what the question is testing, and the constant-space claim is the interesting part.

---

## Quick revision checklist

If you only drill twenty, drill these — they are the highest-frequency and cover the most patterns:

| # | Problem | Why it matters |
|---|---|---|
| 2 | Palindrome | Two pointers plus input normalisation |
| 4 | Anagrams | `Counter` and the O(n) vs O(n log n) trade-off |
| 7 | First non-repeating char | Two-pass counting |
| 16 | Factorial | Iteration vs recursion limits |
| 17 | Fibonacci | The O(2ⁿ) → O(n) insight |
| 19 | Prime check | Why √n suffices |
| 21 | Armstrong number | Reading the definition carefully |
| 23 | Reverse digits | `divmod` and sign handling |
| 25 | GCD | Euclid's algorithm |
| 30 | Even/odd | Negative modulo in Python |
| 40 | Second largest | Single-pass tracking, duplicate handling |
| 42 | Remove duplicates | Order-preserving dedupe idiom |
| 45 | Merge sorted lists | Two pointers; the merge-sort building block |
| 55 | Pair sum | Hash-set complement lookup |
| 62 | Sort dict by value | `key=` functions |
| 67 | Word frequency | Normalisation plus `Counter` |
| 71 | Balanced parentheses | The canonical stack problem |
| 74 | Binary search | Loop bounds without off-by-one errors |
| 76 | FizzBuzz | Condition ordering |
| 100 | Single number (XOR) | Constant-space bit trick |

For the next level up — sliding window, BFS/DFS, heaps, topological sort — continue to [`02-dsa-problem-solving.md`](02-dsa-problem-solving.md).
