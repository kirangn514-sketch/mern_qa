# Coding Practice — Data Structures & Patterns (Complete Notes)

*Code examples in JavaScript, since these patterns apply directly to interview coding rounds regardless of language.*

---

## 1. Arrays

**Simple definition:** An array is a collection of elements stored in **contiguous memory**, accessed by an **index**. It's the most fundamental data structure — most other structures (stacks, queues, hash maps) are built using arrays under the hood.

### Key characteristics
| Operation | Time Complexity | Why |
|---|---|---|
| Access by index (`arr[i]`) | O(1) | Direct memory address calculation |
| Search (unsorted) | O(n) | Must check each element |
| Search (sorted, binary search) | O(log n) | Halves the search space each step |
| Insert/Delete at end | O(1) amortized | No shifting needed |
| Insert/Delete at beginning/middle | O(n) | Requires shifting all subsequent elements |

### Common patterns & example
```js
// Find the maximum subarray sum (Kadane's Algorithm) — classic array problem
function maxSubArray(nums) {
  let maxSoFar = nums[0];
  let maxEndingHere = nums[0];

  for (let i = 1; i < nums.length; i++) {
    // either extend the previous subarray, or start fresh at nums[i]
    maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
    maxSoFar = Math.max(maxSoFar, maxEndingHere);
  }
  return maxSoFar;
}

maxSubArray([-2, 1, -3, 4, -1, 2, 1, -5, 4]); // 6 → [4, -1, 2, 1]
```
**Why this pattern matters:** Kadane's algorithm demonstrates a key array-problem technique — tracking a **running value** as you iterate once (O(n)), instead of checking every possible subarray (which would be O(n²) or worse).

### When arrays show up in interviews
- Subarray/subsequence problems (max sum, product, length).
- Sorting-based problems (merge intervals, find duplicates).
- Matrix problems (2D arrays — rotate, search, traverse in spiral order).

---

## 2. Strings

**Simple definition:** A string is essentially a **sequence/array of characters** — many array techniques (two pointers, sliding window, hashing) apply directly to strings too, since a string can be indexed and iterated just like an array.

### Common operations to know
```js
"hello".length;                  // 5
"hello".charAt(1);                // "e"
"hello".slice(1, 3);               // "el"
"hello".split("");                 // ["h", "e", "l", "l", "o"]
"hello".split("").reverse().join(""); // "olleh" — classic reverse pattern
```

### Example: Check if a string is a palindrome (two-pointer pattern)
```js
function isPalindrome(s) {
  let left = 0;
  let right = s.length - 1;

  while (left < right) {
    if (s[left] !== s[right]) return false;
    left++;
    right--;
  }
  return true;
}

isPalindrome("racecar"); // true
isPalindrome("hello");   // false
```

### Example: Check if two strings are anagrams (hash map pattern)
```js
function isAnagram(s1, s2) {
  if (s1.length !== s2.length) return false;

  const charCount = {};
  for (const char of s1) {
    charCount[char] = (charCount[char] || 0) + 1;
  }
  for (const char of s2) {
    if (!charCount[char]) return false; // missing or already used up
    charCount[char]--;
  }
  return true;
}

isAnagram("listen", "silent"); // true
```
**Why this matters:** This shows the common pattern of using a hash map to **count character frequencies** — a technique that generalizes to many string problems (first unique character, group anagrams, longest substring without repeats).

---

## 3. HashMap (Hash Table / Dictionary / Object)

**Simple definition:** A HashMap stores **key-value pairs**, using a **hash function** to compute where each key's data should live — giving average **O(1)** time complexity for insert, lookup, and delete, regardless of how many items are stored.

**Why it's arguably the single most useful interview data structure:** It converts "have I seen this before?" or "how many times has this occurred?" questions from an O(n) linear scan into an O(1) lookup — this single trick underlies a huge fraction of common coding interview optimizations.

```js
// JavaScript: plain objects or Map can act as a hash map
const map = new Map();
map.set("apple", 3);
map.get("apple");     // 3
map.has("banana");    // false
map.delete("apple");
```

