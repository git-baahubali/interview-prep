# Easy Python Problems (Warm-up Bank)

The 100 classic beginner Python problems that appear in screening rounds, phone screens, and the first ten minutes of a coding interview — Armstrong numbers, palindromes, Fibonacci, prime checks, FizzBuzz, and the rest.

Every problem has four things: the **logic** in plain words, **working code**, **complexity with the reason**, and the **trap** interviewers watch for. Most of these are failed on an edge case, not on the core idea — so the trap line is the part worth memorising.

**Problems:** 100

For the pattern-based interview problems (two pointers, sliding window, BFS/DFS, heaps, topological sort) see [`02-dsa-problem-solving.md`](02-dsa-problem-solving.md).

**How to use this:** cover the code, write it yourself from the logic line, then compare. If you can do the *Very High* frequency rows cold in under two minutes each, you are ready for any screening round.

---

## The five things that actually fail these problems

1. **Not normalising input.** Palindrome and anagram questions almost always intend case-insensitive, punctuation-stripped comparison. Ask, then handle it.
2. **Empty and single-element input.** `max([])` raises `ValueError`. Every solution should survive `""`, `[]`, and `0`.
3. **Negative numbers.** In Python `-7 % 2 == 1`, not `-1`. Digit-sum and digit-reversal problems need `abs()`.
4. **In-place methods return `None`.** `lst.reverse()`, `lst.sort()`, `lst.append()` all return `None`, so `x = lst.sort()` gives you `None`. This is the single most common Python interview bug.
5. **Accidentally quadratic code.** `lst.count(x)` inside a loop over `lst`, or `x in some_list` inside a loop, is O(n²). Reach for `Counter` and `set`.

---

## Index

Frequency is how often the problem shows up in screening rounds. Drill **Very High** first.