### Example: Two Sum (the single most famous hash map problem)
```js
// Given an array and a target, find two numbers that add up to the target
function twoSum(nums, target) {
  const seen = new Map(); // value → index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(nums[i], i);
  }
  return [];
}

twoSum([2, 7, 11, 15], 9); // [0, 1] — because nums[0] + nums[1] = 9
```
**Why this is the "aha" pattern:** The naive approach checks every pair (O(n²) — nested loops). Using a hash map, you check **as you go**: "have I already seen the number that would complete this pair?" — turning it into a single O(n) pass.

### Common use cases
- Counting frequencies (character counts, most frequent element).
- Detecting duplicates in O(n) instead of O(n²).
- Grouping items by a computed key (e.g., grouping anagrams by their sorted letters).
- Caching/memoization (storing already-computed results, keyed by input).

---

## 4. Linked List

**Simple definition:** A linked list is a linear data structure where each element (**node**) holds its **value** and a **pointer/reference to the next node** — unlike arrays, elements are NOT stored in contiguous memory, so there's no direct indexing; you must traverse from the head to reach a given node.

```js
class ListNode {
  constructor(value) {
    this.value = value;
    this.next = null;
  }
}

// Building: 1 -> 2 -> 3 -> null
const head = new ListNode(1);
head.next = new ListNode(2);
head.next.next = new ListNode(3);
```

### Array vs Linked List
| Operation | Array | Linked List |
|---|---|---|
| Access by index | O(1) | O(n) — must traverse from head |
| Insert/delete at beginning | O(n) — shift everything | O(1) — just update pointers |
| Insert/delete at end | O(1) amortized | O(1) if tail pointer tracked, else O(n) |
| Memory | Contiguous, cache-friendly | Scattered, extra memory for pointers |

### Example: Reverse a linked list (extremely common interview question)
```js
function reverseList(head) {
  let prev = null;
  let current = head;

  while (current !== null) {
    const nextNode = current.next; // save the next node before overwriting
    current.next = prev;            // reverse the pointer
    prev = current;                 // move prev forward
    current = nextNode;             // move current forward
  }
  return prev; // prev is now the new head
}
```

### Example: Detect a cycle (Floyd's Cycle Detection — "tortoise and hare")
```js
function hasCycle(head) {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow.next;          // moves 1 step
    fast = fast.next.next;     // moves 2 steps

    if (slow === fast) return true; // they meet → there's a cycle
  }
  return false; // fast reached the end → no cycle
}
```
**Why this pattern matters:** Using two pointers moving at different speeds is a classic technique for detecting cycles or finding the middle of a linked list in O(n) time and O(1) extra space, without needing a hash set to track visited nodes.

---

## 5. Stack

**Simple definition:** A stack is a **LIFO** (Last-In, First-Out) data structure — the most recently added item is the first one removed. Think of a stack of plates: you add and remove from the top only.

```js
const stack = [];
stack.push(1);   // [1]
stack.push(2);   // [1, 2]
stack.push(3);   // [1, 2, 3]
stack.pop();     // removes and returns 3 → [1, 2]
stack[stack.length - 1]; // "peek" at the top without removing: 2
```
| Operation | Time Complexity |
|---|---|
| Push (add to top) | O(1) |
| Pop (remove from top) | O(1) |
| Peek (view top) | O(1) |

### Example: Valid Parentheses (the quintessential stack problem)
```js
function isValid(s) {
  const stack = [];
  const pairs = { ")": "(", "]": "[", "}": "{" };

  for (const char of s) {
    if (char === "(" || char === "[" || char === "{") {
      stack.push(char); // opening bracket — push it
    } else {
      // closing bracket — must match the most recent opening bracket
      if (stack.pop() !== pairs[char]) return false;
    }
  }
  return stack.length === 0; // valid only if every opening bracket was matched
}

isValid("({[]})"); // true
isValid("(]");     // false
```
**Why stacks fit this problem:** The most recently opened bracket must be the first one closed — that's exactly LIFO behavior, which is why a stack is the natural fit.