| # | Problem | Category | Frequency | Time | Space |
|---|---|---|---|---|---|
| [1](#1-reverse-a-string) | Reverse a string | String | Very High | O(n) | O(n) |
| [2](#2-check-if-a-string-is-a-palindrome) | Check if a string is a palindrome | String | Very High | O(n) | O(1) |
| [3](#3-count-vowels-in-a-string) | Count vowels in a string | String | High | O(n) | O(1) |
| [4](#4-check-if-two-strings-are-anagrams) | Check if two strings are anagrams | String | Very High | O(n) | O(k) |
| [5](#5-count-character-frequency) | Count character frequency | String | High | O(n) | O(k) |
| [6](#6-remove-duplicate-characters-keeping-order) | Remove duplicate characters, keeping order | String | Medium | O(n) | O(n) |
| [7](#7-find-the-first-non-repeating-character) | Find the first non-repeating character | String | High | O(n) | O(k) |
| [8](#8-capitalise-the-first-letter-of-each-word) | Capitalise the first letter of each word | String | Medium | O(n) | O(n) |
| [9](#9-count-words-in-a-sentence) | Count words in a sentence | String | Medium | O(n) | O(n) |
| [10](#10-reverse-the-word-order-in-a-sentence) | Reverse the word order in a sentence | String | High | O(n) | O(n) |
| [11](#11-check-if-a-string-contains-only-digits) | Check if a string contains only digits | String | Low | O(n) | O(1) |
| [12](#12-longest-word-in-a-sentence) | Longest word in a sentence | String | Medium | O(n) | O(n) |
| [13](#13-remove-all-whitespace) | Remove all whitespace | String | Low | O(n) | O(n) |
| [14](#14-check-if-a-string-is-a-pangram) | Check if a string is a pangram | String | Low | O(n) | O(1) |
| [15](#15-toggle-the-case-of-each-character) | Toggle the case of each character | String | Low | O(n) | O(n) |
| [16](#16-factorial-of-a-number) | Factorial of a number | Number | Very High | O(n) | O(1) |
| [17](#17-nth-fibonacci-number) | Nth Fibonacci number | Number | Very High | O(n) | O(1) |
| [18](#18-print-the-fibonacci-series-up-to-n-terms) | Print the Fibonacci series up to n terms | Number | Very High | O(n) | O(n) |
| [19](#19-check-if-a-number-is-prime) | Check if a number is prime | Number | Very High | O(√n) | O(1) |
| [20](#20-print-all-primes-in-an-interval) | Print all primes in an interval | Number | High | O(n log log n) | O(n) |
| [21](#21-check-if-a-number-is-an-armstrong-number) | Check if a number is an Armstrong number | Number | Very High | O(d) | O(1) |
| [22](#22-sum-the-digits-of-a-number) | Sum the digits of a number | Number | High | O(d) | O(1) |
| [23](#23-reverse-the-digits-of-a-number) | Reverse the digits of a number | Number | High | O(d) | O(1) |
| [24](#24-check-if-a-number-is-a-palindrome) | Check if a number is a palindrome | Number | High | O(d) | O(d) |
| [25](#25-greatest-common-divisor) | Greatest common divisor | Number | High | O(log n) | O(1) |
| [26](#26-least-common-multiple) | Least common multiple | Number | Medium | O(log n) | O(1) |
| [27](#27-check-if-a-number-is-a-perfect-number) | Check if a number is a perfect number | Number | Medium | O(√n) | O(1) |
| [28](#28-count-the-digits-in-a-number) | Count the digits in a number | Number | Medium | O(d) | O(d) |
| [29](#29-swap-two-numbers-without-a-temp-variable) | Swap two numbers without a temp variable | Number | Medium | O(1) | O(1) |
| [30](#30-check-if-a-number-is-even-or-odd) | Check if a number is even or odd | Number | Very High | O(1) | O(1) |
| [31](#31-find-the-largest-of-three-numbers) | Find the largest of three numbers | Number | High | O(1) | O(1) |
| [32](#32-multiplication-table-of-a-number) | Multiplication table of a number | Number | Medium | O(1) | O(1) |
| [33](#33-check-if-a-number-is-a-power-of-two) | Check if a number is a power of two | Number | Medium | O(1) | O(1) |
| [34](#34-sum-of-squares-of-the-first-n-naturals) | Sum of squares of the first n naturals | Number | Low | O(1) | O(1) |
| [35](#35-convert-decimal-to-binary) | Convert decimal to binary | Number | Medium | O(log n) | O(log n) |
| [36](#36-check-if-a-year-is-a-leap-year) | Check if a year is a leap year | Number | Medium | O(1) | O(1) |
| [37](#37-simple-and-compound-interest) | Simple and compound interest | Number | Low | O(1) | O(1) |
| [38](#38-sum-all-elements-in-a-list) | Sum all elements in a list | List | Very High | O(n) | O(1) |
| [39](#39-find-the-largest-and-smallest-in-a-list) | Find the largest and smallest in a list | List | Very High | O(n) | O(1) |
| [40](#40-find-the-second-largest-in-a-list) | Find the second largest in a list | List | Very High | O(n) | O(1) |
| [41](#41-reverse-a-list) | Reverse a list | List | High | O(n) | O(n) / O(1) |
| [42](#42-remove-duplicates-from-a-list) | Remove duplicates from a list | List | Very High | O(n) | O(n) |
| [43](#43-sort-a-list-without-using-sort) | Sort a list without using sort | List | High | O(n²) | O(1) |
| [44](#44-count-element-occurrences-in-a-list) | Count element occurrences in a list | List | High | O(n) | O(k) |
| [45](#45-merge-two-sorted-lists) | Merge two sorted lists | List | Very High | O(n+m) | O(n+m) |
| [46](#46-sum-of-even-and-odd-numbers-in-a-list) | Sum of even and odd numbers in a list | List | Medium | O(n) | O(1) |
| [47](#47-move-all-zeroes-to-the-end) | Move all zeroes to the end | List | High | O(n) | O(1) |
| [48](#48-rotate-a-list-by-k-positions) | Rotate a list by k positions | List | High | O(n) | O(n) |
| [49](#49-find-the-intersection-of-two-lists) | Find the intersection of two lists | List | High | O(n+m) | O(n) |
| [50](#50-find-missing-numbers-in-a-range) | Find missing numbers in a range | List | High | O(n) | O(n) |
| [51](#51-flatten-a-nested-list) | Flatten a nested list | List | High | O(n) | O(n) |
| [52](#52-check-if-a-list-is-sorted) | Check if a list is sorted | List | Medium | O(n) | O(1) |
| [53](#53-sum-a-matrix-list-of-lists) | Sum a matrix (list of lists) | List | Medium | O(n) | O(1) |
| [54](#54-cumulative-sum-of-a-list) | Cumulative sum of a list | List | Medium | O(n) | O(n) |
| [55](#55-find-all-pairs-summing-to-a-target) | Find all pairs summing to a target | List | Very High | O(n) | O(n) |
| [56](#56-split-a-list-into-equal-chunks) | Split a list into equal chunks | List | Medium | O(n) | O(n) |
| [57](#57-find-the-mode-of-a-list) | Find the mode of a list | List | Medium | O(n) | O(k) |
| [58](#58-interchange-the-first-and-last-elements) | Interchange the first and last elements | List | Low | O(1) | O(1) |
| [59](#59-multiply-all-numbers-in-a-list) | Multiply all numbers in a list | List | Medium | O(n) | O(1) |
| [60](#60-find-array-leaders) | Find array leaders | List | Medium | O(n) | O(n) |
| [61](#61-merge-two-dictionaries) | Merge two dictionaries | Dict | Very High | O(n) | O(n) |
| [62](#62-sort-a-dictionary-by-value) | Sort a dictionary by value | Dict | Very High | O(n log n) | O(n) |
| [63](#63-invert-a-dictionary) | Invert a dictionary | Dict | High | O(n) | O(n) |
| [64](#64-sum-all-values-in-a-dictionary) | Sum all values in a dictionary | Dict | Medium | O(n) | O(1) |
| [65](#65-find-the-key-with-the-maximum-value) | Find the key with the maximum value | Dict | High | O(n) | O(1) |
| [66](#66-group-items-by-a-property) | Group items by a property | Dict | High | O(n) | O(n) |
| [67](#67-count-word-frequency-in-a-text) | Count word frequency in a text | Dict | Very High | O(n) | O(k) |
| [68](#68-check-if-two-dictionaries-are-equal) | Check if two dictionaries are equal | Dict | Low | O(n) | O(1) |
| [69](#69-remove-a-key-safely) | Remove a key safely | Dict | Medium | O(1) | O(1) |
| [70](#70-nested-dictionary-access-with-a-default) | Nested dictionary access with a default | Dict | Medium | O(1) | O(1) |
| [71](#71-check-for-balanced-parentheses) | Check for balanced parentheses | Stack | Very High | O(n) | O(n) |
| [72](#72-implement-a-stack-using-a-list) | Implement a stack using a list | Stack | High | O(1) | O(n) |
| [73](#73-implement-a-queue) | Implement a queue | Queue | High | O(1) | O(n) |
| [74](#74-binary-search-in-a-sorted-list) | Binary search in a sorted list | Search | Very High | O(log n) | O(1) |
| [75](#75-linear-search) | Linear search | Search | Medium | O(n) | O(1) |
| [76](#76-fizzbuzz) | FizzBuzz | Logic | Very High | O(n) | O(1) |
| [77](#77-right-angled-star-triangle) | Right-angled star triangle | Pattern | High | O(n²) | O(1) |
| [78](#78-pyramid-star-pattern) | Pyramid star pattern | Pattern | High | O(n²) | O(1) |
| [79](#79-inverted-star-pattern) | Inverted star pattern | Pattern | Medium | O(n²) | O(1) |
| [80](#80-pascals-triangle) | Pascal's triangle | Pattern | Medium | O(n²) | O(n) |
| [81](#81-read-a-file-and-count-lines-and-words) | Read a file and count lines and words | File | High | O(n) | O(1) |
| [82](#82-most-frequent-word-in-a-large-file) | Most frequent word in a large file | File | Medium | O(n) | O(k) |
| [83](#83-write-a-list-to-a-file-line-by-line) | Write a list to a file line by line | File | Medium | O(n) | O(1) |
| [84](#84-check-if-a-value-appears-twice-in-a-list) | Check if a value appears twice in a list | Set | High | O(n) | O(n) |
| [85](#85-check-if-all-list-elements-are-unique) | Check if all list elements are unique | Set | High | O(n) | O(n) |
| [86](#86-union-and-difference-of-two-lists) | Union and difference of two lists | Set | Medium | O(n+m) | O(n) |
| [87](#87-remove-elements-of-one-list-from-another) | Remove elements of one list from another | Set | Medium | O(n+m) | O(m) |
| [88](#88-fibonacci-with-memoisation) | Fibonacci with memoisation | Recursion | High | O(n) | O(n) |
| [89](#89-sum-of-digits-recursively) | Sum of digits recursively | Recursion | Medium | O(d) | O(d) |
| [90](#90-reverse-a-string-recursively) | Reverse a string recursively | Recursion | Medium | O(n²) | O(n) |
| [91](#91-tower-of-hanoi) | Tower of Hanoi | Recursion | Medium | O(2ⁿ) | O(n) |
| [92](#92-fast-exponentiation) | Fast exponentiation | Recursion | Medium | O(log n) | O(log n) |
| [93](#93-flatten-a-nested-dictionary) | Flatten a nested dictionary | Recursion | High | O(n) | O(n) |
| [94](#94-bubble-sort) | Bubble sort | Sorting | High | O(n²) | O(1) |
| [95](#95-insertion-sort) | Insertion sort | Sorting | Medium | O(n²) | O(1) |
| [96](#96-selection-sort) | Selection sort | Sorting | Medium | O(n²) | O(1) |
| [97](#97-merge-sort) | Merge sort | Sorting | High | O(n log n) | O(n) |
| [98](#98-quick-sort) | Quick sort | Sorting | High | O(n log n) avg | O(log n) |
| [99](#99-count-set-bits-in-a-number) | Count set bits in a number | Bit | Medium | O(log n) | O(1) |
| [100](#100-find-the-single-number-in-a-list-of-pairs) | Find the single number in a list of pairs | Bit | High | O(n) | O(1) |

---

## Strings

### 1. Reverse a string

*String · Very High*

**Logic:** a slice with a step of `-1` walks the string from the end to the start, producing a new reversed string. There is no in-place option because Python strings are immutable.

```python
def reverse(s: str) -> str:
    return s[::-1]
```

**Time:** O(n) — every character is copied once.
**Space:** O(n) — a new string is allocated; you cannot reverse a `str` in place.
**Trap:** claiming an O(1)-space in-place reversal. If the interviewer wants that, they mean a *list* of characters.

---

### 2. Check if a string is a palindrome

*String · Very High*

**Logic:** strip out anything that is not a letter or digit, lowercase the rest, then compare the result to its own reverse. The two-pointer version does the same comparison without building a second string: walk one index in from each end, skipping non-alphanumerics, and fail the moment a pair disagrees.

```python
def is_palindrome(s: str) -> bool:
    cleaned = [c.lower() for c in s if c.isalnum()]
    return cleaned == cleaned[::-1]

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

**Time:** O(n) — each pointer only ever moves forward, so the total work is linear even with the inner skip loops.
**Space:** O(n) for the slice version, **O(1)** for two pointers.
**Trap:** forgetting to case-fold and strip punctuation. `"A man, a plan, a canal: Panama"` is the canonical test and it fails a naive `s == s[::-1]`.

---

### 3. Count vowels in a string

*String · High*

**Logic:** lowercase once so you only test five characters, then sum a membership test over every character — Python treats `True` as 1 and `False` as 0, so `sum()` over booleans is a count.

```python
def count_vowels(s: str) -> int:
    return sum(c in "aeiou" for c in s.lower())
```

**Time:** O(n) · **Space:** O(1) — the vowel set is a fixed size.
**Trap:** not lowercasing, so `"AEIOU"` counts as zero.

---

### 4. Check if two strings are anagrams

*String · Very High*

**Logic:** two strings are anagrams exactly when they use the same characters the same number of times, so compare their character counts. `Counter` builds both tallies in one linear pass each and compares them as dicts.

```python
from collections import Counter

def is_anagram(a: str, b: str) -> bool:
    return Counter(a) == Counter(b)

# case- and space-insensitive variant
def is_anagram_loose(a: str, b: str) -> bool:
    norm = lambda s: Counter(s.lower().replace(" ", ""))
    return norm(a) == norm(b)
```

**Time:** O(n) · **Space:** O(k) for k distinct characters.
**Trap:** offering only `sorted(a) == sorted(b)`. It is correct but O(n log n) — know that it is slower and say so. Also ask whether spaces and case matter.

---

### 5. Count character frequency

*String · High*

**Logic:** a single pass tallying each character into a dict. `Counter` is that loop, written for you.

```python
from collections import Counter

freq = Counter("mississippi")   # {'i': 4, 's': 4, 'p': 2, 'm': 1}
```

**Time:** O(n) · **Space:** O(k)
**Trap:** hand-rolling `if c in d: d[c] += 1 else: d[c] = 1`. It works, but `Counter` or `defaultdict(int)` is what a Python engineer writes.

---

### 6. Remove duplicate characters, keeping order

*String · Medium*

**Logic:** dict keys are unique *and*, since Python 3.7, keep insertion order — so building a dict from the characters drops duplicates while preserving first-appearance order, and joining the keys rebuilds the string.

```python
def dedupe(s: str) -> str:
    return "".join(dict.fromkeys(s))

dedupe("mississippi")   # 'misp'
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `"".join(set(s))` — a set has no order, so the output comes back scrambled.

---

### 7. Find the first non-repeating character

*String · High*

**Logic:** you cannot know a character is unique until you have seen the whole string, so count everything first. Then scan the *original* string in order and return the first character whose count is 1 — scanning the string rather than the counter is what makes the answer "first".

```python
from collections import Counter

def first_unique(s: str) -> str | None:
    freq = Counter(s)
    return next((c for c in s if freq[c] == 1), None)

first_unique("swiss")   # 'w'
```

**Time:** O(n) — two sequential passes, not nested · **Space:** O(k)
**Trap:** a nested loop calling `s.count(c)` per character, which is O(n²). Also: return `None` rather than crashing when every character repeats.

---

### 8. Capitalise the first letter of each word

*String · Medium*

**Logic:** split on whitespace, uppercase the first character of each piece, join back with single spaces.

```python
" ".join(w.capitalize() for w in s.split())
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `str.title()` treats any non-letter as a word boundary, so `"o'brien".title()` gives `"O'Brien"` and `"don't".title()` gives `"Don'T"`. Use `capitalize()` per word.

---

### 9. Count words in a sentence

*String · Medium*

**Logic:** `split()` with no argument treats any run of whitespace as one separator and discards empties, so the length of the result is the word count.

```python
len(s.split())
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `s.split(" ")` splits on each *single* space, so `"a  b"` yields `['a', '', 'b']` — a phantom word. Bare `split()` also handles tabs and newlines.

---

### 10. Reverse the word order in a sentence

*String · High*

**Logic:** split into words, reverse the list, join with spaces. The characters inside each word stay untouched.

```python
" ".join(s.split()[::-1])

"the sky is blue"   # -> 'blue is sky the'
```

**Time:** O(n) · **Space:** O(n)
**Trap:** reversing characters (`s[::-1]`) instead of words. Read the question twice.

---

### 11. Check if a string contains only digits

*String · Low*

**Logic:** `str.isdigit()` returns `True` only when every character is a digit and the string is non-empty.

```python
s.isdigit()      # "123" -> True,  "" -> False,  "-1" -> False,  "1.5" -> False
```

**Time:** O(n) · **Space:** O(1)
**Trap:** `isdigit()` is also `True` for superscripts like `"²"`, and `False` for negatives and decimals. For "is this a valid number", use a `try: float(s)` block instead.

---

### 12. Longest word in a sentence

*String · Medium*

**Logic:** `max` with `key=len` compares words by length rather than alphabetically, returning the first longest on a tie.

```python
max(s.split(), key=len, default="")
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `max()` on an empty sequence raises `ValueError` — pass `default=`.

---

### 13. Remove all whitespace

*String · Low*

**Logic:** `split()` discards every whitespace run, so joining the pieces with an empty string removes all of it — internal as well as external.

```python
"".join(s.split())
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `s.strip()` only removes *leading and trailing* whitespace.

---

### 14. Check if a string is a pangram

*String · Low*

**Logic:** a pangram uses every letter at least once, so the alphabet must be a subset of the string's letters. `<=` on sets is the subset operator.

```python
from string import ascii_lowercase

def is_pangram(s: str) -> bool:
    return set(ascii_lowercase) <= set(s.lower())
```

**Time:** O(n) · **Space:** O(1) — the alphabet is a fixed 26 characters.
**Trap:** comparing the sets for equality instead of subset; extra punctuation and digits in the string would then fail it.

---

### 15. Toggle the case of each character

*String · Low*

**Logic:** one builtin already does it per character.

```python
s.swapcase()      # "Hello World" -> 'hELLO wORLD'
```

**Time:** O(n) · **Space:** O(n)

---

## Numbers

### 16. Factorial of a number

*Number · Very High*

**Logic:** multiply every integer from 2 up to n into a running product. Start the accumulator at 1 so that `0!` and `1!` both come out as 1 without a special case.

```python
import math
math.factorial(n)

def factorial(n: int) -> int:
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

**Time:** O(n) · **Space:** O(1) iteratively, O(n) recursively for the call stack.
**Trap:** `0! == 1` (an empty product), and the recursive version hits Python's ~1000-frame recursion limit. Mention both.

---

### 17. Nth Fibonacci number

*Number · Very High*

**Logic:** each term needs only the previous two, so you never need the whole series in memory. Hold two variables and roll them forward n times — `a, b = b, a + b` builds the tuple on the right before unpacking, so no temporary is needed.

```python
def fib(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

**Time:** O(n) · **Space:** O(1)
**Trap:** naive recursion is **O(2ⁿ)** because it recomputes the same subproblems exponentially often — `fib(40)` takes seconds and `fib(50)` effectively never finishes. Either iterate as above or memoise ([88](#88-fibonacci-with-memoisation)). Being able to state the exponential blow-up *is* the question.

---

### 18. Print the Fibonacci series up to n terms

*Number · Very High*

**Logic:** identical rolling pair to [17](#17-nth-fibonacci-number), but collect each value as you pass it instead of returning only the last.

```python
def fib_series(n: int) -> list[int]:
    out, a, b = [], 0, 1
    for _ in range(n):
        out.append(a)
        a, b = b, a + b
    return out

fib_series(10)   # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

**Time:** O(n) · **Space:** O(n) for the output.
**Trap:** off-by-one — clarify whether "up to n" means n *terms* or values *below* n.

---

### 19. Check if a number is prime

*Number · Very High*

**Logic:** test for a divisor. You only need to check up to √n, because if `n = a × b` then one factor must be ≤ √n — a factor above it would already have been found as the smaller partner. Handle 2 separately, then step by 2 to skip all even candidates.

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
**Trap:** looping all the way to `n` (far too slow), and mishandling `n < 2` — 0, 1 and every negative are not prime. Being able to *explain* why √n suffices is what separates an understood answer from a memorised one.

---

### 20. Print all primes in an interval

*Number · High*

**Logic:** the Sieve of Eratosthenes. Assume everything is prime, then walk up from 2 and cross out every multiple of each surviving number. Start crossing out at `i*i` because smaller multiples of `i` already had a smaller factor and were struck off earlier.

```python
def sieve(limit: int) -> list[int]:
    is_p = [True] * (limit + 1)
    is_p[0:2] = [False, False]
    for i in range(2, int(limit ** 0.5) + 1):
        if is_p[i]:
            is_p[i*i::i] = [False] * len(is_p[i*i::i])
    return [i for i, p in enumerate(is_p) if p]

sieve(31)   # [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31]
```

**Time:** O(n log log n) · **Space:** O(n)
**Trap:** calling `is_prime()` on every number in the range — correct, but O(n√n). The sieve shares work across candidates instead of restarting for each.

---

### 21. Check if a number is an Armstrong number

*Number · Very High*

**Logic:** an Armstrong (narcissistic) number equals the sum of its digits each raised to the power of **the number of digits**. So count the digits first, then use that count as the exponent: 153 = 1³+5³+3³, and 9474 = 9⁴+4⁴+7⁴+4⁴.

```python
def is_armstrong(n: int) -> bool:
    digits = str(n)
    power = len(digits)
    return n == sum(int(d) ** power for d in digits)

[n for n in range(1, 1000) if is_armstrong(n)]
# [1..9, 153, 370, 371, 407]
```

**Time:** O(d) for d digits · **Space:** O(1)
**Trap:** hardcoding `** 3`. That only works for 3-digit numbers and is precisely the mistake this question exists to catch — the exponent must be the digit count.

---

### 22. Sum the digits of a number

*Number · High*

**Logic:** either treat the number as text and add up its characters, or peel digits off arithmetically with `divmod(n, 10)`, which returns the quotient and remainder together.

```python
def digit_sum(n: int) -> int:
    return sum(int(d) for d in str(abs(n)))

def digit_sum_math(n: int) -> int:
    n, total = abs(n), 0
    while n:
        n, r = divmod(n, 10)
        total += r
    return total
```

**Time:** O(d) · **Space:** O(1)
**Trap:** forgetting `abs()`, so a negative input tries `int('-')` and raises `ValueError`.

---

### 23. Reverse the digits of a number

*Number · High*

**Logic:** peel the last digit off with `% 10` and push it onto the growing result with `rev = rev * 10 + d`, which shifts what you have left by one place. Save the sign first and reapply it at the end.

```python
def reverse_digits(n: int) -> int:
    sign = -1 if n < 0 else 1
    n, rev = abs(n), 0
    while n:
        n, d = divmod(n, 10)
        rev = rev * 10 + d
    return sign * rev

reverse_digits(-1230)   # -321
```

**Time:** O(d) · **Space:** O(1)
**Trap:** losing the sign, and not noticing that trailing zeros vanish (120 → 21). That is correct behaviour, but say it out loud so the interviewer knows you spotted it.

---

### 24. Check if a number is a palindrome

*Number · High*

**Logic:** compare the number's digits to their reverse. Negatives can never qualify because the minus sign would have to move.

```python
def is_num_palindrome(n: int) -> bool:
    return n >= 0 and str(n) == str(n)[::-1]
```

**Time:** O(d) · **Space:** O(d)
**Trap:** forgetting the negative case. If string conversion is banned, reverse arithmetically ([23](#23-reverse-the-digits-of-a-number)) and compare.

---

### 25. Greatest common divisor

*Number · High*

**Logic:** Euclid's algorithm. Any common divisor of `a` and `b` also divides `a % b`, so replacing the pair with `(b, a % b)` never changes the answer but shrinks the numbers fast. When the remainder hits 0, the other value is the GCD.

```python
import math
math.gcd(a, b)

def gcd(a: int, b: int) -> int:
    while b:
        a, b = b, a % b
    return a
```

**Time:** O(log min(a,b)) — the values shrink geometrically · **Space:** O(1)
**Trap:** looping down from `min(a, b)` testing every candidate — O(n) instead of O(log n).

---

### 26. Least common multiple

*Number · Medium*

**Logic:** `a × b` counts the shared factors twice, so dividing by the GCD removes the duplication: `lcm = |a·b| / gcd(a,b)`.

```python
import math

def lcm(a: int, b: int) -> int:
    return abs(a * b) // math.gcd(a, b)
```

**Time:** O(log n) · **Space:** O(1)
**Trap:** using `/` instead of `//`, which returns a float and loses precision on large inputs. `math.lcm` exists in Python 3.9+.

---

### 27. Check if a number is a perfect number

*Number · Medium*

**Logic:** a perfect number equals the sum of its **proper** divisors (excluding itself): 6 = 1+2+3, 28 = 1+2+4+7+14. Divisors come in pairs around √n, so find one and add its partner — but only add the partner when it differs, or a perfect square double-counts its root.

```python
def is_perfect(n: int) -> bool:
    if n < 2:
        return False
    total = 1
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            total += i + (n // i if i != n // i else 0)
    return total == n

[n for n in range(2, 500) if is_perfect(n)]   # [6, 28, 496]
```

**Time:** O(√n) · **Space:** O(1)
**Trap:** including `n` itself in the sum, and double-counting the square root.

---

### 28. Count the digits in a number

*Number · Medium*

**Logic:** the length of the string form, ignoring any minus sign.

```python
len(str(abs(n)))
```

**Time:** O(d) · **Space:** O(d)
**Trap:** a `while n: n //= 10` counting loop returns 0 for input 0, but 0 has one digit. Guard that case.

---

### 29. Swap two numbers without a temp variable

*Number · Medium*

**Logic:** Python evaluates the whole right-hand side into a tuple first, then unpacks it — so both old values are safely captured before either name is rebound.

```python
a, b = b, a
```

**Time:** O(1) · **Space:** O(1)
**Trap:** reaching for the XOR trick (`a ^= b; b ^= a; a ^= b`). It is correct for integers but unidiomatic, and it silently zeroes the value if both names refer to the same variable.

---

### 30. Check if a number is even or odd

*Number · Very High*

**Logic:** an even number leaves no remainder when divided by 2.

```python
def is_even(n: int) -> bool:
    return n % 2 == 0
```

**Time:** O(1) · **Space:** O(1)
**Trap:** Python's `%` returns a result with the sign of the *divisor*, so `-7 % 2 == 1`, not `-1` as in C. That means `n % 2 == 1` does correctly detect odd negatives here — but `n % 2 != 0` is the habit that ports safely. `n & 1` also works.

---

### 31. Find the largest of three numbers

*Number · High*

**Logic:** `max` takes any number of positional arguments.

```python
max(a, b, c)
```

**Time:** O(1) · **Space:** O(1)
**Trap:** writing a six-branch if/elif chain. If the interviewer bans `max`, nested comparison is fine — but say what you would normally reach for.

---

### 32. Multiplication table of a number

*Number · Medium*

**Logic:** loop 1 through 10 and print each product.

```python
for i in range(1, 11):
    print(f"{n} x {i} = {n * i}")
```

**Time:** O(1) — a fixed 10 iterations · **Space:** O(1)
**Trap:** `range(1, 10)` stops at 9.

---

### 33. Check if a number is a power of two

*Number · Medium*

**Logic:** a power of two has exactly one bit set (`1000`). Subtracting 1 flips that bit to 0 and sets every bit below it (`0111`), so the two values share no bits and `n & (n-1)` is 0.

```python
def is_power_of_two(n: int) -> bool:
    return n > 0 and n & (n - 1) == 0
```

**Time:** O(1) · **Space:** O(1)
**Trap:** omitting `n > 0`. Without it `0 & -1 == 0` and zero wrongly passes.

---

### 34. Sum of squares of the first n naturals

*Number · Low*

**Logic:** the closed form `n(n+1)(2n+1)/6` gives the answer in constant time; the loop is the O(n) fallback.

```python
n * (n + 1) * (2 * n + 1) // 6
sum(i * i for i in range(1, n + 1))
```

**Time:** O(1) closed form, O(n) loop · **Space:** O(1)
**Trap:** `/` returns a float. Use `//`.

---

### 35. Convert decimal to binary

*Number · Medium*

**Logic:** repeated division by 2 collects the bits from least to most significant — or let Python format it directly.

```python
bin(n)[2:]        # strip the '0b' prefix
f"{n:b}"          # cleaner
f"{n:08b}"        # zero-padded to 8 bits
```

**Time:** O(log n) — one step per bit · **Space:** O(log n)
**Trap:** forgetting to strip `0b`. The f-string form sidesteps it.

---

### 36. Check if a year is a leap year

*Number · Medium*

**Logic:** every 4th year is a leap year, except century years, except those divisible by 400.

```python
def is_leap(y: int) -> bool:
    return y % 4 == 0 and (y % 100 != 0 or y % 400 == 0)
```

**Time:** O(1) · **Space:** O(1)
**Trap:** stopping at `y % 4`. **1900 is not a leap year** (divisible by 100, not 400); 2000 is. That exact case is why the question is asked.

---

### 37. Simple and compound interest

*Number · Low*

**Logic:** direct formulas. Compound interest is the final amount minus the principal.

```python
simple   = (p * r * t) / 100
compound = p * ((1 + r / 100) ** t) - p
```

**Time:** O(1) · **Space:** O(1)
**Trap:** integer division truncating the rate — use float literals.

---

## Lists

### 38. Sum all elements in a list

*List · Very High*

**Logic:** the builtin does the accumulation loop.

```python
sum(lst)
sum(lst) / len(lst) if lst else 0      # average, guarded
```

**Time:** O(n) · **Space:** O(1)
**Trap:** dividing by `len(lst)` without checking for an empty list.

---

### 39. Find the largest and smallest in a list

*List · Very High*

**Logic:** one linear scan each, tracking the best value seen.

```python
max(lst), min(lst)
max(lst, default=None)      # safe on empty input
```

**Time:** O(n) · **Space:** O(1)
**Trap:** `max([])` raises `ValueError` — pass `default=` when the list may be empty.

---

### 40. Find the second largest in a list

*List · Very High*

**Logic:** track the top two in a single pass. When a new value beats the best, the old best slides down into second. The `first > x > second` guard is what skips duplicates of the maximum.

```python
def second_largest(lst: list[int]) -> int | None:
    first = second = float("-inf")
    for x in lst:
        if x > first:
            first, second = x, first
        elif first > x > second:
            second = x
    return None if second == float("-inf") else second

second_largest([5, 5, 3])   # 3
```

**Time:** O(n), single pass · **Space:** O(1)
**Trap:** `sorted(lst)[-2]` returns 5 for `[5, 5, 3]` because the maximum is duplicated — usually the intended answer is 3. Clarify whether duplicates count; `sorted(set(lst))[-2]` is an acceptable O(n log n) alternative.

---

### 41. Reverse a list

*List · High*

**Logic:** slicing produces a reversed copy; `.reverse()` mutates in place with no extra allocation; `reversed()` gives a lazy iterator.

```python
rev = lst[::-1]          # new list, O(n) space
lst.reverse()            # in place, O(1) space, returns None
list(reversed(lst))      # via iterator
```

**Time:** O(n) · **Space:** O(n) for the copy, O(1) in place.
**Trap:** `lst = lst.reverse()` sets `lst` to `None`. In-place methods return `None` — the most common Python interview bug there is.

---

### 42. Remove duplicates from a list

*List · Very High*

**Logic:** dict keys are unique and ordered by insertion, so `dict.fromkeys` dedupes while keeping first-appearance order. `set` is faster to type but unordered.

```python
list(dict.fromkeys(lst))   # preserves order — preferred
list(set(lst))             # does NOT preserve order
```

**Time:** O(n) · **Space:** O(n)
**Trap:** using `set` when order matters, and `TypeError` when elements are unhashable (lists, dicts). Unhashable items need an O(n²) scan or a key function.

---

### 43. Sort a list without using sort

*List · High*

**Logic:** bubble sort. Repeatedly compare adjacent pairs and swap them if out of order; after pass `i` the largest `i` elements have bubbled to the end, so the inner loop can shrink. If a pass makes no swaps the list is already sorted and you can stop.

```python
def bubble_sort(lst: list[int]) -> list[int]:
    a = lst[:]
    for i in range(len(a)):
        swapped = False
        for j in range(len(a) - i - 1):
            if a[j] > a[j + 1]:
                a[j], a[j + 1] = a[j + 1], a[j]
                swapped = True
        if not swapped:
            break
    return a
```

**Time:** O(n²) average and worst — two nested loops each proportional to n; **O(n)** best case thanks to the early exit.
**Space:** O(1) extra. **Stable:** yes.
**Trap:** omitting the `swapped` flag, and being unable to explain where the O(n²) comes from.

---

### 44. Count element occurrences in a list

*List · High*

**Logic:** `Counter` tallies everything in one pass; `list.count(x)` scans for one value.

```python
from collections import Counter
Counter(lst)          # all counts at once
lst.count(x)          # a single element
```

**Time:** O(n) · **Space:** O(k)
**Trap:** `for x in lst: print(lst.count(x))` is O(n²) *and* prints duplicates.

---

### 45. Merge two sorted lists

*List · Very High*

**Logic:** because both inputs are already sorted, the next smallest element overall is always at the front of one of them. Keep an index into each, repeatedly take the smaller head, then append whatever tail remains.

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

**Time:** O(n+m) — each element is visited once · **Space:** O(n+m)
**Trap:** `sorted(a + b)` is O((n+m)log(n+m)) and throws away the sortedness that is the whole point. Also: don't forget the leftover tail. Using `<=` rather than `<` keeps the merge stable — the building block of merge sort ([97](#97-merge-sort)).

---

### 46. Sum of even and odd numbers in a list

*List · Medium*

**Logic:** filter by remainder inside a generator expression and sum each group.

```python
evens = sum(x for x in lst if x % 2 == 0)
odds  = sum(x for x in lst if x % 2)
```

**Time:** O(n) · **Space:** O(1)
**Trap:** negative-odd modulo behaviour — see [30](#30-check-if-a-number-is-even-or-odd).

---

### 47. Move all zeroes to the end

*List · High*

**Logic:** keep a write pointer at the position where the next non-zero belongs. Scan with a read pointer; on each non-zero, swap it into the write slot and advance. Non-zeros stay in relative order and the zeros are pushed right automatically.

```python
def move_zeroes(lst: list[int]) -> None:
    pos = 0
    for i, x in enumerate(lst):
        if x != 0:
            lst[pos], lst[i] = lst[i], lst[pos]
            pos += 1

# [0, 1, 0, 3, 12] -> [1, 3, 12, 0, 0]
```

**Time:** O(n) · **Space:** O(1), fully in place.
**Trap:** building a new list when the question wants in place, and breaking the relative order of the non-zero elements.

---

### 48. Rotate a list by k positions

*List · High*

**Logic:** a left rotation by k is just the slice from k onwards followed by the slice up to k. Reduce k modulo n first, since rotating by n returns the original.

```python
def rotate(lst: list, k: int) -> list:
    if not lst:
        return lst
    k %= len(lst)
    return lst[k:] + lst[:k]         # left rotation
    # right rotation: lst[-k:] + lst[:-k]
```

**Time:** O(n) · **Space:** O(n)
**Trap:** not taking `k % n`, so `k > n` gives wrong output or an empty slice. Clarify left vs right.

---

### 49. Find the intersection of two lists

*List · High*

**Logic:** convert to sets and use `&`, which hashes both sides so membership is O(1) instead of a scan.

```python
list(set(a) & set(b))

b_set = set(b)
[x for x in a if x in b_set]      # preserves a's order
```

**Time:** O(n+m) · **Space:** O(n)
**Trap:** `[x for x in a if x in b]` with `b` still a *list* is O(n·m). Convert once and hoist it out of the comprehension. Sets also lose order and drop duplicates.

---

### 50. Find missing numbers in a range

*List · High*

**Logic:** set difference against the full expected range. If exactly one number is missing, Gauss's formula `n(n+1)/2` gives the expected total and the gap is the difference — O(1) space, no set needed.

```python
missing = sorted(set(range(1, n + 1)) - set(lst))

def one_missing(lst: list[int], n: int) -> int:
    return n * (n + 1) // 2 - sum(lst)
```

**Time:** O(n) · **Space:** O(n), or **O(1)** for the sum trick.
**Trap:** not asking how many are missing. If it is exactly one, the formula is the answer they want.

---

### 51. Flatten a nested list

*List · High*

**Logic:** a double comprehension flattens exactly one level. For arbitrary depth you need recursion: yield plain items, and recurse with `yield from` whenever you hit another list.

```python
flat = [x for sub in nested for x in sub]        # one level only

def deep_flatten(lst):
    for x in lst:
        if isinstance(x, (list, tuple)):
            yield from deep_flatten(x)
        else:
            yield x

list(deep_flatten([1, [2, [3, [4]]], 5]))   # [1, 2, 3, 4, 5]
```

**Time:** O(n) · **Space:** O(n)
**Trap:** offering only the comprehension when the nesting is arbitrarily deep. Ask about depth first.

---

### 52. Check if a list is sorted

*List · Medium*

**Logic:** zip the list with itself offset by one to get every adjacent pair, then assert each pair is in order. `all()` short-circuits at the first violation.

```python
all(a <= b for a, b in zip(lst, lst[1:]))
```

**Time:** O(n), stopping early on failure · **Space:** O(1) if you zip iterators.
**Trap:** `lst == sorted(lst)` is O(n log n) and allocates a full copy.

---

### 53. Sum a matrix (list of lists)

*List · Medium*

**Logic:** sum each row, then sum those row totals.

```python
sum(map(sum, matrix))
```

**Time:** O(n) over all elements · **Space:** O(1)
**Trap:** assuming rectangular rows — this works on ragged input too, which is worth noting.

---

### 54. Cumulative sum of a list

*List · Medium*

**Logic:** each output element is the previous output plus the current input, so one pass with a running total suffices. `itertools.accumulate` is exactly that loop.

```python
from itertools import accumulate
list(accumulate(lst))      # [1,2,3,4] -> [1, 3, 6, 10]
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `[sum(lst[:i+1]) for i in range(len(lst))]` is O(n²) — it re-adds the whole prefix at every step.

---

### 55. Find all pairs summing to a target

*List · Very High*

**Logic:** for each element, the number that would complete the pair is `target - x`. Keep a set of everything seen so far and check for that complement in O(1). Checking *before* inserting `x` is what prevents an element pairing with itself.

```python
def pair_sum(lst: list[int], target: int) -> list[tuple[int, int]]:
    seen, out = set(), []
    for x in lst:
        if target - x in seen:
            out.append((target - x, x))
        seen.add(x)
    return out

pair_sum([1, 2, 3, 4, 3], 6)   # [(2, 4), (3, 3)]
```

**Time:** O(n) · **Space:** O(n)
**Trap:** nested loops (O(n²)), double-counting each pair, and self-pairing. See `02-dsa-problem-solving.md` Q1 for the index-returning Two Sum variant.

---

### 56. Split a list into equal chunks

*List · Medium*

**Logic:** step through the indices in jumps of k and slice. Slicing past the end is safe in Python, so the short final chunk needs no special handling.

```python
def chunks(lst: list, k: int) -> list[list]:
    return [lst[i:i + k] for i in range(0, len(lst), k)]

chunks([1,2,3,4,5], 2)   # [[1, 2], [3, 4], [5]]
```

**Time:** O(n) · **Space:** O(n)
**Trap:** clarify whether the short final chunk should be kept, padded, or dropped.

---

### 57. Find the mode of a list

*List · Medium*

**Logic:** count everything, then take the highest count.

```python
from collections import Counter

c = Counter(lst)
mode = c.most_common(1)[0][0]
all_modes = [v for v, n in c.items() if n == max(c.values())]   # handles ties
```

**Time:** O(n) · **Space:** O(k)
**Trap:** ties — `most_common` picks arbitrarily among equal counts. Ask whether all modes are wanted.

---

### 58. Interchange the first and last elements

*List · Low*

**Logic:** tuple unpacking swaps both slots at once; `-1` indexes the last element.

```python
if len(lst) >= 2:
    lst[0], lst[-1] = lst[-1], lst[0]
```

**Time:** O(1) · **Space:** O(1)
**Trap:** an empty list raises `IndexError`.

---

### 59. Multiply all numbers in a list

*List · Medium*

**Logic:** fold multiplication across the list, starting from 1.

```python
import math
math.prod(lst)

from functools import reduce      # pre-3.8
reduce(lambda a, b: a * b, lst, 1)
```

**Time:** O(n) · **Space:** O(1)
**Trap:** initialising the accumulator to 0 — everything collapses to 0.

---

### 60. Find array leaders

*List · Medium*

**Logic:** a leader is greater than everything to its right. Scanning **right to left** means "everything to the right" is just one running maximum, so a single pass answers every position. Reverse the collected result at the end.

```python
def leaders(lst: list[int]) -> list[int]:
    out, running_max = [], float("-inf")
    for x in reversed(lst):
        if x > running_max:
            out.append(x)
            running_max = x
    return out[::-1]

leaders([16, 17, 4, 3, 5, 2])   # [17, 5, 2]
```

**Time:** O(n) · **Space:** O(n) for the output.
**Trap:** the obvious nested loop is O(n²). The direction of the scan is the whole insight.

---

## Dictionaries

### 61. Merge two dictionaries

*Dict · Very High*

**Logic:** unpack both into a new dict; later keys win on collision.

```python
merged = a | b            # Python 3.9+
merged = {**a, **b}       # any version
a.update(b)               # in place, mutates a
```

**Time:** O(n) · **Space:** O(n)
**Trap:** collisions are silent — `b` overwrites `a`. Check `a.keys() & b.keys()` first if that matters.

---

### 62. Sort a dictionary by value

*Dict · Very High*

**Logic:** dicts do not sort themselves, so sort the `(key, value)` pairs with a key function that selects the value, then rebuild a dict (which keeps that insertion order).

```python
dict(sorted(d.items(), key=lambda kv: kv[1]))
dict(sorted(d.items(), key=lambda kv: kv[1], reverse=True))    # descending
```

**Time:** O(n log n) · **Space:** O(n)
**Trap:** sorting `d` directly sorts the **keys**. You must sort `d.items()`.

---

### 63. Invert a dictionary

*Dict · High*

**Logic:** swap each key and value in a comprehension. If values repeat, collect keys into lists instead or entries are lost.

```python
{v: k for k, v in d.items()}

from collections import defaultdict
inv = defaultdict(list)
for k, v in d.items():
    inv[v].append(k)                # safe when values repeat
```

**Time:** O(n) · **Space:** O(n)
**Trap:** duplicate values collapse silently — last one wins.

---

### 64. Sum all values in a dictionary

*Dict · Medium*

**Logic:** `.values()` is already an iterable of the values.

```python
sum(d.values())
```

**Time:** O(n) · **Space:** O(1)

---

### 65. Find the key with the maximum value

*Dict · High*

**Logic:** iterating a dict yields keys, so tell `max` to *rank* those keys by their value via `key=d.get`.

```python
max(d, key=d.get)
```

**Time:** O(n) · **Space:** O(1)
**Trap:** `max(d)` returns the largest **key**, not the key of the largest value. The `key=d.get` is the entire answer.

---

### 66. Group items by a property

*Dict · High*

**Logic:** map each group value to a list of members. `defaultdict(list)` creates the empty list on first access so you never check for the key.

```python
from collections import defaultdict

groups = defaultdict(list)
for word in words:
    groups[len(word)].append(word)
```

**Time:** O(n) · **Space:** O(n)
**Trap:** a plain dict raises `KeyError` on the first append. Use `defaultdict(list)` or `d.setdefault(k, []).append(x)`.

---

### 67. Count word frequency in a text

*Dict · Very High*

**Logic:** normalise case, extract word tokens with a regex so punctuation is not glued on, then tally with `Counter`.

```python
import re
from collections import Counter

def word_freq(text: str) -> Counter:
    return Counter(re.findall(r"[a-z']+", text.lower()))

word_freq("The cat. the CAT cat!").most_common(1)   # [('cat', 3)]
```

**Time:** O(n) · **Space:** O(k) distinct words.
**Trap:** `text.split()` leaves punctuation attached so `"cat."` and `"cat"` count separately, and skipping `.lower()` splits `"The"` from `"the"`.

---

### 68. Check if two dictionaries are equal

*Dict · Low*

**Logic:** `==` compares keys and values directly.

```python
a == b
```

**Time:** O(n) · **Space:** O(1)
**Trap:** none really — but know that key insertion **order does not affect equality**.

---

### 69. Remove a key safely

*Dict · Medium*

**Logic:** `pop` with a default returns the default instead of raising when the key is absent.

```python
d.pop(key, None)
```

**Time:** O(1) · **Space:** O(1)
**Trap:** `del d[key]` raises `KeyError` when the key is missing.

---

### 70. Nested dictionary access with a default

*Dict · Medium*

**Logic:** chain `.get()` calls, defaulting each intermediate level to an empty dict so the next `.get()` always has something to call.

```python
value = d.get("a", {}).get("b", "default")
```

**Time:** O(1) · **Space:** O(1)
**Trap:** `d["a"]["b"]` raises `KeyError`, and if `d["a"]` is `None` you get `TypeError` instead. For deep paths, loop over the keys.

---

## Stacks, queues and search

### 71. Check for balanced parentheses

*Stack · Very High*

**Logic:** the most recently opened bracket must close first — that is last-in-first-out, so use a stack. Push every opener; on a closer, pop and check it matches. At the end the stack must be empty, or something was never closed.

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

is_balanced("{[()]}")   # True
is_balanced("(((")      # False
```

**Time:** O(n) · **Space:** O(n) worst case, all openers.
**Trap:** returning `True` without the final `not stack` check, so `"((("` passes. Also: popping an empty stack when a closer arrives first — the `not stack` guard handles it.

---

### 72. Implement a stack using a list

*Stack · High*

**Logic:** push and pop at the **same** end. A Python list appends and pops at the tail in amortised O(1) because no elements shift.

```python
stack = []
stack.append(x)        # push — O(1)
top = stack.pop()      # pop  — O(1)
peek = stack[-1]       # peek — O(1)
```

**Time:** O(1) per operation · **Space:** O(n)
**Trap:** `pop(0)` is O(n) and makes it a queue, not a stack.

---

### 73. Implement a queue

*Queue · High*

**Logic:** FIFO needs cheap removal from the front. `collections.deque` is a doubly-linked structure, so both ends are O(1).

```python
from collections import deque

q = deque()
q.append(x)            # enqueue — O(1)
first = q.popleft()    # dequeue — O(1)
```

**Time:** O(1) per operation · **Space:** O(n)
**Trap:** `list.pop(0)` is O(n) because every remaining element shifts left — fine for 10 items, fatal for 100,000.

---

### 74. Binary search in a sorted list

*Search · Very High*

**Logic:** compare the middle element to the target. Because the list is sorted, one half can be discarded every time — so the search space halves each iteration. Move `lo` past `mid` or `hi` before it so the range always shrinks and the loop terminates.

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

**Time:** O(log n) — halving n takes log₂n steps · **Space:** O(1)
**Trap:** `while lo < hi` misses the single-element case, and dropping the `± 1` gives an infinite loop. Python's `bisect` module is the production answer.

---

### 75. Linear search

*Search · Medium*

**Logic:** scan until you find it; `enumerate` gives the index alongside the value.

```python
def linear_search(lst: list, target) -> int:
    for i, x in enumerate(lst):
        if x == target:
            return i
    return -1
```

**Time:** O(n) · **Space:** O(1)
**Trap:** returning the value instead of the index, and forgetting a "not found" sentinel.

---

### 76. FizzBuzz

*Logic · Very High*

**Logic:** test the most specific condition first. A multiple of 15 is also a multiple of 3 and of 5, so if you check 3 first it matches and the `elif` chain exits before ever reaching the 15 case.

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
**Trap:** checking 3 and 5 before 15, so 15 prints "Fizz". Condition ordering is the entire point of the question.

---

## Patterns

### 77. Right-angled star triangle

*Pattern · High*

**Logic:** row `i` holds `i` stars, and string multiplication builds the row without an inner loop.

```python
for i in range(1, n + 1):
    print("*" * i)
```

**Time:** O(n²) — total characters printed · **Space:** O(1)
**Trap:** off-by-one in the `range` bounds.

---

### 78. Pyramid star pattern

*Pattern · High*

**Logic:** to centre the stars, each row needs `n - i` leading spaces, and the star count grows by 2 per row: `2i - 1` gives 1, 3, 5, 7. Derive it rather than memorising — the widest row must be `2n - 1` wide.

```python
n = 5
for i in range(1, n + 1):
    print(" " * (n - i) + "*" * (2 * i - 1))
```

**Time:** O(n²) · **Space:** O(1)
**Trap:** miscounting the spaces so the pyramid leans.

---

### 79. Inverted star pattern

*Pattern · Medium*

**Logic:** the same as [77](#77-right-angled-star-triangle) with the row counter running downwards.

```python
for i in range(n, 0, -1):
    print("*" * i)
```

**Time:** O(n²) · **Space:** O(1)
**Trap:** reversed `range` bounds — `range(n, 0, -1)` stops at 1, not 0.

---

### 80. Pascal's triangle

*Pattern · Medium*

**Logic:** every interior entry is the sum of the two entries above it, and each row starts and ends with 1. So build each row from pairwise sums of the previous row, wrapped in 1s.

```python
row = [1]
for _ in range(n):
    print(row)
    row = [1] + [row[j] + row[j + 1] for j in range(len(row) - 1)] + [1]

# [1] / [1,1] / [1,2,1] / [1,3,3,1] / [1,4,6,4,1]
```

**Time:** O(n²) · **Space:** O(n) for one row.
**Trap:** mishandling the leading and trailing 1s, and looping `len(row)` instead of `len(row) - 1` (an `IndexError`).

---

## Files

### 81. Read a file and count lines and words

*File · High*

**Logic:** iterating a file object yields one line at a time, so memory stays constant regardless of file size. `with` closes the handle even if an exception fires.

```python
with open("file.txt", encoding="utf-8") as f:
    lines = words = 0
    for line in f:
        lines += 1
        words += len(line.split())
```

**Time:** O(n) · **Space:** O(1) — one line held at a time.
**Trap:** `f.readlines()` loads the whole file into memory, and omitting `with` leaks the handle.

---

### 82. Most frequent word in a large file

*File · Medium*

**Logic:** stream line by line and fold each line's tokens into a running `Counter`. Never materialise the whole file.

```python
import re
from collections import Counter

freq = Counter()
with open("file.txt", encoding="utf-8") as f:
    for line in f:
        freq.update(re.findall(r"[a-z']+", line.lower()))

print(freq.most_common(5))
```

**Time:** O(n) · **Space:** O(k) distinct words — not O(file size).
**Trap:** `f.read()` on a multi-gigabyte file. Also note the counter still grows with vocabulary, which is the genuine memory bound.

---

### 83. Write a list to a file line by line

*File · Medium*

**Logic:** join with newlines, or write each item with an explicit `\n`.

```python
with open("out.txt", "w", encoding="utf-8") as f:
    f.write("\n".join(map(str, items)))
    # or: f.writelines(f"{x}\n" for x in items)
```

**Time:** O(n) · **Space:** O(1) with the generator form.
**Trap:** `writelines` does **not** add newlines — you must include them yourself.

---

## Sets

### 84. Check if a value appears twice in a list

*Set · High*

**Logic:** remember what you have seen in a set; the first time you meet something already in it, you have your duplicate. Returns early rather than scanning the whole list.

```python
def has_duplicate(lst: list) -> bool:
    seen = set()
    for x in lst:
        if x in seen:
            return True
        seen.add(x)
    return False
```

**Time:** O(n), short-circuiting · **Space:** O(n)
**Trap:** nested loops are O(n²). Set membership is O(1) average because it hashes rather than scans.

---

### 85. Check if all list elements are unique

*Set · High*

**Logic:** a set drops duplicates, so if the deduplicated size matches the original, nothing repeated.

```python
len(set(lst)) == len(lst)
```

**Time:** O(n) · **Space:** O(n)
**Trap:** `TypeError` on unhashable elements. Note this version always scans everything, whereas [84](#84-check-if-a-value-appears-twice-in-a-list) exits at the first duplicate.

---

### 86. Union and difference of two lists

*Set · Medium*

**Logic:** set operators express these directly.

```python
set(a) | set(b)      # union
set(a) & set(b)      # intersection
set(a) - set(b)      # in a but not b
set(a) ^ set(b)      # in exactly one of them
```

**Time:** O(n+m) · **Space:** O(n)
**Trap:** sets lose order and drop duplicates — wrap in `sorted()` if order matters.

---

### 87. Remove elements of one list from another

*Set · Medium*

**Logic:** hash the exclusion list once, then filter with a comprehension so `a`'s order survives.

```python
b_set = set(b)
result = [x for x in a if x not in b_set]
```

**Time:** O(n+m) · **Space:** O(m)
**Trap:** mutating a list while iterating over it skips elements — build a new list instead.

---

## Recursion

### 88. Fibonacci with memoisation

*Recursion · High*

**Logic:** the naive recursion recomputes the same values exponentially often. Caching each result by argument means every distinct `n` is computed once, which collapses the exponential recursion tree into a linear chain.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

**Time:** O(n) — n distinct subproblems, each O(1) after caching · **Space:** O(n) for the cache and the stack.
**Trap:** forgetting the decorator leaves it O(2ⁿ). This problem tests whether you can spot overlapping subproblems — say the words.

---

### 89. Sum of digits recursively

*Recursion · Medium*

**Logic:** the last digit is `n % 10` and the rest of the number is `n // 10`, so the sum is the last digit plus the sum of the rest. Base case: a single digit is its own sum.

```python
def digit_sum(n: int) -> int:
    n = abs(n)
    return n if n < 10 else n % 10 + digit_sum(n // 10)
```

**Time:** O(d) · **Space:** O(d) call stack.
**Trap:** a missing or wrong base case gives infinite recursion and `RecursionError`.

---

### 90. Reverse a string recursively

*Recursion · Medium*

**Logic:** the reverse of a string is the reverse of everything after the first character, with that character appended.

```python
def rev(s: str) -> str:
    return s if len(s) <= 1 else rev(s[1:]) + s[0]
```

**Time:** **O(n²)** — each call slices a fresh string of length n−1, so the copying costs sum to a quadratic.
**Space:** O(n) stack depth, O(n²) total allocation.
**Trap:** claiming O(n). Volunteering the quadratic slicing cost unprompted is what the interviewer is listening for — and `s[::-1]` is the real answer.

---

### 91. Tower of Hanoi

*Recursion · Medium*

**Logic:** to move n disks from source to destination, first move the top n−1 out of the way onto the auxiliary peg, move the largest disk across, then move those n−1 on top of it. The peg roles rotate in the two recursive calls.

```python
def hanoi(n: int, src: str = "A", dst: str = "C", aux: str = "B") -> None:
    if n == 0:
        return
    hanoi(n - 1, src, aux, dst)
    print(f"Move disk {n} from {src} to {dst}")
    hanoi(n - 1, aux, dst, src)
```

**Time:** O(2ⁿ) — exactly 2ⁿ−1 moves, which is provably minimal · **Space:** O(n) stack depth.
**Trap:** swapping the auxiliary and destination pegs in the recursive calls.

---

### 92. Fast exponentiation

*Recursion · Medium*

**Logic:** `bᵉ = (b^(e/2))²`, so halving the exponent squares the result — that turns n multiplications into log n. Compute the half **once** and reuse it; an odd exponent needs one extra factor of the base.

```python
def power(base: float, exp: int) -> float:
    if exp < 0:
        return 1 / power(base, -exp)
    if exp == 0:
        return 1
    half = power(base, exp // 2)
    return half * half * (base if exp % 2 else 1)
```

**Time:** O(log n) · **Space:** O(log n) stack.
**Trap:** calling `power(base, exp//2)` twice instead of storing it — that makes it O(n) again. Also handle negative and zero exponents.

---

### 93. Flatten a nested dictionary

*Recursion · High*

**Logic:** walk the dict; when a value is itself a dict, recurse with the path so far joined onto the key. Non-dict values become leaves keyed by their full dotted path.

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

flatten({"a": {"b": 1, "c": {"d": 2}}})   # {'a.b': 1, 'a.c.d': 2}
```

**Time:** O(n) · **Space:** O(n)
**Trap:** not handling dicts nested inside **lists** — ask whether that case exists. This one genuinely comes up in data-engineering and API-integration roles.

---

## Sorting

### 94. Bubble sort

*Sorting · High*

**Logic:** see [43](#43-sort-a-list-without-using-sort) — repeatedly swap adjacent out-of-order pairs so the largest element bubbles to the end each pass.

**Time:** O(n²) average and worst, O(n) best with the early-exit flag · **Space:** O(1) · **Stable:** yes.
**Trap:** omitting the early exit, which is the only thing that gives it a linear best case.

---

### 95. Insertion sort

*Sorting · Medium*

**Logic:** treat the left portion as sorted. Take the next element, shift every larger element in the sorted part one place right, and drop it into the gap — like sorting a hand of cards.

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

**Time:** O(n²) average, **O(n) best on nearly-sorted input** (the inner `while` exits immediately) · **Space:** O(1) · **Stable:** yes.
**Trap:** not knowing about the linear best case — it is exactly why real implementations, including Python's Timsort, use insertion sort for small or nearly-sorted runs.

---

### 96. Selection sort

*Sorting · Medium*

**Logic:** find the smallest remaining element and swap it into the current position. Repeat for each index.

```python
def selection_sort(a: list[int]) -> list[int]:
    a = a[:]
    for i in range(len(a)):
        m = min(range(i, len(a)), key=a.__getitem__)
        a[i], a[m] = a[m], a[i]
    return a
```

**Time:** O(n²) in **all** cases — it always scans the remainder, so no early exit is possible · **Space:** O(1) · **Stable:** **no**.
**Trap:** claiming it is stable; the long-distance swap can reorder equal elements. Its one virtue is making only n−1 swaps.

---

### 97. Merge sort

*Sorting · High*

**Logic:** split the list in half, sort each half recursively, then merge the two sorted halves ([45](#45-merge-two-sorted-lists)). There are log n levels of splitting and each level merges O(n) elements.

```python
def merge_sort(a: list[int]) -> list[int]:
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    return merge(merge_sort(a[:mid]), merge_sort(a[mid:]))
```

**Time:** O(n log n) **guaranteed** — log n levels × O(n) merging per level · **Space:** O(n) auxiliary · **Stable:** yes.
**Trap:** forgetting the O(n) extra space, which is merge sort's main cost against quicksort.

---

### 98. Quick sort

*Sorting · High*

**Logic:** pick a pivot and partition the list into smaller, equal, and larger groups, then recurse on the outer two. A good pivot halves the input each time, giving log n depth.

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

**Time:** O(n log n) average, **O(n²) worst** when the pivot is consistently the smallest or largest — which is what happens on already-sorted input if you always pick the first element.
**Space:** O(log n) stack for the in-place version, O(n) for this one.
**Trap:** not knowing the worst case or its cause. Random or median-of-three pivoting mitigates it.

---

## Bit tricks

### 99. Count set bits in a number

*Bit · Medium*

**Logic:** `n & (n - 1)` clears the **lowest set bit**, because subtracting 1 flips that bit off and all zeros below it to 1. Loop until the number is 0 and you have run once per set bit.

```python
bin(n).count("1")
n.bit_count()              # Python 3.10+

def popcount(n: int) -> int:      # Brian Kernighan's algorithm
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count
```

**Time:** O(number of set bits) for Kernighan, O(log n) for the string version · **Space:** O(1)
**Trap:** not knowing the `n & (n-1)` trick — it is the same identity behind the power-of-two check ([33](#33-check-if-a-number-is-a-power-of-two)).

---

### 100. Find the single number in a list of pairs

*Bit · High*

**Logic:** XOR every element together. `x ^ x == 0` so each duplicate cancels itself, and `0 ^ y == y` so the unpaired value survives. XOR is commutative and associative, so the order of the list is irrelevant.

```python
from functools import reduce
import operator

def single_number(lst: list[int]) -> int:
    return reduce(operator.xor, lst)

single_number([4, 1, 2, 1, 2])   # 4
```

**Time:** O(n) · **Space:** **O(1)**
**Trap:** reaching for a `Counter` — correct, but O(n) space. The constant-space XOR solution is what the question is testing.

---

## Quick revision checklist

If you only drill twenty, drill these — highest frequency, widest pattern coverage.

| # | Problem | What it really tests |
|---|---|---|
| [2](#2-check-if-a-string-is-a-palindrome) | Palindrome | Two pointers plus input normalisation |
| [4](#4-check-if-two-strings-are-anagrams) | Anagrams | `Counter`, and O(n) vs O(n log n) |
| [7](#7-find-the-first-non-repeating-character) | First non-repeating char | Two-pass counting, and why order matters |
| [16](#16-factorial-of-a-number) | Factorial | Iteration vs recursion limits |
| [17](#17-nth-fibonacci-number) | Fibonacci | The O(2ⁿ) → O(n) insight |
| [19](#19-check-if-a-number-is-prime) | Prime check | Why √n suffices |
| [21](#21-check-if-a-number-is-an-armstrong-number) | Armstrong number | Reading the definition carefully |
| [23](#23-reverse-the-digits-of-a-number) | Reverse digits | `divmod` and sign handling |
| [25](#25-greatest-common-divisor) | GCD | Euclid's algorithm |
| [30](#30-check-if-a-number-is-even-or-odd) | Even/odd | Negative modulo in Python |
| [40](#40-find-the-second-largest-in-a-list) | Second largest | Single-pass tracking, duplicate handling |
| [42](#42-remove-duplicates-from-a-list) | Remove duplicates | Order-preserving dedupe idiom |
| [45](#45-merge-two-sorted-lists) | Merge sorted lists | Two pointers; the merge-sort building block |
| [55](#55-find-all-pairs-summing-to-a-target) | Pair sum | Hash-set complement lookup |
| [62](#62-sort-a-dictionary-by-value) | Sort dict by value | `key=` functions |
| [67](#67-count-word-frequency-in-a-text) | Word frequency | Normalisation plus `Counter` |
| [71](#71-check-for-balanced-parentheses) | Balanced parentheses | The canonical stack problem |
| [74](#74-binary-search-in-a-sorted-list) | Binary search | Loop bounds without off-by-one errors |
| [76](#76-fizzbuzz) | FizzBuzz | Condition ordering |
| [100](#100-find-the-single-number-in-a-list-of-pairs) | Single number (XOR) | Constant-space bit trick |

For the next level up — sliding window, BFS/DFS, heaps, topological sort — continue to [`02-dsa-problem-solving.md`](02-dsa-problem-solving.md).