### Common use cases
- Matching/validating brackets, parsing expressions.
- Undo/redo functionality.
- Tracking function call history (this is literally how the JavaScript **call stack** works).
- Depth-First Search (DFS) — can be implemented iteratively using an explicit stack.

---

## 6. Queue

**Simple definition:** A queue is a **FIFO** (First-In, First-Out) data structure — the first item added is the first one removed. Think of a checkout line: whoever arrived first gets served first.

```js
const queue = [];
queue.push(1);       // enqueue: [1]
queue.push(2);       // enqueue: [1, 2]
queue.push(3);       // enqueue: [1, 2, 3]
queue.shift();       // dequeue: removes and returns 1 → [2, 3]
```
⚠️ **Performance note:** `Array.shift()` is O(n) in JavaScript since it re-indexes every remaining element. For performance-critical queue implementations, use a proper circular buffer or a linked-list-based queue instead of a plain array.

| Operation | Time Complexity (ideal implementation) |
|---|---|
| Enqueue (add to back) | O(1) |
| Dequeue (remove from front) | O(1) |

### Example: Breadth-First Search (BFS) using a queue — extremely common pattern
```js
function bfs(graph, start) {
  const visited = new Set([start]);
  const queue = [start];
  const order = [];

  while (queue.length > 0) {
    const node = queue.shift(); // process the OLDEST added node first
    order.push(node);

    for (const neighbor of graph[node]) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor); // add newly discovered nodes to the back
      }
    }
  }
  return order;
}
```
**Why queues fit BFS:** BFS explores level by level — you must finish visiting all nodes at the current distance before moving further out, which requires processing nodes in the exact order they were discovered (FIFO).

### Stack vs Queue
| | Stack (LIFO) | Queue (FIFO) |
|---|---|---|
| Order | Last in, first out | First in, first out |
| Real-world analogy | Stack of plates | Checkout line |
| Common use | DFS, undo/redo, expression parsing | BFS, task scheduling, rate limiting |

---

## 7. Sliding Window

**Simple definition:** A technique for problems involving **contiguous subarrays/substrings** — instead of recalculating a window's result from scratch every time it moves, you **maintain a running window** and incrementally add/remove elements as the window slides — turning an O(n²) or O(n³) brute-force approach into O(n).

### When to recognize this pattern
Look for phrases like: "contiguous subarray/substring," "maximum/minimum sum of size k," "longest substring with condition X."

### Example: Maximum sum of a subarray of size k (fixed-size window)
```js
function maxSumSubarray(nums, k) {
  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += nums[i]; // build the initial window

  let maxSum = windowSum;

  for (let i = k; i < nums.length; i++) {
    windowSum += nums[i] - nums[i - k]; // slide: add new element, remove the oldest
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}

maxSumSubarray([2, 1, 5, 1, 3, 2], 3); // 9 → [5, 1, 3]
```
**Without sliding window (brute force):** You'd recompute the sum of every k-sized window from scratch — O(n·k). **With sliding window:** Each step does O(1) work (subtract the outgoing element, add the incoming one) — O(n) total.

### Example: Longest substring without repeating characters (variable-size window)
```js
function lengthOfLongestSubstring(s) {
  const seen = new Map(); // char → last seen index
  let maxLength = 0;
  let windowStart = 0;

  for (let windowEnd = 0; windowEnd < s.length; windowEnd++) {
    const char = s[windowEnd];

    if (seen.has(char) && seen.get(char) >= windowStart) {
      windowStart = seen.get(char) + 1; // shrink window from the left, past the duplicate
    }

    seen.set(char, windowEnd);
    maxLength = Math.max(maxLength, windowEnd - windowStart + 1);
  }
  return maxLength;
}

lengthOfLongestSubstring("abcabcbb"); // 3 → "abc"
```
**Key difference from the fixed-size example:** Here the window's size **changes dynamically** — it grows by moving `windowEnd` forward, and shrinks by moving `windowStart` forward whenever the constraint (no repeating characters) is violated.

---

## 8. Two Pointer

**Simple definition:** A technique using **two index variables** that traverse a data structure (usually a sorted array or string) — often moving toward each other, or at different speeds — to solve problems in O(n) instead of a brute-force O(n²) nested loop.

### When to recognize this pattern
Look for phrases like: "pair that sums to X," "sorted array," "remove duplicates in-place," "reverse in-place," "palindrome check."

### Example: Two Sum on a SORTED array (opposite-direction two pointers)
```js
function twoSumSorted(nums, target) {
  let left = 0;
  let right = nums.length - 1;

  while (left < right) {
    const sum = nums[left] + nums[right];

    if (sum === target) return [left, right];
    if (sum < target) left++;   // sum too small — need a bigger number, move left pointer up
    else right--;                // sum too big — need a smaller number, move right pointer down
  }
  return [];
}

twoSumSorted([1, 2, 3, 4, 6], 6); // [1, 3] → nums[1] + nums[3] = 2 + 4 = 6
```
**Why this works (and why the array must be sorted):** Because the array is sorted, moving `left` forward always increases the sum, and moving `right` backward always decreases it — this lets you eliminate half the remaining possibilities with each step, similar in spirit to binary search.

### Example: Remove duplicates from a sorted array in-place (same-direction two pointers)
```js
function removeDuplicates(nums) {
  if (nums.length === 0) return 0;

  let slow = 0; // points to the last unique element's position

  for (let fast = 1; fast < nums.length; fast++) {
    if (nums[fast] !== nums[slow]) {
      slow++;
      nums[slow] = nums[fast]; // overwrite with the new unique value
    }
  }
  return slow + 1; // length of the deduplicated portion
}

const arr = [1, 1, 2, 2, 3];
removeDuplicates(arr); // returns 3, arr becomes [1, 2, 3, 2, 3] (only first 3 matter)
```

### Two Pointer variations summary
| Variation | Movement | Typical use |
|---|---|---|
| Opposite ends (converging) | `left` starts at 0, `right` at end, move toward each other | Pair-sum problems on sorted arrays, palindrome checks |
| Same direction (fast/slow) | Both start at 0, move forward at different rates/conditions | Removing duplicates in-place, cycle detection in linked lists |

---

## Quick Pattern Recognition Cheat Sheet

| If the problem mentions... | Consider using... |
|---|---|
| "Have I seen this before?" / counting occurrences | HashMap |
| Pair/triplet that sums to a target, on a **sorted** array | Two Pointer |
| Contiguous subarray/substring with a max/min/target condition | Sliding Window |
| Matching brackets, undo functionality, DFS | Stack |
| Level-order traversal, shortest path in unweighted graph, BFS | Queue |
| Reversing, cycle detection, "middle of the list" | Linked List + Two Pointer (fast/slow) |
| In-place array modification, removing duplicates | Two Pointer |

## Complexity Cheat Sheet

| Structure | Access | Search | Insert | Delete |
|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) worst / O(1) at end | O(n) worst / O(1) at end |
| Linked List | O(n) | O(n) | O(1) (with reference) | O(1) (with reference) |
| Hash Map | — | O(1) avg | O(1) avg | O(1) avg |
| Stack | O(n) (top only fast) | O(n) | O(1) (top) | O(1) (top) |
| Queue | O(n) (front only fast) | O(n) | O(1) (back) | O(1) (front) |

## General Interview Problem-Solving Approach

1. **Clarify the problem** — ask about input size, edge cases (empty input, duplicates, negative numbers), and expected output format.
2. **Start with brute force** — describe the naive O(n²) or worse solution first, out loud, to show you understand the problem correctly.
3. **Identify the pattern** — use the recognition table above to spot whether HashMap, Two Pointer, Sliding Window, Stack, or Queue naturally fits.
4. **Optimize** — explain *why* your optimized approach reduces time/space complexity compared to brute force.
5. **Code it cleanly** — use clear variable names (`left`/`right`, `slow`/`fast`, `windowStart`/`windowEnd` are conventional and communicate intent).
6. **Test with edge cases** — empty array, single element, all duplicates, already sorted/reverse sorted, etc.
7. **State final time/space complexity** explicitly at the end.
