# Learn DSA — Big Pattern → Question Type → Small Recipe → Java

> Everything in SkillForge's **Learn** section: 6 phases, 26 big patterns and 132 question types, in study order. Exported from the app on 2026-10-05.

**How to read it.** Every problem belongs to a **big pattern** (Hashing, Two Pointers, Heap…). Inside a pattern, each **question type** answers one question you ask yourself while reading a problem ("Have I seen this value before?"). Each question type has one **small recipe** — the data structure or move that answers it (HashSet) — and the **Java** that implements it. In an interview, go down the same ladder: *which pattern? → which question am I really asking? → which recipe? → write the Java.*

## Contents

- [Cheat sheet — all patterns on one page](#cheat-sheet--all-patterns-on-one-page)
- [Phase 1 — Arrays, Strings & Hashing](#phase-1--arrays-strings--hashing)
  - [1. Hashing](#1-hashing)
  - [2. Frequency Counting](#2-frequency-counting)
  - [3. Prefix Sum](#3-prefix-sum)
  - [4. Kadane](#4-kadane)
  - [5. Two Pointers](#5-two-pointers)
  - [6. Sliding Window (Fixed)](#6-sliding-window-fixed)
  - [7. Sliding Window (Dynamic)](#7-sliding-window-dynamic)
- [Phase 2 — Stacks, Queues & Heaps](#phase-2--stacks-queues--heaps)
  - [8. Stack](#8-stack)
  - [9. Monotonic Stack](#9-monotonic-stack)
  - [10. Queue](#10-queue)
  - [11. Monotonic Queue](#11-monotonic-queue)
  - [12. Heap (Priority Queue)](#12-heap-priority-queue)
  - [13. Top K](#13-top-k)
- [Phase 3 — Sorting, Searching & Greedy](#phase-3--sorting-searching--greedy)
  - [14. Intervals](#14-intervals)
  - [15. Binary Search](#15-binary-search)
  - [16. Greedy](#16-greedy)
- [Phase 4 — Linked Lists, Trees & Tries](#phase-4--linked-lists-trees--tries)
  - [17. Linked List](#17-linked-list)
  - [18. Trees](#18-trees)
  - [19. Trie (Prefix Tree)](#19-trie-prefix-tree)
- [Phase 5 — Graphs](#phase-5--graphs)
  - [20. BFS / DFS](#20-bfs--dfs)
  - [21. Graphs](#21-graphs)
  - [22. Topological Sort](#22-topological-sort)
  - [23. Union-Find (Disjoint Set)](#23-union-find-disjoint-set)
- [Phase 6 — Recursion, DP & Bits](#phase-6--recursion-dp--bits)
  - [24. Backtracking](#24-backtracking)
  - [25. Dynamic Programming](#25-dynamic-programming)
  - [26. Bit Manipulation](#26-bit-manipulation)

## Cheat sheet — all patterns on one page

### Phase 1 · Arrays, Strings & Hashing

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **Hashing** | Seen before / duplicate detection | Have I seen this value before? | `HashSet` |
|  | Complement lookup | What partner value do I need? | `HashMap` |
|  | Fast lookup | Can I replace repeated scanning with memory? | `HashMap / HashSet` |
|  | Grouping | Which items belong in the same bucket? | `HashMap<Key, List<...>>` |
| **Frequency Counting** | Count occurrences | How many times did each thing appear? | `HashMap<T,Integer>` |
|  | Small bounded values | Are values limited to a tiny known range? | `Frequency array` |
|  | Most / least frequent | After counting, which count wins? | `Frequency map + scan/heap` |
|  | Anagram equality | Do both strings contain exactly the same counts? | `Compare frequencies` |
| **Prefix Sum** | Range sum query | Can I answer many subarray sums instantly? | `prefix[r+1] - prefix[l]` |
|  | Subarray sum equals K | Have I seen prefixSum-K before? | `Prefix sum + HashMap` |
|  | Zero-sum subarray | Did the running total repeat? | `Repeated prefix sum` |
|  | 2D rectangle sum | Can I reuse precomputed rectangle totals? | `2D prefix matrix` |
| **Kadane** | Maximum subarray | Should I extend the old subarray or restart here? | `Running best` |
|  | Minimum subarray | What is the worst contiguous segment? | `Running minimum` |
|  | Track actual range | Where did the best run start and end? | `Kadane + indices` |
|  | Circular maximum subarray | Can the answer wrap around the end? | `total - minimum subarray` |
| **Two Pointers** | Pair in sorted array | Can two ends move based on whether the sum is small or large? | `Left + right pointers` |
|  | Palindrome | Do characters match while pointers move inward? | `Compare both ends` |
|  | Remove duplicates in-place | Can one pointer read while another writes only valid items? | `Read + write pointer` |
|  | Container / max area | Which side limits the current answer? | `Move limiting side` |
| **Sliding Window (Fixed)** | Window sum / average | Is the window size always exactly K? | `Add right, remove left` |
|  | Count condition in every K window | Can I update the answer by only considering entering/leaving items? | `Running count` |
|  | Find anagrams | Does every length-K window have the target counts? | `Window frequency map` |
|  | Window max/min | Do I need the best value for every fixed window? | `Monotonic deque` |
| **Sliding Window (Dynamic)** | Longest valid window | Can I grow until a rule breaks, then repair from the left? | `Expand right, shrink when invalid` |
|  | Smallest valid window | Once valid, can I make it smaller? | `Expand until valid, shrink aggressively` |
|  | At most K distinct | Does validity depend on how many distinct values are inside? | `Frequency map + left pointer` |
|  | No repeats | When a duplicate appears, where should left jump? | `Set / last-seen map` |

### Phase 2 · Stacks, Queues & Heaps

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **Stack** | Balanced brackets | Does the newest unfinished thing need to finish first? | `Push opens, match closes` |
|  | Expression evaluation | Do nested operations resolve in LIFO order? | `Operand/operator stack` |
|  | Undo history | Should the most recent action be undone first? | `Push actions` |
|  | Iterative DFS | Can I replace recursion with my own stack? | `Explicit stack` |
| **Monotonic Stack** | Next greater element | Who is the first bigger value to my right? | `Decreasing stack` |
|  | Next smaller element | Who is the first smaller value to my right? | `Increasing stack` |
|  | Previous greater/smaller | Who is the nearest valid value on my left? | `Clean stack then peek` |
|  | Daily Temperatures | When will a warmer day arrive? | `Decreasing stack of indices` |
|  | Largest Rectangle | When a bar drops, which taller rectangles are now complete? | `Increasing stack + widths` |
| **Queue** | FIFO processing | Should the oldest waiting item go first? | `offer → poll` |
|  | Tree level order | Do I need to process all nodes at one depth together? | `Queue by level` |
|  | BFS frontier | Do I explore distance 1 before distance 2? | `Queue` |
|  | Work buffer | Should new work wait behind older work? | `Producer/consumer queue` |
| **Monotonic Queue** | Sliding window maximum | Can I keep only candidates that could still become maximum? | `Decreasing deque` |
|  | Sliding window minimum | Can I keep only candidates that could still become minimum? | `Increasing deque` |
|  | Expire old values | How do I know when a candidate leaves the window? | `Store indices` |
|  | DP window optimization | Am I repeatedly taking max/min over a moving DP range? | `Deque of best states` |
| **Heap (Priority Queue)** | Repeated smallest | Do I repeatedly need the smallest/earliest item? | `Min Heap` |
|  | Repeated largest | Do I repeatedly need the largest item? | `Max Heap` |
|  | Kth largest | Can I keep only the K largest seen so far? | `Min heap of size K` |
|  | Kth smallest | Can I keep only the K smallest seen so far? | `Max heap of size K` |
|  | Merge K sorted streams | Which stream currently has the smallest next item? | `Min heap of heads` |
|  | Scheduling by end time | Which occupied resource becomes free first? | `Min heap of end times` |
| **Top K** | Top K largest | Can I throw away everything outside the current top K? | `Min heap size K` |
|  | Top K smallest | Can I throw away everything larger than current top K smallest? | `Max heap size K` |
|  | Top K frequent | Can I count first, then rank by frequency? | `Frequency + heap/bucket` |
|  | K closest points | Can I keep only K smallest distances? | `Max heap size K` |
|  | Streaming Top K | Can I maintain top K without storing the whole stream? | `Bounded heap` |

### Phase 3 · Sorting, Searching & Greedy

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **Intervals** | Detect any overlap | After sorting, does current start before previous ends? | `Sort by start → check adjacent` |
|  | Merge overlaps | Should current interval extend the previous merged range? | `Sort by start → compare last merged` |
|  | Insert interval | Where does the new interval fit among existing ranges? | `Before → merge → after` |
|  | Minimum meeting rooms | Can a new meeting reuse the earliest-free room? | `Sort + min heap of end times` |
|  | Maximum simultaneous intervals | How many intervals are active at once? | `Sweep line / start-end pointers` |
|  | Remove minimum overlaps | Which interval should I keep to leave maximum future space? | `Sort by end → greedy` |
|  | Interval intersection | How do two sorted interval lists overlap? | `Two pointers` |
| **Binary Search** | Exact search | Can I discard half after one comparison? | `Classic binary search` |
|  | First / last occurrence | After finding target, should I keep searching one side? | `Biased binary search` |
|  | Lower / upper bound | Am I searching for the first position where a condition becomes true? | `Boundary binary search` |
|  | Rotated array | Which half is guaranteed sorted right now? | `Identify sorted half` |
|  | Binary search on answer | If X works, do all larger/smaller X also work? | `Monotonic feasibility` |
|  | Peak finding | Which direction is rising? | `Compare mid with neighbor` |
| **Greedy** | Interval scheduling | Which choice leaves the most room for future choices? | `Sort by end → take earliest finish` |
|  | Jump Game | What is the farthest index reachable so far? | `Track farthest reachable` |
|  | Gas station | If starting here fails, can any point inside this failed segment work? | `Reset after failed prefix` |
|  | Minimum arrows / overlap removal | Can one local endpoint cover as much future work as possible? | `Sort intervals strategically` |
|  | Activity selection | Can I safely commit to the earliest-finishing activity? | `Earliest compatible finish` |

### Phase 4 · Linked Lists, Trees & Tries

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **Linked List** | Reverse list | Can I safely flip one arrow at a time? | `prev / curr / next` |
|  | Middle node | Can one pointer move twice as fast? | `Slow + fast` |
|  | Cycle detection | Will runners meet if the track loops? | `Floyd slow/fast` |
|  | Merge sorted lists | Can I always attach the smaller front node? | `Dummy head + tail` |
|  | Remove nth from end | Can I keep pointers N nodes apart? | `Two pointers with gap` |
|  | Reorder list | Can I combine three known recipes? | `Middle → reverse → merge` |
| **Trees** | Preorder | Do I need the parent before its children? | `Node → left → right` |
|  | Inorder | Do I want BST values in sorted order? | `Left → node → right` |
|  | Postorder | Does parent depend on child results? | `Left → right → node` |
|  | Height / depth | Can each subtree report a small answer upward? | `DFS returns child result` |
|  | Path problems | Do I carry a running state from root to leaf? | `DFS + path state` |
|  | Lowest common ancestor | Where do two targets diverge? | `Recursive split / BST ordering` |
| **Trie (Prefix Tree)** | Insert word | Can shared prefixes share the same path? | `Create child per character` |
|  | Exact search | Did I reach a complete stored word? | `Walk + end marker` |
|  | Prefix search | Does any stored word begin with this prefix? | `Walk prefix only` |
|  | Autocomplete | After reaching prefix, what words live below it? | `Prefix node + DFS descendants` |
|  | Word Search II | Can prefix pruning avoid useless grid paths? | `Trie + grid DFS` |

### Phase 5 · Graphs

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **BFS / DFS** | Reachability | Can I mark everything reachable from a start? | `DFS or BFS + visited` |
|  | Shortest unweighted path | Does each edge cost the same? | `BFS` |
|  | Connected components | How many separate groups exist? | `Traversal from every unvisited node` |
|  | Flood fill / islands | Can I treat neighboring cells as graph edges? | `Grid DFS/BFS` |
|  | Tree levels | Do I need one depth at a time? | `BFS by queue size` |
| **Graphs** | Adjacency list | How should I store sparse edges? | `List of neighbors` |
|  | Undirected connectivity | Do I need exploration or dynamic grouping? | `DFS/BFS or Union Find` |
|  | Weighted shortest path | Are weights non-negative? | `Dijkstra` |
|  | Negative edges | Can edges have negative weights? | `Bellman-Ford` |
|  | All-pairs shortest path | Do I need distance between every pair? | `Floyd-Warshall` |
|  | Cycle detection | Is there a loop? | `DFS colors / Union Find` |
| **Topological Sort** | Course schedule possible? | Can prerequisites be satisfied without a cycle? | `Kahn / DFS cycle detection` |
|  | Produce dependency order | Which items have zero prerequisites now? | `Indegree queue` |
|  | Build order | What must happen before what? | `Topological ordering` |
|  | Cycle in directed graph | Did some nodes remain blocked forever? | `Processed count / DFS colors` |
| **Union-Find (Disjoint Set)** | Connectivity query | Do two items belong to the same group? | `find(a) == find(b)` |
|  | Merge groups | Can I join two components quickly? | `union(a,b)` |
|  | Path compression | Can future find operations become almost constant? | `Flatten find path` |
|  | Union by rank/size | Can I keep trees shallow? | `Attach smaller tree under bigger` |
|  | Redundant edge | Would this edge create a cycle? | `Already connected?` |
|  | Count components | How many groups remain after merges? | `Start n, decrement on union` |

### Phase 6 · Recursion, DP & Bits

| Big pattern | Question type | The question | Small recipe |
|---|---|---|---|
| **Backtracking** | Subsets | For each item, do I include it or not? | `Choose / skip` |
|  | Permutations | Who goes in the next position? | `Choose unused item` |
|  | Combination Sum | Can I build target while pruning impossible branches? | `Choose candidate repeatedly` |
|  | N-Queens | Can I safely place one queen in this row? | `Place → validate → recurse → remove` |
|  | Sudoku | Which digit can legally go here? | `Fill → recurse → undo` |
|  | Word Search | Can this path spell the word without reusing a cell? | `Grid DFS + temporary visited` |
| **Dynamic Programming** | 1D choose / skip | Is today's best answer based on taking or skipping current item? | `dp[i] from earlier states` |
|  | Knapsack | Do I choose this item under a capacity constraint? | `Item × capacity state` |
|  | Grid DP | Can each cell be solved from already solved nearby cells? | `Reuse top/left/neighbors` |
|  | LCS | Do two prefixes end with matching characters? | `2D prefix DP` |
|  | LIS | What is the best increasing sequence ending here? | `DP or tails + binary search` |
|  | Coin Change | Can I build this amount from smaller solved amounts? | `Amount state` |
|  | Memoization | Am I solving the same state again? | `Recursive state + cache` |
| **Bit Manipulation** | Check bit | Is bit k ON? | `x & (1<<k)` |
|  | Set bit | How do I turn bit k ON? | `x \| (1<<k)` |
|  | Clear bit | How do I turn bit k OFF? | `x & ~(1<<k)` |
|  | Toggle bit | How do I flip bit k? | `x ^ (1<<k)` |
|  | Single Number | Can pairs cancel each other? | `XOR everything` |
|  | Power of two | Does n have exactly one set bit? | `n & (n-1)` |
|  | Enumerate subsets | Can each bit represent include/exclude? | `Bitmask 0..2^n-1` |

## Phase 1 — Arrays, Strings & Hashing

### 1. Hashing

**The big idea:** A HashSet or HashMap is a notebook that answers "have I seen this?" or "what did I store for this?" in almost no time — O(1) on average.

**Think of it like this:** Like a school register: instead of searching every desk for a student, you look up the name and instantly know if they are present.

**How to spot it:**
- "Have I seen this before?" / duplicates
- Pair that adds up to a target
- Group items that belong together
- A nested loop that searches again and again

**Template:**

```java
Set<Integer> seen = new HashSet<>();
for (int x : nums) {
    if (seen.contains(x)) {
        // x appeared before
    }
    seen.add(x);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Seen before / duplicate detection](#11-seen-before--duplicate-detection) | Have I seen this value before? | `HashSet` | O(n) |
| [Complement lookup](#12-complement-lookup) | What partner value do I need? | `HashMap` | O(n) |
| [Fast lookup](#13-fast-lookup) | Can I replace repeated scanning with memory? | `HashMap / HashSet` | O(n + m) |
| [Grouping](#14-grouping) | Which items belong in the same bucket? | `HashMap<Key, List<...>>` | O(n · L log L) |

#### 1.1 Seen before / duplicate detection

- **Big pattern:** Hashing
- **Ask yourself:** *Have I seen this value before?*
- **Small recipe:** `HashSet`
- **Memory sentence:** Hashing → Seen Before / Duplicate Detection → seen before → Set
- **Words that give it away:** duplicate, seen before, lookup, pair, group

**In plain English:** Keep a set of every value you have already met. Before adding the next value, ask the set: "do you already have this?"

**Steps:**
1. Create an empty HashSet.
2. For each value, check the set first.
3. If it is there → duplicate. Otherwise add it and move on.

**Tiny example:**

```text
nums = [4, 2, 7, 2]
4 → set {4}
2 → set {4, 2}
7 → set {4, 2, 7}
2 → already in set → return true
```

**Java:**

```java
boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int x : nums) {
        if (!seen.add(x)) {   // add() returns false if x was already there
            return true;
        }
    }
    return false;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Check BEFORE inserting if duplicate detection matters.

**Practise on:** Contains Duplicate · Two Sum · Group Anagrams

#### 1.2 Complement lookup

- **Big pattern:** Hashing
- **Ask yourself:** *What partner value do I need?*
- **Small recipe:** `HashMap`
- **Memory sentence:** Hashing → Complement Lookup → need = target - current
- **Words that give it away:** duplicate, seen before, lookup, pair, group

**In plain English:** For each number, work out the partner you need (target − current) and ask the map whether that partner appeared earlier.

**Steps:**
1. Map stores value → index of numbers already seen.
2. need = target − current.
3. If need is in the map → answer found.
4. Otherwise store current and continue.

**Tiny example:**

```text
nums = [2, 7, 11, 15], target = 9
i=0: need 7, map {} → store 2
i=1: need 2, map {2:0} → found! [0, 1]
```

**Java:**

```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> indexOf = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (indexOf.containsKey(need)) {
            return new int[]{indexOf.get(need), i};
        }
        indexOf.put(nums[i], i);   // store AFTER checking, so we never pair i with itself
    }
    return new int[]{-1, -1};
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Usually lookup first, then put current value, so you don't pair an item with itself.

**Practise on:** Contains Duplicate · Two Sum · Group Anagrams

#### 1.3 Fast lookup

- **Big pattern:** Hashing
- **Ask yourself:** *Can I replace repeated scanning with memory?*
- **Small recipe:** `HashMap / HashSet`
- **Memory sentence:** Hashing → Fast Membership Lookup → replace repeated scan with O(1) expected lookup
- **Words that give it away:** duplicate, seen before, lookup, pair, group

**In plain English:** If your code keeps searching a list with a loop, put the list into a HashSet once. Every later search becomes instant.

**Steps:**
1. Put the values you search into a HashSet (one pass).
2. Replace every "loop and compare" with set.contains(x).
3. Total work drops from O(n·m) to O(n + m).

**Tiny example:**

```text
banned = ["hit", "ball"], words = ["bob", "hit", "ball", "the"]
set = {hit, ball}
bob ✓ keep, hit ✗, ball ✗, the ✓ keep
```

**Java:**

```java
List<String> removeBanned(String[] words, String[] banned) {
    Set<String> bannedSet = new HashSet<>(Arrays.asList(banned));
    List<String> kept = new ArrayList<>();
    for (String w : words) {
        if (!bannedSet.contains(w)) {   // O(1) instead of scanning banned[]
            kept.add(w);
        }
    }
    return kept;
}
```

**Complexity:** time O(n + m) · space O(m)

> ⚠️ **Trap:** HashSet has no duplicates and no meaningful order.

**Practise on:** Contains Duplicate · Two Sum · Group Anagrams · Membership / whitelist checks

#### 1.4 Grouping

- **Big pattern:** Hashing
- **Ask yourself:** *Which items belong in the same bucket?*
- **Small recipe:** `HashMap<Key, List<...>>`
- **Memory sentence:** Hashing → Grouping by Key → compute key → append to bucket
- **Words that give it away:** duplicate, seen before, lookup, pair, group

**In plain English:** Give every item a "label" (key) so that items belonging together get the same label. Then drop each item into the bucket for its label.

**Steps:**
1. Decide the key: e.g. the sorted letters of a word.
2. map.computeIfAbsent(key, new list).add(item).
3. The map values are your groups.

**Tiny example:**

```text
words = [eat, tea, tan, ate]
eat → key "aet" → {aet:[eat]}
tea → key "aet" → {aet:[eat, tea]}
tan → key "ant" → new bucket
ate → key "aet" → [eat, tea, ate]
```

**Java:**

```java
List<List<String>> groupAnagrams(String[] words) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String w : words) {
        char[] letters = w.toCharArray();
        Arrays.sort(letters);
        String key = new String(letters);          // same letters → same key
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(w);
    }
    return new ArrayList<>(groups.values());
}
```

**Complexity:** time O(n · L log L) · space O(n · L)

> ⚠️ **Trap:** The hardest part is usually designing a stable grouping key.

**Practise on:** Contains Duplicate · Two Sum · Group Anagrams

**Runnable in SkillForge (Hashing):** Two Sum (Easy) · Contains Duplicate (Easy) · Group Anagrams (Medium) · Longest Consecutive Sequence (Medium) · Missing Number (Easy) · First Missing Positive (Hard)

---

### 2. Frequency Counting

**The big idea:** Count how many times each thing appears, in one pass. Then answer questions using the counts instead of re-scanning.

**Think of it like this:** Like a tally sheet at an election: one mark per vote, and at the end you just read the totals.

**How to spot it:**
- "How many times…"
- Anagram / same letters
- Most or least frequent
- Values in a small range (a–z, 0–100)

**Template:**

```java
Map<Integer, Integer> count = new HashMap<>();
for (int x : nums) {
    count.put(x, count.getOrDefault(x, 0) + 1);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Count occurrences](#21-count-occurrences) | How many times did each thing appear? | `HashMap<T,Integer>` | O(n) |
| [Small bounded values](#22-small-bounded-values) | Are values limited to a tiny known range? | `Frequency array` | O(n) |
| [Most / least frequent](#23-most--least-frequent) | After counting, which count wins? | `Frequency map + scan/heap` | O(n) |
| [Anagram equality](#24-anagram-equality) | Do both strings contain exactly the same counts? | `Compare frequencies` | O(n) |

#### 2.1 Count occurrences

- **Big pattern:** Frequency Counting
- **Ask yourself:** *How many times did each thing appear?*
- **Small recipe:** `HashMap<T,Integer>`
- **Memory sentence:** Frequency Counting → HashMap Frequency Count → count[x]++
- **Words that give it away:** count, frequency, most common, anagram

**In plain English:** Walk once. For each item, add 1 to its counter in a HashMap.

**Steps:**
1. Create HashMap<item, count>.
2. For each item: count = old count (or 0) + 1.
3. Read the answers from the map.

**Tiny example:**

```text
[a, b, a, c, a]
a:1 → b:1 → a:2 → c:1 → a:3
```

**Java:**

```java
Map<String, Integer> countWords(String[] words) {
    Map<String, Integer> count = new HashMap<>();
    for (String w : words) {
        count.put(w, count.getOrDefault(w, 0) + 1);
        // Java 8 short form: count.merge(w, 1, Integer::sum);
    }
    return count;
}
```

**Complexity:** time O(n) · space O(distinct items)

> ⚠️ **Trap:** getOrDefault is the interview-friendly default.

**Practise on:** Valid Anagram · Top K Frequent Elements · Ransom Note · Count occurrences

#### 2.2 Small bounded values

- **Big pattern:** Frequency Counting
- **Ask yourself:** *Are values limited to a tiny known range?*
- **Small recipe:** `Frequency array`
- **Memory sentence:** Frequency Counting → Frequency Array → small bounded domain → array count
- **Words that give it away:** count, frequency, most common, anagram

**In plain English:** When values are small and known (like letters a–z or scores 0–100), use a plain int array as the counter. Index = value.

**Steps:**
1. Create int[range size].
2. count[value]++ (for letters: count[c - 'a']++).
3. Loop over the array to read counts in sorted order for free.

**Tiny example:**

```text
s = "banana"
count['a'-'a'=0] = 3, count['b'-'a'=1] = 1, count['n'-'a'=13] = 2
```

**Java:**

```java
int[] letterCounts(String s) {
    int[] count = new int[26];            // one slot per lowercase letter
    for (char c : s.toCharArray()) {
        count[c - 'a']++;                 // 'a'→0, 'b'→1, ... 'z'→25
    }
    return count;
}
```

**Complexity:** time O(n) · space O(range) — here O(26) = O(1)

> ⚠️ **Trap:** Only use when value range is safe and known.

**Practise on:** Valid Anagram · Top K Frequent Elements · Ransom Note · Scores / lowercase letters / bounded values

#### 2.3 Most / least frequent

- **Big pattern:** Frequency Counting
- **Ask yourself:** *After counting, which count wins?*
- **Small recipe:** `Frequency map + scan/heap`
- **Memory sentence:** Frequency Counting → Character Count → char → index
- **Words that give it away:** count, frequency, most common, anagram

**In plain English:** First count everything. Then scan the counts once to find the winner (or use a heap if you need the top K).

**Steps:**
1. Build the frequency map.
2. Loop over map entries, remember the best count and its key.
3. For top-K, see the Top K pattern.

**Tiny example:**

```text
[1, 3, 1, 2, 3, 1]
counts {1:3, 3:2, 2:1}
best so far: 1 (count 3)
```

**Java:**

```java
int mostFrequent(int[] nums) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int x : nums) count.merge(x, 1, Integer::sum);

    int best = nums[0], bestCount = 0;
    for (Map.Entry<Integer, Integer> e : count.entrySet()) {
        if (e.getValue() > bestCount) {
            bestCount = e.getValue();
            best = e.getKey();
        }
    }
    return best;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Works only for expected alphabet, e.g. lowercase a-z.

**Practise on:** Valid Anagram · Top K Frequent Elements · Ransom Note · Anagram / character frequency

#### 2.4 Anagram equality

- **Big pattern:** Frequency Counting
- **Ask yourself:** *Do both strings contain exactly the same counts?*
- **Small recipe:** `Compare frequencies`
- **Memory sentence:** Frequency Counting → Compare Frequency Maps → build counts → compare all buckets
- **Words that give it away:** count, frequency, most common, anagram

**In plain English:** Two words are anagrams if every letter appears the same number of times. Add counts for the first word, subtract for the second — everything must end at zero.

**Steps:**
1. Different lengths → not anagrams.
2. count[s[i]]++ and count[t[i]]-- in the same loop.
3. All 26 counts must be 0.

**Tiny example:**

```text
listen vs silent
after the loop every letter count is 0 → anagram ✓
```

**Java:**

```java
boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (int i = 0; i < s.length(); i++) {
        count[s.charAt(i) - 'a']++;
        count[t.charAt(i) - 'a']--;
    }
    for (int c : count) {
        if (c != 0) return false;
    }
    return true;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Check equal lengths first.

**Practise on:** Valid Anagram · Top K Frequent Elements · Ransom Note

**Runnable in SkillForge (Frequency Counting):** Character Frequency (Easy) · Valid Anagram (Easy) · Count Vowels (Easy) · First Unique Character in a String (Easy) · Majority Element (Easy)

---

### 3. Prefix Sum

**The big idea:** Store running totals once. Then the sum of ANY range is just one subtraction: total up to the right end minus total before the left end.

**Think of it like this:** Like a car’s odometer: to know how far you drove between two towns, subtract the two readings — no need to re-drive the road.

**How to spot it:**
- Many "sum from l to r" questions
- Count subarrays whose sum is K
- Works with negative numbers (sliding window does not)
- Sum of a rectangle in a grid

**Template:**

```java
long[] prefix = new long[n + 1];
for (int i = 0; i < n; i++) prefix[i + 1] = prefix[i] + nums[i];
// sum of nums[l..r] = prefix[r + 1] - prefix[l]
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Range sum query](#31-range-sum-query) | Can I answer many subarray sums instantly? | `prefix[r+1] - prefix[l]` | O(n) build, O(1) per query |
| [Subarray sum equals K](#32-subarray-sum-equals-k) | Have I seen prefixSum-K before? | `Prefix sum + HashMap` | O(n) |
| [Zero-sum subarray](#33-zero-sum-subarray) | Did the running total repeat? | `Repeated prefix sum` | O(n) |
| [2D rectangle sum](#34-2d-rectangle-sum) | Can I reuse precomputed rectangle totals? | `2D prefix matrix` | O(R·C) build, O(1) per query |

#### 3.1 Range sum query

- **Big pattern:** Prefix Sum
- **Ask yourself:** *Can I answer many subarray sums instantly?*
- **Small recipe:** `prefix[r+1] - prefix[l]`
- **Memory sentence:** Prefix Sum → Range Sum → sum(l..r) = prefix[r+1] - prefix[l]
- **Words that give it away:** range sum, subarray sum, many queries, cumulative

**In plain English:** Build prefix[] once (size n+1, prefix[0] = 0). Each range sum is prefix[r+1] − prefix[l], answered instantly.

**Steps:**
1. prefix[0] = 0.
2. prefix[i+1] = prefix[i] + nums[i].
3. sum(l..r) = prefix[r+1] − prefix[l].

**Tiny example:**

```text
nums = [2, 4, 1, 3]
prefix = [0, 2, 6, 7, 10]
sum(1..3) = 10 − 2 = 8
```

**Java:**

```java
class RangeSum {
    private final long[] prefix;

    RangeSum(int[] nums) {
        prefix = new long[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    long sum(int l, int r) {          // inclusive l..r
        return prefix[r + 1] - prefix[l];
    }
}
```

**Complexity:** time O(n) build, O(1) per query · space O(n)

> ⚠️ **Trap:** Use inclusive right only if your formula matches it.

**Practise on:** Range Sum Query · Subarray Sum Equals K · Continuous Subarray Sum · Many range sum queries

#### 3.2 Subarray sum equals K

- **Big pattern:** Prefix Sum
- **Ask yourself:** *Have I seen prefixSum-K before?*
- **Small recipe:** `Prefix sum + HashMap`
- **Memory sentence:** Prefix Sum → Subarray Sum Equals K → currentPrefix - k seen before?
- **Words that give it away:** range sum, subarray sum, many queries, cumulative

**In plain English:** A subarray ending here has sum K if some earlier running total equals (current total − K). Count earlier totals in a HashMap.

**Steps:**
1. seen = {0: 1} (the empty prefix).
2. Add each number to the running total.
3. answer += seen[total − K].
4. seen[total]++.

**Tiny example:**

```text
nums = [1, 2, 3], K = 3
total 1: need −2 (0 times) → seen {0:1, 1:1}
total 3: need 0 (1 time) → count 1 → [1,2]
total 6: need 3 (1 time) → count 2 → [3]
```

**Java:**

```java
int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> seen = new HashMap<>();
    seen.put(0, 1);                   // handles subarrays starting at index 0
    int total = 0, count = 0;
    for (int x : nums) {
        total += x;
        count += seen.getOrDefault(total - k, 0);
        seen.merge(total, 1, Integer::sum);
    }
    return count;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** The seed seen.put(0,1) handles subarrays starting at index 0.

**Practise on:** Range Sum Query · Subarray Sum Equals K · Continuous Subarray Sum

#### 3.3 Zero-sum subarray

- **Big pattern:** Prefix Sum
- **Ask yourself:** *Did the running total repeat?*
- **Small recipe:** `Repeated prefix sum`
- **Memory sentence:** Prefix Sum → Zero-sum subarray → Repeated prefix sum
- **Words that give it away:** range sum, subarray sum, many queries, cumulative

**In plain English:** If the running total is the same at two positions, the numbers between them add up to zero.

**Steps:**
1. Keep a set of running totals, starting with 0.
2. After adding each number, check if the total was seen before.
3. Seen before → a zero-sum subarray exists.

**Tiny example:**

```text
nums = [4, 2, −3, 1, 6]
totals: 0, 4, 6, 3, 4 ← 4 repeats!
so [2, −3, 1] sums to 0
```

**Java:**

```java
boolean hasZeroSumSubarray(int[] nums) {
    Set<Long> totals = new HashSet<>();
    totals.add(0L);
    long total = 0;
    for (int x : nums) {
        total += x;
        if (!totals.add(total)) {     // total repeated → middle part sums to 0
            return true;
        }
    }
    return false;
}
```

**Complexity:** time O(n) · space O(n)

**Practise on:** Range Sum Query · Subarray Sum Equals K · Continuous Subarray Sum

#### 3.4 2D rectangle sum

- **Big pattern:** Prefix Sum
- **Ask yourself:** *Can I reuse precomputed rectangle totals?*
- **Small recipe:** `2D prefix matrix`
- **Memory sentence:** Prefix Sum → 2D Prefix Sum → rectangle sum by inclusion-exclusion
- **Words that give it away:** range sum, subarray sum, many queries, cumulative

**In plain English:** P[r][c] holds the sum of the rectangle from the top-left corner to (r−1, c−1). Any rectangle = big box − top strip − left strip + the corner you removed twice.

**Steps:**
1. P has one extra row and column of zeros.
2. P[r+1][c+1] = grid[r][c] + P[r][c+1] + P[r+1][c] − P[r][c].
3. sum(r1,c1..r2,c2) = P[r2+1][c2+1] − P[r1][c2+1] − P[r2+1][c1] + P[r1][c1].

**Tiny example:**

```text
The corner P[r1][c1] is subtracted twice (top and left strips) — so add it back once.
```

**Java:**

```java
long[][] build2D(int[][] grid) {
    int rows = grid.length, cols = grid[0].length;
    long[][] p = new long[rows + 1][cols + 1];
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            p[r + 1][c + 1] = grid[r][c] + p[r][c + 1] + p[r + 1][c] - p[r][c];
        }
    }
    return p;
}

long rectSum(long[][] p, int r1, int c1, int r2, int c2) {
    return p[r2 + 1][c2 + 1] - p[r1][c2 + 1] - p[r2 + 1][c1] + p[r1][c1];
}
```

**Complexity:** time O(R·C) build, O(1) per query · space O(R·C)

> ⚠️ **Trap:** Remember the overlap is added twice, so subtract it once.

**Practise on:** Range Sum Query · Subarray Sum Equals K · Continuous Subarray Sum · Matrix range sum

**Runnable in SkillForge (Prefix Sum):** Running (Prefix) Sums (Easy) · Range Sum Queries (Easy) · Subarray Sum Equals K (Medium) · Product of Array Except Self (Medium)

---

### 4. Kadane

**The big idea:** Walk through the array keeping the best sum of a subarray that ENDS here. At each number choose: continue the old run, or start fresh from this number.

**Think of it like this:** Like a savings streak: if your running balance has gone negative, it only drags you down — start a new streak today.

**How to spot it:**
- Maximum (or minimum) sum of a contiguous subarray
- Best profit / best streak
- Circular array maximum

**Template:**

```java
int cur = nums[0], best = nums[0];
for (int i = 1; i < nums.length; i++) {
    cur = Math.max(nums[i], cur + nums[i]);   // restart or extend
    best = Math.max(best, cur);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Maximum subarray](#41-maximum-subarray) | Should I extend the old subarray or restart here? | `Running best` | O(n) |
| [Minimum subarray](#42-minimum-subarray) | What is the worst contiguous segment? | `Running minimum` | O(n) |
| [Track actual range](#43-track-actual-range) | Where did the best run start and end? | `Kadane + indices` | O(n) |
| [Circular maximum subarray](#44-circular-maximum-subarray) | Can the answer wrap around the end? | `total - minimum subarray` | O(n) |

#### 4.1 Maximum subarray

- **Big pattern:** Kadane
- **Ask yourself:** *Should I extend the old subarray or restart here?*
- **Small recipe:** `Running best`
- **Memory sentence:** Kadane → Maximum Subarray → extend old run OR restart here
- **Words that give it away:** maximum contiguous sum, minimum contiguous sum, best subarray

**In plain English:** cur = best sum ending at this index = max(this number alone, cur + this number). Track the best cur ever seen.

**Steps:**
1. Start cur and best at nums[0] (not 0!).
2. cur = max(x, cur + x).
3. best = max(best, cur).

**Tiny example:**

```text
[−2, 1, −3, 4, −1, 2, 1, −5, 4]
cur: −2, 1, −2, 4, 3, 5, 6, 1, 5
best = 6
```

**Java:**

```java
int maxSubArray(int[] nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);
        best = Math.max(best, cur);
    }
    return best;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Initialize from nums[0], not 0, or all-negative arrays break.

**Practise on:** Maximum Subarray · Maximum Sum Circular Subarray

#### 4.2 Minimum subarray

- **Big pattern:** Kadane
- **Ask yourself:** *What is the worst contiguous segment?*
- **Small recipe:** `Running minimum`
- **Memory sentence:** Kadane → Minimum Subarray → take smaller of restart vs extend
- **Words that give it away:** maximum contiguous sum, minimum contiguous sum, best subarray

**In plain English:** Same idea, flipped: keep the smallest sum ending here. min(this number alone, cur + this number).

**Steps:**
1. cur = best = nums[0].
2. cur = min(x, cur + x).
3. best = min(best, cur).

**Tiny example:**

```text
[3, −4, 2, −3, −1, 7]
cur: 3, −4, −2, −5, −6, 1
worst segment = −6 → [−4, 2, −3, −1]
```

**Java:**

```java
int minSubArray(int[] nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.min(nums[i], cur + nums[i]);
        best = Math.min(best, cur);
    }
    return best;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Same pattern, flip max to min.

**Practise on:** Maximum Subarray · Maximum Sum Circular Subarray · Minimum contiguous sum

#### 4.3 Track actual range

- **Big pattern:** Kadane
- **Ask yourself:** *Where did the best run start and end?*
- **Small recipe:** `Kadane + indices`
- **Memory sentence:** Kadane → Track Best Range → reset candidate start when restarting
- **Words that give it away:** maximum contiguous sum, minimum contiguous sum, best subarray

**In plain English:** Remember where the current run started. Whenever you restart, the new start is this index. Whenever best improves, save the start and end.

**Steps:**
1. candidateStart = 0.
2. If x alone beats cur + x → restart, candidateStart = i.
3. If cur beats best → bestStart = candidateStart, bestEnd = i.

**Tiny example:**

```text
[−2, 1, −3, 4, −1, 2, 1]
restart at i=3 (value 4)
best 6 reached at i=6 → range [3, 6]
```

**Java:**

```java
int[] maxSubArrayRange(int[] nums) {
    int cur = nums[0], best = nums[0];
    int candidateStart = 0, bestStart = 0, bestEnd = 0;
    for (int i = 1; i < nums.length; i++) {
        if (nums[i] > cur + nums[i]) {       // restarting is better
            cur = nums[i];
            candidateStart = i;
        } else {
            cur += nums[i];
        }
        if (cur > best) {
            best = cur;
            bestStart = candidateStart;
            bestEnd = i;
        }
    }
    return new int[]{bestStart, bestEnd, best};
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Separate current candidate start from best start.

**Practise on:** Maximum Subarray · Maximum Sum Circular Subarray · Return subarray indices

#### 4.4 Circular maximum subarray

- **Big pattern:** Kadane
- **Ask yourself:** *Can the answer wrap around the end?*
- **Small recipe:** `total - minimum subarray`
- **Memory sentence:** Kadane → Circular maximum subarray → total - minimum subarray
- **Words that give it away:** maximum contiguous sum, minimum contiguous sum, best subarray

**In plain English:** A wrapping subarray = everything EXCEPT a middle piece. So best wrap = total − (worst middle piece). Answer = max(normal Kadane, total − min subarray). If all numbers are negative, use normal Kadane.

**Steps:**
1. Run max-Kadane and min-Kadane together, and sum the total.
2. If maxSum < 0, every number is negative → return maxSum.
3. Else return max(maxSum, total − minSum).

**Tiny example:**

```text
[5, −3, 5]
max Kadane = 7, total = 7, min subarray = −3
wrap = 7 − (−3) = 10 → [5, 5] across the end
```

**Java:**

```java
int maxSubarraySumCircular(int[] nums) {
    int total = 0;
    int curMax = 0, maxSum = nums[0];
    int curMin = 0, minSum = nums[0];
    for (int x : nums) {
        curMax = Math.max(x, curMax + x);
        maxSum = Math.max(maxSum, curMax);
        curMin = Math.min(x, curMin + x);
        minSum = Math.min(minSum, curMin);
        total += x;
    }
    return maxSum < 0 ? maxSum : Math.max(maxSum, total - minSum);
}
```

**Complexity:** time O(n) · space O(1)

**Practise on:** Maximum Subarray · Maximum Sum Circular Subarray

**Runnable in SkillForge (Kadane):** Maximum Subarray (Kadane's) (Medium) · Best Time to Buy and Sell Stock (Easy)

---

### 5. Two Pointers

**The big idea:** Use two indexes that move through the data together — often from both ends toward the middle. Each move throws away options that can never be the answer.

**Think of it like this:** Like two friends searching a sorted bookshelf from both ends: depending on what they see, only one of them needs to step inward.

**How to spot it:**
- Sorted array + pair / triplet with a target
- Palindrome checks
- Remove or move items in place
- Compare from both ends

**Template:**

```java
int left = 0, right = nums.length - 1;
while (left < right) {
    int sum = nums[left] + nums[right];
    if (sum == target) return new int[]{left, right};
    if (sum < target) left++;   // need bigger
    else right--;               // need smaller
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Pair in sorted array](#51-pair-in-sorted-array) | Can two ends move based on whether the sum is small or large? | `Left + right pointers` | O(n) |
| [Palindrome](#52-palindrome) | Do characters match while pointers move inward? | `Compare both ends` | O(n) |
| [Remove duplicates in-place](#53-remove-duplicates-in-place) | Can one pointer read while another writes only valid items? | `Read + write pointer` | O(n) |
| [Container / max area](#54-container--max-area) | Which side limits the current answer? | `Move limiting side` | O(n) |

#### 5.1 Pair in sorted array

- **Big pattern:** Two Pointers
- **Ask yourself:** *Can two ends move based on whether the sum is small or large?*
- **Small recipe:** `Left + right pointers`
- **Memory sentence:** Two Pointers → Pair Sum in Sorted Array → sum too small → left++; too large → right--
- **Words that give it away:** sorted, pair, palindrome, both ends, in-place

**In plain English:** In a SORTED array, start at both ends. Sum too small → move the left pointer right (bigger). Too big → move the right pointer left (smaller).

**Steps:**
1. left = 0, right = n − 1.
2. Compare nums[left] + nums[right] with target.
3. Move exactly one pointer; stop when they meet.

**Tiny example:**

```text
[1, 2, 4, 6, 10], target 8
1+10=11 → R−−
1+6=7 → L++
2+6=8 ✓
```

**Java:**

```java
int[] pairWithSum(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) return new int[]{left, right};
        if (sum < target) left++;
        else right--;
    }
    return new int[]{-1, -1};
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** The array must be sorted for this directional move to be valid.

**Practise on:** Two Sum II · Valid Palindrome · Container With Most Water

#### 5.2 Palindrome

- **Big pattern:** Two Pointers
- **Ask yourself:** *Do characters match while pointers move inward?*
- **Small recipe:** `Compare both ends`
- **Memory sentence:** Two Pointers → Palindrome → compare ends → move inward
- **Words that give it away:** sorted, pair, palindrome, both ends, in-place

**In plain English:** Compare the first and last characters, then step both inward. Any mismatch → not a palindrome.

**Steps:**
1. i = 0, j = end.
2. Skip characters you are told to ignore (spaces, punctuation).
3. Compare lowercase chars; move both inward.

**Tiny example:**

```text
racecar
r=r → a=a → c=c → middle reached ✓
```

**Java:**

```java
boolean isPalindrome(String s) {
    int i = 0, j = s.length() - 1;
    while (i < j) {
        while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
        while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
        if (Character.toLowerCase(s.charAt(i)) != Character.toLowerCase(s.charAt(j))) {
            return false;
        }
        i++;
        j--;
    }
    return true;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** If ignoring punctuation/case, clean or skip invalid characters.

**Practise on:** Two Sum II · Valid Palindrome · Container With Most Water

#### 5.3 Remove duplicates in-place

- **Big pattern:** Two Pointers
- **Ask yourself:** *Can one pointer read while another writes only valid items?*
- **Small recipe:** `Read + write pointer`
- **Memory sentence:** Two Pointers → Read / Write Pointer → read everything, write only kept items
- **Words that give it away:** sorted, pair, palindrome, both ends, in-place

**In plain English:** One pointer READS every element. A second pointer marks where to WRITE the next element you want to keep.

**Steps:**
1. write = 1 (first element is always kept).
2. read goes through the rest.
3. If nums[read] differs from the last kept value → nums[write++] = nums[read].
4. Return write (the new length).

**Tiny example:**

```text
[1, 1, 2, 2, 3]
read 1 → skip; read 2 → write at 1; read 2 → skip; read 3 → write at 2
result [1, 2, 3], length 3
```

**Java:**

```java
int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int write = 1;
    for (int read = 1; read < nums.length; read++) {
        if (nums[read] != nums[write - 1]) {
            nums[write++] = nums[read];
        }
    }
    return write;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Return write count, not last index.

**Practise on:** Two Sum II · Valid Palindrome · Container With Most Water · Remove elements / duplicates

#### 5.4 Container / max area

- **Big pattern:** Two Pointers
- **Ask yourself:** *Which side limits the current answer?*
- **Small recipe:** `Move limiting side`
- **Memory sentence:** Two Pointers → Container / max area → Move limiting side
- **Words that give it away:** sorted, pair, palindrome, both ends, in-place

**In plain English:** Water level is limited by the SHORTER wall. Moving the taller wall can never help, so always move the shorter wall inward.

**Steps:**
1. left = 0, right = n − 1.
2. area = min(h[l], h[r]) × (r − l); keep the best.
3. Move the pointer at the shorter wall.

**Tiny example:**

```text
heights [1, 8, 6, 2, 5, 4, 8, 3, 7]
l=0 (1), r=8 (7) → area 8 → move l
l=1 (8), r=8 (7) → area 49 ← best
```

**Java:**

```java
int maxArea(int[] h) {
    int left = 0, right = h.length - 1, best = 0;
    while (left < right) {
        int area = Math.min(h[left], h[right]) * (right - left);
        best = Math.max(best, area);
        if (h[left] < h[right]) left++;
        else right--;
    }
    return best;
}
```

**Complexity:** time O(n) · space O(1)

**Practise on:** Two Sum II · Valid Palindrome · Container With Most Water

**Runnable in SkillForge (Two Pointers):** Reverse a String (Easy) · Valid Palindrome (Easy) · Move Zeroes (Easy) · Pair With Target Sum (Sorted) (Easy) · Container With Most Water (Medium) · Remove Duplicates from Sorted Array (Easy) · Merge Sorted Array (Easy) · Squares of a Sorted Array (Easy) · Sort Colors (Dutch National Flag) (Medium) · 3Sum (Medium) · Trapping Rain Water (Hard) · Rotate Array (Medium) · Find the Duplicate Number (Medium) · Happy Number (Easy)

---

### 6. Sliding Window (Fixed)

**The big idea:** The window always has exactly K items. Each step, one item enters on the right and one leaves on the left — update your answer with just those two changes.

**Think of it like this:** Like a train window of fixed width: as the train moves, one new tree appears on one side and one disappears on the other.

**How to spot it:**
- "Subarray / substring of length K"
- "Every window of size K"
- Average / sum / count over K consecutive items

**Template:**

```java
int sum = 0, best = Integer.MIN_VALUE;
for (int right = 0; right < n; right++) {
    sum += nums[right];                  // enter
    if (right >= k) sum -= nums[right - k];   // leave
    if (right >= k - 1) best = Math.max(best, sum);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Window sum / average](#61-window-sum--average) | Is the window size always exactly K? | `Add right, remove left` | O(n) |
| [Count condition in every K window](#62-count-condition-in-every-k-window) | Can I update the answer by only considering entering/leaving items? | `Running count` | O(n) |
| [Find anagrams](#63-find-anagrams) | Does every length-K window have the target counts? | `Window frequency map` | O(26 · n) = O(n) |
| [Window max/min](#64-window-maxmin) | Do I need the best value for every fixed window? | `Monotonic deque` | O(n) |

#### 6.1 Window sum / average

- **Big pattern:** Sliding Window (Fixed)
- **Ask yourself:** *Is the window size always exactly K?*
- **Small recipe:** `Add right, remove left`
- **Memory sentence:** Sliding Window Fixed → Fixed Window Sum → window size = k
- **Words that give it away:** size K, every window, fixed length, contiguous

**In plain English:** Add the number entering the window, subtract the one leaving. Only start reading answers once the window has K items.

**Steps:**
1. sum += nums[right].
2. If right ≥ K: sum −= nums[right − K].
3. If right ≥ K − 1: the window is full → update the answer.

**Tiny example:**

```text
[2, 1, 5, 1, 3, 2], K = 3
sums: 8, 7, 9, 6 → best 9
```

**Java:**

```java
double maxAverage(int[] nums, int k) {
    long sum = 0, best = Long.MIN_VALUE;
    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];
        if (right >= k) sum -= nums[right - k];
        if (right >= k - 1) best = Math.max(best, sum);
    }
    return (double) best / k;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Update answer only once the window reaches size k.

**Practise on:** Maximum Average Subarray I · Find All Anagrams in a String · Maximum sum of size K

#### 6.2 Count condition in every K window

- **Big pattern:** Sliding Window (Fixed)
- **Ask yourself:** *Can I update the answer by only considering entering/leaving items?*
- **Small recipe:** `Running count`
- **Memory sentence:** Sliding Window Fixed → Fixed Window Count → enter contributes, leave removes
- **Words that give it away:** size K, every window, fixed length, contiguous

**In plain English:** Keep a running count of items that match the rule. The entering item may add 1, the leaving item may remove 1.

**Steps:**
1. If the entering char matches → count++.
2. If the leaving char matched → count−−.
3. When the window is full, compare count with the best.

**Tiny example:**

```text
s = "abciiidef", K = 3, count vowels
windows: abc 1, bci 1, cii 2, iii 3 ← best
```

**Java:**

```java
int maxVowels(String s, int k) {
    int count = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        if (isVowel(s.charAt(right))) count++;
        if (right >= k && isVowel(s.charAt(right - k))) count--;
        if (right >= k - 1) best = Math.max(best, count);
    }
    return best;
}

boolean isVowel(char c) {
    return "aeiou".indexOf(c) >= 0;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Mirror every add with a matching remove.

**Practise on:** Maximum Average Subarray I · Find All Anagrams in a String · Count vowels in each window

#### 6.3 Find anagrams

- **Big pattern:** Sliding Window (Fixed)
- **Ask yourself:** *Does every length-K window have the target counts?*
- **Small recipe:** `Window frequency map`
- **Memory sentence:** Sliding Window Fixed → Window Frequency → add current char, remove char k steps behind
- **Words that give it away:** size K, every window, fixed length, contiguous

**In plain English:** Count the letters of the target word. Slide a window of the same length over the text, keeping its letter counts. Whenever the counts match, record the start index.

**Steps:**
1. need[26] = counts of p; have[26] = counts of the window.
2. Add the entering letter, remove the letter K steps behind.
3. If have equals need → window start is an answer.

**Tiny example:**

```text
s = "cbaebabacd", p = "abc"
window "cba" matches → 0
window "bac" matches → 6
```

**Java:**

```java
List<Integer> findAnagrams(String s, String p) {
    List<Integer> result = new ArrayList<>();
    int k = p.length();
    int[] need = new int[26], have = new int[26];
    for (char c : p.toCharArray()) need[c - 'a']++;
    for (int right = 0; right < s.length(); right++) {
        have[s.charAt(right) - 'a']++;
        if (right >= k) have[s.charAt(right - k) - 'a']--;
        if (right >= k - 1 && Arrays.equals(have, need)) {
            result.add(right - k + 1);
        }
    }
    return result;
}
```

**Complexity:** time O(26 · n) = O(n) · space O(1)

> ⚠️ **Trap:** Use array counts for fixed lowercase alphabet.

**Practise on:** Maximum Average Subarray I · Find All Anagrams in a String · Find Anagrams

#### 6.4 Window max/min

- **Big pattern:** Sliding Window (Fixed)
- **Ask yourself:** *Do I need the best value for every fixed window?*
- **Small recipe:** `Monotonic deque`
- **Memory sentence:** Monotonic Queue → Sliding Window Maximum → expire old → remove smaller back → add current
- **Words that give it away:** size K, every window, fixed length, contiguous

**In plain English:** For the max of every window, keep a deque of indexes whose values are decreasing. Smaller values behind a bigger new value can never win, so drop them. The front is always the window max.

**Steps:**
1. Drop the front if its index left the window.
2. Pop from the back while the back value ≤ the new value.
3. Push the new index; once the window is full, record nums[front].

**Tiny example:**

```text
[1, 3, −1, −3, 5, 3], K = 3
maxes: 3, 3, 5, 5
```

**Java:**

```java
int[] maxSlidingWindow(int[] nums, int k) {
    int[] result = new int[nums.length - k + 1];
    Deque<Integer> dq = new ArrayDeque<>();          // indexes, values decreasing
    for (int i = 0; i < nums.length; i++) {
        if (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();   // expired
        while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) dq.pollLast();
        dq.offerLast(i);
        if (i >= k - 1) result[i - k + 1] = nums[dq.peekFirst()];
    }
    return result;
}
```

**Complexity:** time O(n) · space O(k)

> ⚠️ **Trap:** Order of operations matters: expire, clean back, add, then read front.

**Practise on:** Maximum Average Subarray I · Find All Anagrams in a String · Sliding Window Maximum

**Runnable in SkillForge (Sliding Window (Fixed)):** Maximum Sum Subarray of Size K (Easy)

---

### 7. Sliding Window (Dynamic)

**The big idea:** The window grows on the right. When it breaks a rule, shrink it from the left until the rule holds again. The window size changes as you go.

**Think of it like this:** Like a caterpillar: the head crawls forward; when the body gets too stretched, the tail catches up.

**How to spot it:**
- Longest / shortest substring or subarray with a condition
- "At most K distinct"
- "No repeating characters"
- Sum ≥ target with positive numbers

**Template:**

```java
int left = 0, best = 0;
for (int right = 0; right < n; right++) {
    add(nums[right]);
    while (windowIsInvalid()) {
        remove(nums[left++]);
    }
    best = Math.max(best, right - left + 1);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Longest valid window](#71-longest-valid-window) | Can I grow until a rule breaks, then repair from the left? | `Expand right, shrink when invalid` | O(n) — each index enters and leaves once |
| [Smallest valid window](#72-smallest-valid-window) | Once valid, can I make it smaller? | `Expand until valid, shrink aggressively` | O(n) |
| [At most K distinct](#73-at-most-k-distinct) | Does validity depend on how many distinct values are inside? | `Frequency map + left pointer` | O(n) |
| [No repeats](#74-no-repeats) | When a duplicate appears, where should left jump? | `Set / last-seen map` | O(n) |

#### 7.1 Longest valid window

- **Big pattern:** Sliding Window (Dynamic)
- **Ask yourself:** *Can I grow until a rule breaks, then repair from the left?*
- **Small recipe:** `Expand right, shrink when invalid`
- **Memory sentence:** Sliding Window Dynamic → Longest Valid Window → expand right; while invalid shrink left
- **Words that give it away:** longest, shortest, substring, subarray, at most K, no repeats

**In plain English:** Grow the window one step at a time. If it becomes invalid, shrink from the left until it is valid. Then record its length — it is the longest valid window ending here.

**Steps:**
1. Add nums[right] to the window state.
2. while (invalid) remove nums[left++].
3. best = max(best, right − left + 1) — AFTER fixing validity.

**Tiny example:**

```text
Longest subarray of 1s after flipping at most one 0: [1,1,0,1,1,0,1]
window may hold at most one 0
best = 5 → [1,1,0,1,1]
```

**Java:**

```java
int longestOnes(int[] nums, int maxZeros) {
    int left = 0, zeros = 0, best = 0;
    for (int right = 0; right < nums.length; right++) {
        if (nums[right] == 0) zeros++;
        while (zeros > maxZeros) {            // invalid → shrink
            if (nums[left] == 0) zeros--;
            left++;
        }
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Complexity:** time O(n) — each index enters and leaves once · space O(1)

> ⚠️ **Trap:** For longest-valid problems, update after restoring validity.

**Practise on:** Longest Substring Without Repeating Characters · Minimum Size Subarray Sum · Longest substring / subarray under a condition

#### 7.2 Smallest valid window

- **Big pattern:** Sliding Window (Dynamic)
- **Ask yourself:** *Once valid, can I make it smaller?*
- **Small recipe:** `Expand until valid, shrink aggressively`
- **Memory sentence:** Sliding Window Dynamic → Smallest Valid Window → when valid → update → shrink
- **Words that give it away:** longest, shortest, substring, subarray, at most K, no repeats

**In plain English:** Grow until the window becomes valid. While it stays valid, record its length and shrink from the left to try an even smaller one.

**Steps:**
1. sum += nums[right].
2. while (sum ≥ target): best = min(best, length); sum −= nums[left++].
3. If best never changed → 0.

**Tiny example:**

```text
[2, 3, 1, 2, 4, 3], target 7
valid at [2,3,1,2] (4) → shrink…
best = 2 → [4, 3]
```

**Java:**

```java
int minSubArrayLen(int target, int[] nums) {
    int left = 0, sum = 0, best = Integer.MAX_VALUE;
    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];
        while (sum >= target) {               // valid → record, then shrink
            best = Math.min(best, right - left + 1);
            sum -= nums[left++];
        }
    }
    return best == Integer.MAX_VALUE ? 0 : best;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** For minimum-valid problems, update inside the shrinking loop.

**Practise on:** Longest Substring Without Repeating Characters · Minimum Size Subarray Sum

#### 7.3 At most K distinct

- **Big pattern:** Sliding Window (Dynamic)
- **Ask yourself:** *Does validity depend on how many distinct values are inside?*
- **Small recipe:** `Frequency map + left pointer`
- **Memory sentence:** Sliding Window Dynamic → At most K distinct → Frequency map + left pointer
- **Words that give it away:** longest, shortest, substring, subarray, at most K, no repeats

**In plain English:** Count characters inside the window with a map. When the map has more than K different keys, shrink from the left, removing keys whose count drops to 0.

**Steps:**
1. count[s[right]]++.
2. while (count.size() > K): decrement s[left]; remove it if 0; left++.
3. best = max(best, window length).

**Tiny example:**

```text
s = "eceba", K = 2
"ece" has {e, c} ✓ length 3
add b → 3 distinct → shrink to "eb"
```

**Java:**

```java
int longestWithAtMostKDistinct(String s, int k) {
    Map<Character, Integer> count = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        count.merge(s.charAt(right), 1, Integer::sum);
        while (count.size() > k) {
            char out = s.charAt(left++);
            if (count.merge(out, -1, Integer::sum) == 0) count.remove(out);
        }
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Complexity:** time O(n) · space O(k)

**Practise on:** Longest Substring Without Repeating Characters · Minimum Size Subarray Sum

#### 7.4 No repeats

- **Big pattern:** Sliding Window (Dynamic)
- **Ask yourself:** *When a duplicate appears, where should left jump?*
- **Small recipe:** `Set / last-seen map`
- **Memory sentence:** Sliding Window Dynamic → No Repeating Characters → duplicate → jump left
- **Words that give it away:** longest, shortest, substring, subarray, at most K, no repeats

**In plain English:** Remember the last index where each character appeared. When a character repeats inside the window, jump left to just after its previous position.

**Steps:**
1. If c was seen at index p and p ≥ left → left = p + 1.
2. lastSeen[c] = right.
3. best = max(best, right − left + 1).

**Tiny example:**

```text
abcabcbb
abc (3) → a repeats → left 1 → bca → b repeats → left 2 …
best = 3
```

**Java:**

```java
int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c)) {
            left = Math.max(left, lastSeen.get(c) + 1);   // never move left backwards
        }
        lastSeen.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Complexity:** time O(n) · space O(alphabet)

> ⚠️ **Trap:** Math.max prevents left from moving backward.

**Practise on:** Longest Substring Without Repeating Characters · Minimum Size Subarray Sum

**Runnable in SkillForge (Sliding Window (Dynamic)):** Longest Substring Without Repeating Characters (Medium) · Minimum Size Subarray Sum (Medium)

---

## Phase 2 — Stacks, Queues & Heaps

### 8. Stack

**The big idea:** A stack is a pile: the LAST thing you put on is the FIRST thing you take off (LIFO). Use it when the newest unfinished thing must be finished first.

**Think of it like this:** A stack of plates — you always take the plate on top.

**How to spot it:**
- Matching brackets / tags
- Nested structures (expressions, folders)
- Undo
- Replacing recursion with a loop

**Template:**

```java
Deque<Character> stack = new ArrayDeque<>();
stack.push(x);          // put on top
char top = stack.peek(); // look at top
stack.pop();            // remove top
stack.isEmpty();
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Balanced brackets](#81-balanced-brackets) | Does the newest unfinished thing need to finish first? | `Push opens, match closes` | O(n) |
| [Expression evaluation](#82-expression-evaluation) | Do nested operations resolve in LIFO order? | `Operand/operator stack` | O(n) |
| [Undo history](#83-undo-history) | Should the most recent action be undone first? | `Push actions` | O(length of the action) per operation |
| [Iterative DFS](#84-iterative-dfs) | Can I replace recursion with my own stack? | `Explicit stack` | O(V + E) |

#### 8.1 Balanced brackets

- **Big pattern:** Stack
- **Ask yourself:** *Does the newest unfinished thing need to finish first?*
- **Small recipe:** `Push opens, match closes`
- **Memory sentence:** Stack → Balanced Brackets → opening push, closing must match top
- **Words that give it away:** nested, parentheses, undo, LIFO

**In plain English:** Push every opening bracket. When a closing bracket arrives, the top of the stack must be its matching opener. At the end the stack must be empty.

**Steps:**
1. Opening → push.
2. Closing → stack empty or wrong top → invalid; else pop.
3. Finish: valid only if the stack is empty.

**Tiny example:**

```text
({[]})
push ( { [
] pops [, } pops {, ) pops (
empty → valid
```

**Java:**

```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(')');        // push the closer we expect
        else if (c == '[') stack.push(']');
        else if (c == '{') stack.push('}');
        else if (stack.isEmpty() || stack.pop() != c) return false;
    }
    return stack.isEmpty();
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Prefer ArrayDeque over legacy Stack in modern Java.

**Practise on:** Valid Parentheses · Evaluate Reverse Polish Notation

#### 8.2 Expression evaluation

- **Big pattern:** Stack
- **Ask yourself:** *Do nested operations resolve in LIFO order?*
- **Small recipe:** `Operand/operator stack`
- **Memory sentence:** Stack → Expression evaluation → Operand/operator stack
- **Words that give it away:** nested, parentheses, undo, LIFO

**In plain English:** In postfix (Reverse Polish) notation, numbers go on the stack. An operator pops the top two numbers, combines them, and pushes the result.

**Steps:**
1. Number → push.
2. Operator → b = pop, a = pop, push (a op b). Order matters for − and /.
3. The final stack top is the answer.

**Tiny example:**

```text
["2", "1", "+", "3", "*"]
push 2, 1 → "+" → 3 → push 3 → "*" → 9
```

**Java:**

```java
int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (String t : tokens) {
        switch (t) {
            case "+" -> stack.push(stack.pop() + stack.pop());
            case "*" -> stack.push(stack.pop() * stack.pop());
            case "-" -> { int b = stack.pop(), a = stack.pop(); stack.push(a - b); }
            case "/" -> { int b = stack.pop(), a = stack.pop(); stack.push(a / b); }
            default -> stack.push(Integer.parseInt(t));
        }
    }
    return stack.pop();
}
```

**Complexity:** time O(n) · space O(n)

**Practise on:** Valid Parentheses · Evaluate Reverse Polish Notation

#### 8.3 Undo history

- **Big pattern:** Stack
- **Ask yourself:** *Should the most recent action be undone first?*
- **Small recipe:** `Push actions`
- **Memory sentence:** Stack → Undo history → Push actions
- **Words that give it away:** nested, parentheses, undo, LIFO

**In plain English:** Every action is pushed. Undo pops the most recent action and reverses it. (A second stack gives you "redo".)

**Steps:**
1. do(action): apply it and push it; clear the redo stack.
2. undo(): pop from history, reverse it, push to redo.
3. redo(): pop from redo, apply again, push to history.

**Tiny example:**

```text
type "a", type "b", undo → "a", redo → "ab"
```

**Java:**

```java
class TextEditor {
    private final StringBuilder text = new StringBuilder();
    private final Deque<String> history = new ArrayDeque<>();
    private final Deque<String> redo = new ArrayDeque<>();

    void type(String s) {
        text.append(s);
        history.push(s);
        redo.clear();
    }

    void undo() {
        if (history.isEmpty()) return;
        String last = history.pop();
        text.setLength(text.length() - last.length());
        redo.push(last);
    }

    void redoLast() {
        if (redo.isEmpty()) return;
        String s = redo.pop();
        text.append(s);
        history.push(s);
    }

    String value() { return text.toString(); }
}
```

**Complexity:** time O(length of the action) per operation · space O(total actions)

**Practise on:** Valid Parentheses · Evaluate Reverse Polish Notation

#### 8.4 Iterative DFS

- **Big pattern:** Stack
- **Ask yourself:** *Can I replace recursion with my own stack?*
- **Small recipe:** `Explicit stack`
- **Memory sentence:** Stack → Iterative DFS Stack → push work → pop → process
- **Words that give it away:** nested, parentheses, undo, LIFO

**In plain English:** Recursion secretly uses a stack. You can use your own: push the start, then repeatedly pop a node, visit it, and push its unvisited neighbours.

**Steps:**
1. push(start); mark it visited.
2. while stack not empty: node = pop(), visit it.
3. push each unvisited neighbour and mark it.

**Tiny example:**

```text
graph 0-1, 0-2, 1-3
pop 0 → push 1, 2 → pop 2 → pop 1 → push 3 → pop 3
```

**Java:**

```java
List<Integer> dfsIterative(List<List<Integer>> graph, int start) {
    List<Integer> order = new ArrayList<>();
    boolean[] visited = new boolean[graph.size()];
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);
    visited[start] = true;
    while (!stack.isEmpty()) {
        int node = stack.pop();
        order.add(node);
        for (int next : graph.get(node)) {
            if (!visited[next]) {
                visited[next] = true;
                stack.push(next);
            }
        }
    }
    return order;
}
```

**Complexity:** time O(V + E) · space O(V)

> ⚠️ **Trap:** Mark visited consistently: on push or on pop, but understand duplicates.

**Practise on:** Valid Parentheses · Evaluate Reverse Polish Notation · DFS without recursion

**Runnable in SkillForge (Stack):** Valid Parentheses (Easy) · Min Stack (Medium)

---

### 9. Monotonic Stack

**The big idea:** A stack that stays sorted (always decreasing or always increasing). When a new value breaks the order, you pop — and every popped item has just found its answer.

**Think of it like this:** People in a queue looking right for someone taller: when a tall person arrives, everyone shorter in front of them has found their answer and leaves.

**How to spot it:**
- Next / previous greater or smaller element
- "How many days until warmer"
- Largest rectangle in a histogram
- A nested loop that scans right for the first bigger value

**Template:**

```java
Deque<Integer> stack = new ArrayDeque<>();   // indexes
for (int i = 0; i < n; i++) {
    while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
        int j = stack.pop();   // nums[i] is the next greater of nums[j]
    }
    stack.push(i);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Next greater element](#91-next-greater-element) | Who is the first bigger value to my right? | `Decreasing stack` | O(n) |
| [Next smaller element](#92-next-smaller-element) | Who is the first smaller value to my right? | `Increasing stack` | O(n) |
| [Previous greater/smaller](#93-previous-greatersmaller) | Who is the nearest valid value on my left? | `Clean stack then peek` | O(n) |
| [Daily Temperatures](#94-daily-temperatures) | When will a warmer day arrive? | `Decreasing stack of indices` | O(n) |
| [Largest Rectangle](#95-largest-rectangle) | When a bar drops, which taller rectangles are now complete? | `Increasing stack + widths` | O(n) |

#### 9.1 Next greater element

- **Big pattern:** Monotonic Stack
- **Ask yourself:** *Who is the first bigger value to my right?*
- **Small recipe:** `Decreasing stack`
- **Memory sentence:** Monotonic Stack → Next Greater Element → pop while current is greater
- **Words that give it away:** next greater, next smaller, previous greater, histogram

**In plain English:** Keep indexes whose next-greater is still unknown. When a bigger value arrives, pop them — that bigger value is their answer.

**Steps:**
1. answer[] filled with −1.
2. While top value < current: answer[pop] = current.
3. Push current index.

**Tiny example:**

```text
[2, 1, 5]
push 2, push 1
5 pops 1 → 5, pops 2 → 5
answer [5, 5, −1]
```

**Java:**

```java
int[] nextGreater(int[] nums) {
    int[] answer = new int[nums.length];
    Arrays.fill(answer, -1);
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < nums.length; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            answer[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return answer;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Store indices when you need positions or distances.

**Practise on:** Daily Temperatures · Next Greater Element · Largest Rectangle in Histogram

#### 9.2 Next smaller element

- **Big pattern:** Monotonic Stack
- **Ask yourself:** *Who is the first smaller value to my right?*
- **Small recipe:** `Increasing stack`
- **Memory sentence:** Monotonic Stack → Next Smaller Element → pop while current is smaller
- **Words that give it away:** next greater, next smaller, previous greater, histogram

**In plain English:** Mirror image of next greater: keep an increasing stack and pop while the current value is SMALLER than the top.

**Steps:**
1. answer[] = −1.
2. While top value > current: answer[pop] = current.
3. Push current index.

**Tiny example:**

```text
[4, 3, 5, 2]
3 pops 4 → 3
2 pops 5 → 2 and 3 → 2
answer [3, 2, 2, −1]
```

**Java:**

```java
int[] nextSmaller(int[] nums) {
    int[] answer = new int[nums.length];
    Arrays.fill(answer, -1);
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < nums.length; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] > nums[i]) {
            answer[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return answer;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Decide carefully whether equality should pop: < versus <= changes behavior.

**Practise on:** Daily Temperatures · Next Greater Element · Largest Rectangle in Histogram · Next Smaller Element

#### 9.3 Previous greater/smaller

- **Big pattern:** Monotonic Stack
- **Ask yourself:** *Who is the nearest valid value on my left?*
- **Small recipe:** `Clean stack then peek`
- **Memory sentence:** Monotonic Stack → Previous greater/smaller → Clean stack then peek
- **Words that give it away:** next greater, next smaller, previous greater, histogram

**In plain English:** To find the nearest greater value on the LEFT: before pushing the current value, pop everything that is not greater. Whatever remains on top is the answer.

**Steps:**
1. While top ≤ current: pop (it can never be anyone’s previous-greater again).
2. answer[i] = top, or −1 if empty.
3. Push current.

**Tiny example:**

```text
[3, 1, 4, 2]
3 → −1; 1 → 3; 4 pops 1, 3 → −1; 2 → 4
answer [−1, 3, −1, 4]
```

**Java:**

```java
int[] previousGreater(int[] nums) {
    int[] answer = new int[nums.length];
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < nums.length; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] <= nums[i]) {
            stack.pop();
        }
        answer[i] = stack.isEmpty() ? -1 : nums[stack.peek()];
        stack.push(i);
    }
    return answer;
}
```

**Complexity:** time O(n) · space O(n)

**Practise on:** Daily Temperatures · Next Greater Element · Largest Rectangle in Histogram

#### 9.4 Daily Temperatures

- **Big pattern:** Monotonic Stack
- **Ask yourself:** *When will a warmer day arrive?*
- **Small recipe:** `Decreasing stack of indices`
- **Memory sentence:** Monotonic Stack → Daily Temperatures → warmer current day resolves old days
- **Words that give it away:** next greater, next smaller, previous greater, histogram

**In plain English:** Like next greater, but the answer is a DISTANCE in days — so store indexes and compute i − popped index.

**Steps:**
1. Stack of day indexes with decreasing temperatures.
2. A warmer day pops colder days: wait = i − j.
3. Days left in the stack wait 0.

**Tiny example:**

```text
[73, 74, 75, 71, 69, 72, 76, 73]
74 resolves 73 → 1 day; 72 resolves 69 → 1, 71 → 2 …
answer [1, 1, 4, 2, 1, 1, 0, 0]
```

**Java:**

```java
int[] dailyTemperatures(int[] temps) {
    int[] wait = new int[temps.length];
    Deque<Integer> stack = new ArrayDeque<>();
    for (int i = 0; i < temps.length; i++) {
        while (!stack.isEmpty() && temps[stack.peek()] < temps[i]) {
            int j = stack.pop();
            wait[j] = i - j;
        }
        stack.push(i);
    }
    return wait;
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** Distance problems almost always require indices.

**Practise on:** Daily Temperatures · Next Greater Element · Largest Rectangle in Histogram

#### 9.5 Largest Rectangle

- **Big pattern:** Monotonic Stack
- **Ask yourself:** *When a bar drops, which taller rectangles are now complete?*
- **Small recipe:** `Increasing stack + widths`
- **Memory sentence:** Monotonic Stack → Largest Rectangle → Increasing stack + widths
- **Words that give it away:** next greater, next smaller, previous greater, histogram

**In plain English:** Keep bars in increasing height. When a shorter bar arrives, the taller bars on the stack cannot extend further right — pop each and compute its rectangle: height × width between its neighbours.

**Steps:**
1. Loop i from 0 to n (treat i = n as height 0 to flush the stack).
2. While current < top height: pop; width = i − (new top) − 1.
3. area = height × width; keep the best.

**Tiny example:**

```text
[2, 1, 5, 6, 2, 3]
bar 2 at i=4 pops 6 (6×1) and 5 (5×2 = 10)
best = 10
```

**Java:**

```java
int largestRectangleArea(int[] h) {
    Deque<Integer> stack = new ArrayDeque<>();
    int best = 0;
    for (int i = 0; i <= h.length; i++) {
        int cur = (i == h.length) ? 0 : h[i];
        while (!stack.isEmpty() && h[stack.peek()] > cur) {
            int height = h[stack.pop()];
            int left = stack.isEmpty() ? -1 : stack.peek();
            best = Math.max(best, height * (i - left - 1));
        }
        stack.push(i);
    }
    return best;
}
```

**Complexity:** time O(n) · space O(n)

**Practise on:** Daily Temperatures · Next Greater Element · Largest Rectangle in Histogram

**Runnable in SkillForge (Monotonic Stack):** Next Greater Element (Easy) · Daily Temperatures (Medium) · Largest Rectangle in Histogram (Hard)

---

### 10. Queue

**The big idea:** A queue is a line: the FIRST one in is the FIRST one out (FIFO). Use it to process things in the order they arrived — and for exploring level by level (BFS).

**Think of it like this:** A ticket counter line — the person who came first is served first.

**How to spot it:**
- Process in arrival order
- Level-by-level tree traversal
- Shortest path when every step costs the same
- Task buffers

**Template:**

```java
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(x);        // join the rear
int first = queue.poll(); // leave from the front (null if empty)
queue.peek();          // look at the front
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [FIFO processing](#101-fifo-processing) | Should the oldest waiting item go first? | `offer → poll` | O(1) per offer / poll |
| [Tree level order](#102-tree-level-order) | Do I need to process all nodes at one depth together? | `Queue by level` | O(n) |
| [BFS frontier](#103-bfs-frontier) | Do I explore distance 1 before distance 2? | `Queue` | O(V + E) |
| [Work buffer](#104-work-buffer) | Should new work wait behind older work? | `Producer/consumer queue` | O(1) per operation |

#### 10.1 FIFO processing

- **Big pattern:** Queue
- **Ask yourself:** *Should the oldest waiting item go first?*
- **Small recipe:** `offer → poll`
- **Memory sentence:** Queue → Basic FIFO → offer rear, poll front
- **Words that give it away:** FIFO, level order, BFS, first-in-first-out

**In plain English:** Add new items at the back, always take the next item from the front. That keeps everything in arrival order.

**Steps:**
1. offer(item) to add.
2. poll() to take the oldest.
3. Use peek() / isEmpty() to check without removing.

**Tiny example:**

```text
offer 1, 2, 3 → poll → 1 → offer 4 → queue [2, 3, 4]
```

**Java:**

```java
List<String> serveInOrder(List<String> customers) {
    Deque<String> line = new ArrayDeque<>(customers);
    List<String> served = new ArrayList<>();
    while (!line.isEmpty()) {
        served.add(line.poll());    // oldest first
    }
    return served;
}
```

**Complexity:** time O(1) per offer / poll · space O(n)

> ⚠️ **Trap:** offer/poll/peek are safer than add/remove/element for empty queues.

**Practise on:** Binary Tree Level Order Traversal · Rotting Oranges · Queue basics

#### 10.2 Tree level order

- **Big pattern:** Queue
- **Ask yourself:** *Do I need to process all nodes at one depth together?*
- **Small recipe:** `Queue by level`
- **Memory sentence:** Queue → Level Order BFS → capture queue.size() for current level
- **Words that give it away:** FIFO, level order, BFS, first-in-first-out

**In plain English:** Put the root in a queue. At the start of each level, read queue.size() — that many nodes form the current level. Process them and add their children.

**Steps:**
1. queue = [root].
2. size = queue.size(); poll exactly size nodes → one level.
3. Offer each node’s children for the next level.

**Tiny example:**

```text
tree 3 / (9, 20) / (15, 7)
level 1: [3]
level 2: [9, 20]
level 3: [15, 7]
```

**Java:**

```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> levels = new ArrayList<>();
    if (root == null) return levels;
    Deque<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();               // capture BEFORE adding children
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        levels.add(level);
    }
    return levels;
}
```

**Complexity:** time O(n) · space O(width of the tree)

> ⚠️ **Trap:** Capture size before processing the level.

**Practise on:** Binary Tree Level Order Traversal · Rotting Oranges

#### 10.3 BFS frontier

- **Big pattern:** Queue
- **Ask yourself:** *Do I explore distance 1 before distance 2?*
- **Small recipe:** `Queue`
- **Memory sentence:** Queue → BFS frontier → Queue
- **Words that give it away:** FIFO, level order, BFS, first-in-first-out

**In plain English:** The queue holds the "frontier": everything at distance d is processed before anything at distance d + 1. So the first time you reach a node is the shortest way there.

**Steps:**
1. dist[start] = 0, queue = [start].
2. Poll a node; for each unvisited neighbour: dist = dist[node] + 1, offer it.
3. Mark visited when you ENQUEUE.

**Tiny example:**

```text
0 → {1, 2}, 1 → {3}
dist: 0:0, 1:1, 2:1, 3:2
```

**Java:**

```java
int[] bfsDistances(List<List<Integer>> graph, int start) {
    int[] dist = new int[graph.size()];
    Arrays.fill(dist, -1);
    Deque<Integer> queue = new ArrayDeque<>();
    dist[start] = 0;
    queue.offer(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        for (int next : graph.get(node)) {
            if (dist[next] == -1) {
                dist[next] = dist[node] + 1;
                queue.offer(next);
            }
        }
    }
    return dist;
}
```

**Complexity:** time O(V + E) · space O(V)

**Practise on:** Binary Tree Level Order Traversal · Rotting Oranges

#### 10.4 Work buffer

- **Big pattern:** Queue
- **Ask yourself:** *Should new work wait behind older work?*
- **Small recipe:** `Producer/consumer queue`
- **Memory sentence:** Queue → Work buffer → Producer/consumer queue
- **Words that give it away:** FIFO, level order, BFS, first-in-first-out

**In plain English:** A queue can sit between a producer (who adds jobs) and a consumer (who handles them), so bursts of work wait politely in order.

**Steps:**
1. Producer: queue.offer(job).
2. Consumer: job = queue.poll(); handle it.
3. Optionally cap the size to apply back-pressure.

**Tiny example:**

```text
jobs arrive A, B, C quickly; worker handles A, then B, then C
```

**Java:**

```java
class JobBuffer {
    private final Deque<String> jobs = new ArrayDeque<>();
    private final int capacity;

    JobBuffer(int capacity) { this.capacity = capacity; }

    boolean submit(String job) {
        if (jobs.size() == capacity) return false;   // full → reject / retry later
        jobs.offer(job);
        return true;
    }

    String next() {
        return jobs.poll();                          // null when there is no work
    }
}
```

**Complexity:** time O(1) per operation · space O(capacity)

**Practise on:** Binary Tree Level Order Traversal · Rotting Oranges

**Runnable in SkillForge (Queue):** Find the Winner of the Circular Game (Medium)

---

### 11. Monotonic Queue

**The big idea:** A deque (double-ended queue) that keeps only useful candidates in sorted order. The best candidate is always at the front; outdated items leave from the front, useless ones from the back.

**Think of it like this:** A leaderboard for the last K minutes: when a stronger player arrives, weaker older players can never lead again, so they are removed.

**How to spot it:**
- Sliding window maximum / minimum
- Max or min over a moving range inside a DP

**Template:**

```java
Deque<Integer> dq = new ArrayDeque<>();   // indexes
// 1. expire: if dq.peekFirst() <= i - k → pollFirst()
// 2. clean:  while back value <= nums[i] → pollLast()
// 3. add:    offerLast(i)
// 4. read:   nums[dq.peekFirst()] is the window max
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Sliding window maximum](#111-sliding-window-maximum) | Can I keep only candidates that could still become maximum? | `Decreasing deque` | O(n) |
| [Sliding window minimum](#112-sliding-window-minimum) | Can I keep only candidates that could still become minimum? | `Increasing deque` | O(n) |
| [Expire old values](#113-expire-old-values) | How do I know when a candidate leaves the window? | `Store indices` | O(1) amortised |
| [DP window optimization](#114-dp-window-optimization) | Am I repeatedly taking max/min over a moving DP range? | `Deque of best states` | O(n) instead of O(n·K) |

#### 11.1 Sliding window maximum

- **Big pattern:** Monotonic Queue
- **Ask yourself:** *Can I keep only candidates that could still become maximum?*
- **Small recipe:** `Decreasing deque`
- **Memory sentence:** Monotonic Queue → Sliding Window Maximum → expire old → remove smaller back → add current
- **Words that give it away:** window maximum, window minimum, deque, moving range

**In plain English:** Keep indexes with decreasing values. Remove expired ones from the front, remove smaller ones from the back, then the front is the maximum.

**Steps:**
1. Expire the front if it is outside the window.
2. Pop the back while its value ≤ the new value.
3. Push the new index; read the front when the window is full.

**Tiny example:**

```text
[1, 3, −1, −3, 5, 3, 6, 7], K = 3
maxes: 3, 3, 5, 5, 6, 7
```

**Java:**

```java
int[] maxSlidingWindow(int[] nums, int k) {
    int[] out = new int[nums.length - k + 1];
    Deque<Integer> dq = new ArrayDeque<>();
    for (int i = 0; i < nums.length; i++) {
        if (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();
        while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) dq.pollLast();
        dq.offerLast(i);
        if (i >= k - 1) out[i - k + 1] = nums[dq.peekFirst()];
    }
    return out;
}
```

**Complexity:** time O(n) · space O(k)

> ⚠️ **Trap:** Order of operations matters: expire, clean back, add, then read front.

**Practise on:** Sliding Window Maximum · Shortest Subarray with Sum at Least K

#### 11.2 Sliding window minimum

- **Big pattern:** Monotonic Queue
- **Ask yourself:** *Can I keep only candidates that could still become minimum?*
- **Small recipe:** `Increasing deque`
- **Memory sentence:** Monotonic Queue → Sliding Window Minimum → expire old → remove larger back
- **Words that give it away:** window maximum, window minimum, deque, moving range

**In plain English:** Same as maximum, but keep INCREASING values: pop the back while it is ≥ the new value.

**Steps:**
1. Expire the front.
2. Pop the back while back value ≥ new value.
3. Push; front = window minimum.

**Tiny example:**

```text
[4, 2, 12, 3, 8], K = 2
mins: 2, 2, 3, 3
```

**Java:**

```java
int[] minSlidingWindow(int[] nums, int k) {
    int[] out = new int[nums.length - k + 1];
    Deque<Integer> dq = new ArrayDeque<>();
    for (int i = 0; i < nums.length; i++) {
        if (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();
        while (!dq.isEmpty() && nums[dq.peekLast()] >= nums[i]) dq.pollLast();
        dq.offerLast(i);
        if (i >= k - 1) out[i - k + 1] = nums[dq.peekFirst()];
    }
    return out;
}
```

**Complexity:** time O(n) · space O(k)

> ⚠️ **Trap:** Flip the comparison from maximum version.

**Practise on:** Sliding Window Maximum · Shortest Subarray with Sum at Least K · Sliding Window Minimum

#### 11.3 Expire old values

- **Big pattern:** Monotonic Queue
- **Ask yourself:** *How do I know when a candidate leaves the window?*
- **Small recipe:** `Store indices`
- **Memory sentence:** Monotonic Queue → Expire old values → Store indices
- **Words that give it away:** window maximum, window minimum, deque, moving range

**In plain English:** Store INDEXES, not values. Then you can tell when the front item has slid out of the window: its index is ≤ i − K.

**Steps:**
1. Deque holds indexes.
2. Before using the front, check dq.peekFirst() ≤ i − K → pollFirst().
3. Values are read as nums[index].

**Tiny example:**

```text
K = 3, at i = 5 the window is [3..5]
front index 2 ≤ 5 − 3 → expired, remove it
```

**Java:**

```java
void expireOld(Deque<Integer> dq, int i, int k) {
    // the window is (i - k, i], so any index <= i - k has left it
    while (!dq.isEmpty() && dq.peekFirst() <= i - k) {
        dq.pollFirst();
    }
}
```

**Complexity:** time O(1) amortised · space O(1)

**Practise on:** Sliding Window Maximum · Shortest Subarray with Sum at Least K

#### 11.4 DP window optimization

- **Big pattern:** Monotonic Queue
- **Ask yourself:** *Am I repeatedly taking max/min over a moving DP range?*
- **Small recipe:** `Deque of best states`
- **Memory sentence:** Monotonic Queue → DP window optimization → Deque of best states
- **Words that give it away:** window maximum, window minimum, deque, moving range

**In plain English:** Some DP formulas say "dp[i] = nums[i] + best dp in the last K positions". Instead of looping over K positions each time, keep a monotonic deque of dp indexes so the best is always at the front.

**Steps:**
1. dp[i] = nums[i] + max(0, dp[front]).
2. Expire indexes older than i − K.
3. Pop smaller dp values from the back, push i.

**Tiny example:**

```text
Constrained Subsequence Sum: [10, 2, −10, 5, 20], K = 2
dp: 10, 12, 2, 17, 37 → answer 37
```

**Java:**

```java
int constrainedSubsetSum(int[] nums, int k) {
    int[] dp = new int[nums.length];
    Deque<Integer> dq = new ArrayDeque<>();      // indexes, dp values decreasing
    int best = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        if (!dq.isEmpty() && dq.peekFirst() < i - k) dq.pollFirst();
        dp[i] = nums[i] + (dq.isEmpty() ? 0 : Math.max(0, dp[dq.peekFirst()]));
        while (!dq.isEmpty() && dp[dq.peekLast()] <= dp[i]) dq.pollLast();
        dq.offerLast(i);
        best = Math.max(best, dp[i]);
    }
    return best;
}
```

**Complexity:** time O(n) instead of O(n·K) · space O(n)

**Practise on:** Sliding Window Maximum · Shortest Subarray with Sum at Least K

---

### 12. Heap (Priority Queue)

**The big idea:** A heap always gives you the smallest (min-heap) or largest (max-heap) item in O(log n), even while you keep adding new items.

**Think of it like this:** A hospital emergency room: the most urgent patient is always seen next, no matter when they arrived.

**How to spot it:**
- "Repeatedly take the smallest / largest"
- K-th largest / smallest
- Merge K sorted lists
- Scheduling by earliest end time

**Template:**

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
minHeap.offer(x);   // O(log n)
minHeap.peek();     // smallest, O(1)
minHeap.poll();     // remove smallest, O(log n)
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Repeated smallest](#121-repeated-smallest) | Do I repeatedly need the smallest/earliest item? | `Min Heap` | O(n log n) |
| [Repeated largest](#122-repeated-largest) | Do I repeatedly need the largest item? | `Max Heap` | O(n log n) |
| [Kth largest](#123-kth-largest) | Can I keep only the K largest seen so far? | `Min heap of size K` | O(n log k) |
| [Kth smallest](#124-kth-smallest) | Can I keep only the K smallest seen so far? | `Max heap of size K` | O(n log k) |
| [Merge K sorted streams](#125-merge-k-sorted-streams) | Which stream currently has the smallest next item? | `Min heap of heads` | O(N log k) |
| [Scheduling by end time](#126-scheduling-by-end-time) | Which occupied resource becomes free first? | `Min heap of end times` | O(n log n) |

#### 12.1 Repeated smallest

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Do I repeatedly need the smallest/earliest item?*
- **Small recipe:** `Min Heap`
- **Memory sentence:** Heap → Min Heap → smallest at root
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Java’s PriorityQueue is a min-heap by default: poll() always gives the smallest item left.

**Steps:**
1. offer every item.
2. poll() whenever you need the current smallest.
3. peek() to look without removing.

**Tiny example:**

```text
Connect ropes [4, 3, 2, 6] with min cost
take 2+3=5, then 4+5=9, then 6+9=15 → cost 29
```

**Java:**

```java
int connectRopes(int[] ropes) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();
    for (int r : ropes) heap.offer(r);
    int cost = 0;
    while (heap.size() > 1) {
        int a = heap.poll(), b = heap.poll();   // two smallest
        cost += a + b;
        heap.offer(a + b);
    }
    return cost;
}
```

**Complexity:** time O(n log n) · space O(n)

> ⚠️ **Trap:** Java PriorityQueue is a min heap by default.

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists · Repeated smallest / earliest

#### 12.2 Repeated largest

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Do I repeatedly need the largest item?*
- **Small recipe:** `Max Heap`
- **Memory sentence:** Heap → Max Heap → largest at root
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Pass Collections.reverseOrder() (or a comparator) to make a max-heap: poll() gives the largest item.

**Steps:**
1. new PriorityQueue<>(Collections.reverseOrder()).
2. offer items.
3. poll() for the largest.

**Tiny example:**

```text
Last Stone Weight [2, 7, 4, 1, 8, 1]
smash 8 & 7 → 1, then 4 & 2 → 2 … → 1 left
```

**Java:**

```java
int lastStoneWeight(int[] stones) {
    PriorityQueue<Integer> heap = new PriorityQueue<>(Collections.reverseOrder());
    for (int s : stones) heap.offer(s);
    while (heap.size() > 1) {
        int y = heap.poll(), x = heap.poll();   // two largest
        if (y != x) heap.offer(y - x);
    }
    return heap.isEmpty() ? 0 : heap.peek();
}
```

**Complexity:** time O(n log n) · space O(n)

> ⚠️ **Trap:** For custom objects, use a Comparator.

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists · Repeated largest

#### 12.3 Kth largest

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Can I keep only the K largest seen so far?*
- **Small recipe:** `Min heap of size K`
- **Memory sentence:** Heap → Kth Largest → keep only k largest
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Keep a MIN-heap holding only the K largest values seen so far. If it grows past K, remove the smallest. The root is then the K-th largest.

**Steps:**
1. offer each value.
2. if size > K → poll().
3. Answer = peek().

**Tiny example:**

```text
[3, 2, 1, 5, 6, 4], K = 2
heap keeps {5, 6}
answer 5
```

**Java:**

```java
int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();
    for (int x : nums) {
        heap.offer(x);
        if (heap.size() > k) heap.poll();
    }
    return heap.peek();
}
```

**Complexity:** time O(n log k) · space O(k)

> ⚠️ **Trap:** Min heap of size K: root is the smallest among the K largest.

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists

#### 12.4 Kth smallest

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Can I keep only the K smallest seen so far?*
- **Small recipe:** `Max heap of size K`
- **Memory sentence:** Heap → Kth smallest → Max heap of size K
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Mirror of K-th largest: keep a MAX-heap of the K smallest values. When it grows past K, drop the biggest.

**Steps:**
1. Max-heap.
2. offer; if size > K → poll().
3. Answer = peek().

**Tiny example:**

```text
[7, 10, 4, 3, 20, 15], K = 3
heap keeps {3, 4, 7}
answer 7
```

**Java:**

```java
int kthSmallest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>(Collections.reverseOrder());
    for (int x : nums) {
        heap.offer(x);
        if (heap.size() > k) heap.poll();
    }
    return heap.peek();
}
```

**Complexity:** time O(n log k) · space O(k)

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists

#### 12.5 Merge K sorted streams

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Which stream currently has the smallest next item?*
- **Small recipe:** `Min heap of heads`
- **Memory sentence:** Heap → Merge K sorted streams → Min heap of heads
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Put the first element of every list into a min-heap. Repeatedly take the smallest, and push the next element from the same list.

**Steps:**
1. Heap of [value, listIndex, position].
2. poll the smallest → add to the result.
3. If that list has more, offer its next element.

**Tiny example:**

```text
[1,4,5], [1,3,4], [2,6]
heap {1,1,2} → take 1 … result 1,1,2,3,4,4,5,6
```

**Java:**

```java
List<Integer> mergeKSorted(List<List<Integer>> lists) {
    PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));
    for (int i = 0; i < lists.size(); i++) {
        if (!lists.get(i).isEmpty()) heap.offer(new int[]{lists.get(i).get(0), i, 0});
    }
    List<Integer> merged = new ArrayList<>();
    while (!heap.isEmpty()) {
        int[] top = heap.poll();                  // {value, list, position}
        merged.add(top[0]);
        int next = top[2] + 1;
        if (next < lists.get(top[1]).size()) {
            heap.offer(new int[]{lists.get(top[1]).get(next), top[1], next});
        }
    }
    return merged;
}
```

**Complexity:** time O(N log k) · space O(k)

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists

#### 12.6 Scheduling by end time

- **Big pattern:** Heap (Priority Queue)
- **Ask yourself:** *Which occupied resource becomes free first?*
- **Small recipe:** `Min heap of end times`
- **Memory sentence:** Heap → Custom Object Heap → order by field
- **Words that give it away:** smallest repeatedly, largest repeatedly, earliest, priority

**In plain English:** Sort meetings by start. Keep a min-heap of the END times of rooms in use. If the earliest-ending room is free before the next meeting starts, reuse it.

**Steps:**
1. Sort by start time.
2. If heap.peek() ≤ start → poll (reuse that room).
3. offer the new end. Answer = heap size.

**Tiny example:**

```text
[[0,30], [5,10], [15,20]]
[0,30] → 1 room; [5,10] → 2 rooms; [15,20] reuses the room that ended at 10 → 2
```

**Java:**

```java
int minMeetingRooms(int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> Integer.compare(a[0], b[0]));
    PriorityQueue<Integer> ends = new PriorityQueue<>();
    for (int[] m : meetings) {
        if (!ends.isEmpty() && ends.peek() <= m[0]) {
            ends.poll();                     // that room is free again
        }
        ends.offer(m[1]);
    }
    return ends.size();
}
```

**Complexity:** time O(n log n) · space O(n)

> ⚠️ **Trap:** Avoid subtraction comparators like a[1]-b[1] because of overflow.

**Practise on:** Kth Largest Element · Meeting Rooms II · Merge K Sorted Lists · Dijkstra / scheduling

**Runnable in SkillForge (Heap (Priority Queue)):** K-th Largest Element (Medium)

---

### 13. Top K

**The big idea:** To keep the K best items, you do not need to sort everything. A heap of size K keeps the current winners and throws the rest away.

**Think of it like this:** A talent show with K chairs: a new performer takes a chair only if they beat the weakest person currently seated.

**How to spot it:**
- "Top K" / "K largest" / "K most frequent"
- "K closest"
- Data that streams in and never ends

**Template:**

```java
PriorityQueue<Integer> heap = new PriorityQueue<>();   // min-heap for "K largest"
for (int x : nums) {
    heap.offer(x);
    if (heap.size() > k) heap.poll();   // drop the weakest winner
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Top K largest](#131-top-k-largest) | Can I throw away everything outside the current top K? | `Min heap size K` | O(n log k) |
| [Top K smallest](#132-top-k-smallest) | Can I throw away everything larger than current top K smallest? | `Max heap size K` | O(n log k) |
| [Top K frequent](#133-top-k-frequent) | Can I count first, then rank by frequency? | `Frequency + heap/bucket` | O(n log k) |
| [K closest points](#134-k-closest-points) | Can I keep only K smallest distances? | `Max heap size K` | O(n log k) |
| [Streaming Top K](#135-streaming-top-k) | Can I maintain top K without storing the whole stream? | `Bounded heap` | O(log k) per add |

#### 13.1 Top K largest

- **Big pattern:** Top K
- **Ask yourself:** *Can I throw away everything outside the current top K?*
- **Small recipe:** `Min heap size K`
- **Memory sentence:** Top K → Top K Largest → keep k winners
- **Words that give it away:** top K, kth largest, kth smallest, closest K

**In plain English:** A min-heap of size K: the weakest of your K winners sits on top, so it is the one kicked out when a better item arrives.

**Steps:**
1. offer x.
2. if size > K → poll().
3. The heap now holds the K largest.

**Tiny example:**

```text
[5, 1, 9, 3, 7], K = 3 → {5, 7, 9}
```

**Java:**

```java
List<Integer> topKLargest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();
    for (int x : nums) {
        heap.offer(x);
        if (heap.size() > k) heap.poll();
    }
    return new ArrayList<>(heap);
}
```

**Complexity:** time O(n log k) · space O(k)

> ⚠️ **Trap:** Bounded heap prevents storing everything.

**Practise on:** Top K Frequent Elements · K Closest Points to Origin · Top K largest values

#### 13.2 Top K smallest

- **Big pattern:** Top K
- **Ask yourself:** *Can I throw away everything larger than current top K smallest?*
- **Small recipe:** `Max heap size K`
- **Memory sentence:** Top K → Top K smallest → Max heap size K
- **Words that give it away:** top K, kth largest, kth smallest, closest K

**In plain English:** Flip it: a MAX-heap of size K holds the K smallest; the biggest of them is on top and gets removed first.

**Steps:**
1. Max-heap.
2. offer; if size > K → poll().
3. The heap holds the K smallest.

**Tiny example:**

```text
[5, 1, 9, 3, 7], K = 2 → {1, 3}
```

**Java:**

```java
List<Integer> topKSmallest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>(Collections.reverseOrder());
    for (int x : nums) {
        heap.offer(x);
        if (heap.size() > k) heap.poll();
    }
    return new ArrayList<>(heap);
}
```

**Complexity:** time O(n log k) · space O(k)

**Practise on:** Top K Frequent Elements · K Closest Points to Origin

#### 13.3 Top K frequent

- **Big pattern:** Top K
- **Ask yourself:** *Can I count first, then rank by frequency?*
- **Small recipe:** `Frequency + heap/bucket`
- **Memory sentence:** Top K → Top K Frequent → count first → rank counts
- **Words that give it away:** top K, kth largest, kth smallest, closest K

**In plain English:** Two steps: count frequencies, then keep the K most frequent using a min-heap ordered by COUNT (or bucket sort by count for O(n)).

**Steps:**
1. count with a HashMap.
2. Min-heap of keys ordered by count; keep size ≤ K.
3. The heap holds the K most frequent keys.

**Tiny example:**

```text
[1, 1, 1, 2, 2, 3], K = 2
counts {1:3, 2:2, 3:1}
answer {1, 2}
```

**Java:**

```java
List<Integer> topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int x : nums) count.merge(x, 1, Integer::sum);
    PriorityQueue<Integer> heap = new PriorityQueue<>((a, b) -> Integer.compare(count.get(a), count.get(b)));
    for (int key : count.keySet()) {
        heap.offer(key);
        if (heap.size() > k) heap.poll();     // drop the least frequent
    }
    return new ArrayList<>(heap);
}
```

**Complexity:** time O(n log k) · space O(n)

> ⚠️ **Trap:** Heap compares by frequency, not by value.

**Practise on:** Top K Frequent Elements · K Closest Points to Origin

#### 13.4 K closest points

- **Big pattern:** Top K
- **Ask yourself:** *Can I keep only K smallest distances?*
- **Small recipe:** `Max heap size K`
- **Memory sentence:** Top K → K closest points → Max heap size K
- **Words that give it away:** top K, kth largest, kth smallest, closest K

**In plain English:** Distance² = x² + y² (no square root needed). Keep a MAX-heap by distance of size K — the farthest of the kept points leaves first.

**Steps:**
1. Max-heap ordered by distance².
2. offer each point; if size > K → poll().
3. Remaining K points are the closest.

**Tiny example:**

```text
points [[1,3], [−2,2]], K = 1
dist² 10 and 8 → keep [−2, 2]
```

**Java:**

```java
int[][] kClosest(int[][] points, int k) {
    PriorityQueue<int[]> heap = new PriorityQueue<>(
        (a, b) -> Long.compare(dist(b), dist(a)));   // farthest on top
    for (int[] p : points) {
        heap.offer(p);
        if (heap.size() > k) heap.poll();
    }
    return heap.toArray(new int[0][]);
}

long dist(int[] p) {
    return (long) p[0] * p[0] + (long) p[1] * p[1];
}
```

**Complexity:** time O(n log k) · space O(k)

**Practise on:** Top K Frequent Elements · K Closest Points to Origin

#### 13.5 Streaming Top K

- **Big pattern:** Top K
- **Ask yourself:** *Can I maintain top K without storing the whole stream?*
- **Small recipe:** `Bounded heap`
- **Memory sentence:** Top K → Streaming Top K → Bounded heap
- **Words that give it away:** top K, kth largest, kth smallest, closest K

**In plain English:** When numbers arrive forever, you cannot store them all. A size-K min-heap answers "what is the K-th largest so far?" after every new number.

**Steps:**
1. Constructor: add the initial numbers.
2. add(x): offer x; if size > K poll().
3. Return peek() — the K-th largest.

**Tiny example:**

```text
K = 3, stream 4, 5, 8, 2 → 4; add 3 → 4; add 5 → 5; add 10 → 5
```

**Java:**

```java
class KthLargest {
    private final PriorityQueue<Integer> heap = new PriorityQueue<>();
    private final int k;

    KthLargest(int k, int[] nums) {
        this.k = k;
        for (int x : nums) add(x);
    }

    int add(int x) {
        heap.offer(x);
        if (heap.size() > k) heap.poll();
        return heap.peek();
    }
}
```

**Complexity:** time O(log k) per add · space O(k)

**Practise on:** Top K Frequent Elements · K Closest Points to Origin

**Runnable in SkillForge (Top K):** Top K Frequent Elements (Medium)

---

## Phase 3 — Sorting, Searching & Greedy

### 14. Intervals

**The big idea:** An interval is a [start, end] range. Almost every interval problem starts the same way: SORT the intervals (usually by start), then walk through them once comparing neighbours.

**Think of it like this:** Like booking meeting rooms from a calendar: once meetings are in time order, you only need to compare each meeting with the previous one.

**How to spot it:**
- Meetings, bookings, time ranges
- Overlap / merge / insert
- How many at the same time
- Remove the fewest to avoid overlap

**Template:**

```java
Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
int[] last = intervals[0];
for (int i = 1; i < intervals.length; i++) {
    int[] cur = intervals[i];
    if (cur[0] < last[1]) { /* overlap */ }
    last = cur;
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Detect any overlap](#141-detect-any-overlap) | After sorting, does current start before previous ends? | `Sort by start → check adjacent` | O(n log n) |
| [Merge overlaps](#142-merge-overlaps) | Should current interval extend the previous merged range? | `Sort by start → compare last merged` | O(n log n) |
| [Insert interval](#143-insert-interval) | Where does the new interval fit among existing ranges? | `Before → merge → after` | O(n) |
| [Minimum meeting rooms](#144-minimum-meeting-rooms) | Can a new meeting reuse the earliest-free room? | `Sort + min heap of end times` | O(n log n) |
| [Maximum simultaneous intervals](#145-maximum-simultaneous-intervals) | How many intervals are active at once? | `Sweep line / start-end pointers` | O(n log n) |
| [Remove minimum overlaps](#146-remove-minimum-overlaps) | Which interval should I keep to leave maximum future space? | `Sort by end → greedy` | O(n log n) |
| [Interval intersection](#147-interval-intersection) | How do two sorted interval lists overlap? | `Two pointers` | O(n + m) |

#### 14.1 Detect any overlap

- **Big pattern:** Intervals
- **Ask yourself:** *After sorting, does current start before previous ends?*
- **Small recipe:** `Sort by start → check adjacent`
- **Memory sentence:** Intervals → Detect Any Overlap → sort by start → current.start < previous.end?
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Sort by start. If any interval starts before the previous one ends, they overlap.

**Steps:**
1. Sort by start.
2. For i ≥ 1: if start[i] < end[i−1] → overlap.
3. Decide whether touching ([1,3] & [3,5]) counts.

**Tiny example:**

```text
[[0,30], [5,10], [15,20]]
5 < 30 → overlap → cannot attend all meetings
```

**Java:**

```java
boolean canAttendAll(int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> Integer.compare(a[0], b[0]));
    for (int i = 1; i < meetings.length; i++) {
        if (meetings[i][0] < meetings[i - 1][1]) {
            return false;                 // starts before the previous one ends
        }
    }
    return true;
}
```

**Complexity:** time O(n log n) · space O(1) extra

> ⚠️ **Trap:** Equality usually means touching but not overlapping: [1,3] and [3,5].

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II · Meeting Rooms I

#### 14.2 Merge overlaps

- **Big pattern:** Intervals
- **Ask yourself:** *Should current interval extend the previous merged range?*
- **Small recipe:** `Sort by start → compare last merged`
- **Memory sentence:** Intervals → Merge Intervals → overlap → extend end; else start new
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Sort by start. Compare each interval with the LAST MERGED one: if it overlaps, stretch the end; if not, start a new merged interval.

**Steps:**
1. Sort by start.
2. If cur.start ≤ last.end → last.end = max(last.end, cur.end).
3. Else add cur as a new interval.

**Tiny example:**

```text
[[1,3], [2,6], [8,10], [15,18]]
[1,3]+[2,6] → [1,6]; [8,10] new; [15,18] new
```

**Java:**

```java
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    List<int[]> merged = new ArrayList<>();
    for (int[] cur : intervals) {
        if (!merged.isEmpty() && cur[0] <= merged.get(merged.size() - 1)[1]) {
            int[] last = merged.get(merged.size() - 1);
            last[1] = Math.max(last[1], cur[1]);
        } else {
            merged.add(new int[]{cur[0], cur[1]});
        }
    }
    return merged.toArray(new int[0][]);
}
```

**Complexity:** time O(n log n) · space O(n)

> ⚠️ **Trap:** Compare against last MERGED interval, not blindly previous original interval.

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II

#### 14.3 Insert interval

- **Big pattern:** Intervals
- **Ask yourself:** *Where does the new interval fit among existing ranges?*
- **Small recipe:** `Before → merge → after`
- **Memory sentence:** Intervals → Insert Interval → three phases
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** The list is already sorted and non-overlapping. Copy everything that ends before the new interval, merge everything that overlaps it, then copy the rest.

**Steps:**
1. Phase 1: end < new.start → copy.
2. Phase 2: start ≤ new.end → widen the new interval.
3. Add the new interval, then Phase 3: copy the rest.

**Tiny example:**

```text
[[1,2], [3,5], [6,7], [8,10]] + [4,8]
copy [1,2]; merge [3,5] [6,7] [8,10] → [3,10]
```

**Java:**

```java
int[][] insert(int[][] intervals, int[] add) {
    List<int[]> out = new ArrayList<>();
    int i = 0, n = intervals.length;
    int start = add[0], end = add[1];
    while (i < n && intervals[i][1] < start) out.add(intervals[i++]);        // before
    while (i < n && intervals[i][0] <= end) {                               // overlap
        start = Math.min(start, intervals[i][0]);
        end = Math.max(end, intervals[i][1]);
        i++;
    }
    out.add(new int[]{start, end});
    while (i < n) out.add(intervals[i++]);                                  // after
    return out.toArray(new int[0][]);
}
```

**Complexity:** time O(n) · space O(n)

> ⚠️ **Trap:** This assumes existing intervals are already sorted and non-overlapping.

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II

#### 14.4 Minimum meeting rooms

- **Big pattern:** Intervals
- **Ask yourself:** *Can a new meeting reuse the earliest-free room?*
- **Small recipe:** `Sort + min heap of end times`
- **Memory sentence:** Intervals → Meeting Rooms II → reuse earliest free room
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Sort by start. A min-heap holds the end times of busy rooms. If the room that frees earliest is free before this meeting starts, reuse it; otherwise open a new room.

**Steps:**
1. Sort by start.
2. If heap.peek() ≤ start → poll.
3. offer end. Rooms needed = heap size at the end.

**Tiny example:**

```text
[[0,30], [5,10], [15,20]] → 2 rooms
```

**Java:**

```java
int minMeetingRooms(int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> Integer.compare(a[0], b[0]));
    PriorityQueue<Integer> ends = new PriorityQueue<>();
    for (int[] m : meetings) {
        if (!ends.isEmpty() && ends.peek() <= m[0]) ends.poll();
        ends.offer(m[1]);
    }
    return ends.size();
}
```

**Complexity:** time O(n log n) · space O(n)

> ⚠️ **Trap:** Heap stores end times, not whole intervals.

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II · Minimum Meeting Rooms

#### 14.5 Maximum simultaneous intervals

- **Big pattern:** Intervals
- **Ask yourself:** *How many intervals are active at once?*
- **Small recipe:** `Sweep line / start-end pointers`
- **Memory sentence:** Intervals → Maximum simultaneous intervals → Sweep line / start-end pointers
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Turn each interval into two events: +1 at its start, −1 at its end. Sort the starts and ends separately and sweep through time, tracking how many are active.

**Steps:**
1. starts[] and ends[] sorted.
2. If the next start < the next end → a new interval opens (active++).
3. Else one closes (active−−). Track the maximum.

**Tiny example:**

```text
[[1,5], [2,6], [4,8]]
at time 4 all three are active → 3
```

**Java:**

```java
int maxSimultaneous(int[][] intervals) {
    int n = intervals.length;
    int[] starts = new int[n], ends = new int[n];
    for (int i = 0; i < n; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }
    Arrays.sort(starts);
    Arrays.sort(ends);
    int active = 0, best = 0, e = 0;
    for (int s : starts) {
        while (e < n && ends[e] <= s) {   // close everything that ended before s
            active--;
            e++;
        }
        active++;
        best = Math.max(best, active);
    }
    return best;
}
```

**Complexity:** time O(n log n) · space O(n)

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II

#### 14.6 Remove minimum overlaps

- **Big pattern:** Intervals
- **Ask yourself:** *Which interval should I keep to leave maximum future space?*
- **Small recipe:** `Sort by end → greedy`
- **Memory sentence:** Intervals → Remove minimum overlaps → Sort by end → greedy
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Keep as many intervals as possible: sort by END and always keep the interval that finishes first — it leaves the most room for the rest. Removals = total − kept.

**Steps:**
1. Sort by end.
2. If start ≥ lastEnd → keep it, lastEnd = end.
3. Else it overlaps → remove it.

**Tiny example:**

```text
[[1,2], [2,3], [3,4], [1,3]]
keep [1,2], [2,3], [3,4]; remove [1,3] → 1
```

**Java:**

```java
int eraseOverlapIntervals(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[1], b[1]));
    int kept = 0;
    long lastEnd = Long.MIN_VALUE;
    for (int[] cur : intervals) {
        if (cur[0] >= lastEnd) {
            kept++;
            lastEnd = cur[1];
        }
    }
    return intervals.length - kept;
}
```

**Complexity:** time O(n log n) · space O(1) extra

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II

#### 14.7 Interval intersection

- **Big pattern:** Intervals
- **Ask yourself:** *How do two sorted interval lists overlap?*
- **Small recipe:** `Two pointers`
- **Memory sentence:** Intervals → Interval Intersection → overlap = max starts .. min ends
- **Words that give it away:** meeting, booking, start/end, overlap, schedule

**In plain English:** Two sorted lists, one pointer in each. The overlap of two intervals is [max(starts), min(ends)] if that is valid. Then move the pointer whose interval ends first.

**Steps:**
1. lo = max(a.start, b.start), hi = min(a.end, b.end).
2. If lo ≤ hi → add [lo, hi].
3. Advance the list whose current interval ends first.

**Tiny example:**

```text
A [[0,2], [5,10]], B [[1,5], [8,12]]
[1,2], [5,5], [8,10]
```

**Java:**

```java
int[][] intervalIntersection(int[][] a, int[][] b) {
    List<int[]> out = new ArrayList<>();
    int i = 0, j = 0;
    while (i < a.length && j < b.length) {
        int lo = Math.max(a[i][0], b[j][0]);
        int hi = Math.min(a[i][1], b[j][1]);
        if (lo <= hi) out.add(new int[]{lo, hi});
        if (a[i][1] < b[j][1]) i++;
        else j++;
    }
    return out.toArray(new int[0][]);
}
```

**Complexity:** time O(n + m) · space O(n + m) for the output

> ⚠️ **Trap:** Advance the interval that ends first.

**Practise on:** Merge Intervals · Insert Interval · Meeting Rooms I/II · Interval List Intersections

**Runnable in SkillForge (Intervals):** Merge Overlapping Intervals (Medium) · Meeting Rooms II (Medium) · Non-overlapping Intervals (Medium)

---

### 15. Binary Search

**The big idea:** When data is sorted (or an answer is "yes, yes, yes… then no, no"), look at the middle and throw away the half that cannot contain the answer. Each step halves the work → O(log n).

**Think of it like this:** Guessing a number between 1 and 100: "higher / lower" lets you find it in 7 guesses.

**How to spot it:**
- Sorted array
- Must be O(log n)
- First / last position
- "Minimum X such that …" with a yes/no check

**Template:**

```java
int lo = 0, hi = n - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;   // no overflow
    if (nums[mid] == target) return mid;
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid - 1;
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Exact search](#151-exact-search) | Can I discard half after one comparison? | `Classic binary search` | O(log n) |
| [First / last occurrence](#152-first--last-occurrence) | After finding target, should I keep searching one side? | `Biased binary search` | O(log n) |
| [Lower / upper bound](#153-lower--upper-bound) | Am I searching for the first position where a condition becomes true? | `Boundary binary search` | O(log n) |
| [Rotated array](#154-rotated-array) | Which half is guaranteed sorted right now? | `Identify sorted half` | O(log n) |
| [Binary search on answer](#155-binary-search-on-answer) | If X works, do all larger/smaller X also work? | `Monotonic feasibility` | O(n log max) |
| [Peak finding](#156-peak-finding) | Which direction is rising? | `Compare mid with neighbor` | O(log n) |

#### 15.1 Exact search

- **Big pattern:** Binary Search
- **Ask yourself:** *Can I discard half after one comparison?*
- **Small recipe:** `Classic binary search`
- **Memory sentence:** Binary Search → Classic Binary Search → mid → discard half
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** Compare the middle with the target. Equal → done. Smaller → search the right half. Bigger → search the left half.

**Steps:**
1. lo = 0, hi = n − 1.
2. mid = lo + (hi − lo) / 2.
3. Move lo or hi past mid; stop when lo > hi.

**Tiny example:**

```text
[1, 3, 5, 7, 9], target 7
mid 5 < 7 → lo = 3; mid 7 ✓ → index 3
```

**Java:**

```java
int search(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;
        if (nums[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

**Complexity:** time O(log n) · space O(1)

> ⚠️ **Trap:** Use left + (right-left)/2 to avoid overflow.

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas · Search in sorted array

#### 15.2 First / last occurrence

- **Big pattern:** Binary Search
- **Ask yourself:** *After finding target, should I keep searching one side?*
- **Small recipe:** `Biased binary search`
- **Memory sentence:** Binary Search → First / last occurrence → Biased binary search
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** When you find the target, do not stop: save the index and keep searching LEFT (for the first) or RIGHT (for the last).

**Steps:**
1. On a match: ans = mid.
2. First occurrence → hi = mid − 1. Last → lo = mid + 1.
3. Return ans (−1 if never found).

**Tiny example:**

```text
[1, 2, 2, 2, 3], target 2
first: match at 2 → go left → match at 1 → go left → answer 1
```

**Java:**

```java
int firstOccurrence(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1, ans = -1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) {
            ans = mid;
            hi = mid - 1;        // keep looking left (use lo = mid + 1 for the LAST one)
        } else if (nums[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid - 1;
        }
    }
    return ans;
}
```

**Complexity:** time O(log n) · space O(1)

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas

#### 15.3 Lower / upper bound

- **Big pattern:** Binary Search
- **Ask yourself:** *Am I searching for the first position where a condition becomes true?*
- **Small recipe:** `Boundary binary search`
- **Memory sentence:** Binary Search → First True / Lower Bound → when condition true → move left
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** Find the first index where a condition becomes true (e.g. nums[i] ≥ target). Use the half-open range [lo, hi): if mid satisfies it, the answer is mid or to its left.

**Steps:**
1. lo = 0, hi = n.
2. If nums[mid] ≥ target → hi = mid, else lo = mid + 1.
3. When lo == hi, that is the first true position (n if none).

**Tiny example:**

```text
[1, 3, 5, 7], target 4
first value ≥ 4 is 5 at index 2
```

**Java:**

```java
int lowerBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;          // half-open [lo, hi)
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid; // condition true → answer is mid or left
        else lo = mid + 1;
    }
    return lo;                             // upper bound: use nums[mid] > target
}
```

**Complexity:** time O(log n) · space O(1)

> ⚠️ **Trap:** This half-open [left,right) template is very reusable.

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas · First element >= target

#### 15.4 Rotated array

- **Big pattern:** Binary Search
- **Ask yourself:** *Which half is guaranteed sorted right now?*
- **Small recipe:** `Identify sorted half`
- **Memory sentence:** Binary Search → Rotated array → Identify sorted half
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** In a rotated sorted array, at least one half around mid is still sorted. Check if the target lies inside that sorted half; if yes go there, otherwise go to the other half.

**Steps:**
1. If nums[lo] ≤ nums[mid] → the left half is sorted.
2. Target inside [nums[lo], nums[mid]) → hi = mid − 1, else lo = mid + 1.
3. Otherwise the right half is sorted: mirror the check.

**Tiny example:**

```text
[4, 5, 6, 7, 0, 1, 2], target 0
mid 7: left sorted, 0 not in [4,7) → go right
mid 1: right sorted… → found at index 4
```

**Java:**

```java
int searchRotated(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;
        if (nums[lo] <= nums[mid]) {                       // left half sorted
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {                                           // right half sorted
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

**Complexity:** time O(log n) · space O(1)

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas

#### 15.5 Binary search on answer

- **Big pattern:** Binary Search
- **Ask yourself:** *If X works, do all larger/smaller X also work?*
- **Small recipe:** `Monotonic feasibility`
- **Memory sentence:** Binary Search → Binary Search on Answer → if mid works, tighten answer
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** Sometimes you binary search the ANSWER, not the array. If "can we do it with speed X?" is yes for big X and no for small X, find the smallest X that says yes.

**Steps:**
1. Pick the answer range [lo, hi].
2. Write feasible(x) — a yes/no check.
3. feasible(mid) → hi = mid, else lo = mid + 1.

**Tiny example:**

```text
Koko: piles [3, 6, 7, 11], 8 hours
speed 4 works, speed 3 does not → answer 4
```

**Java:**

```java
int minEatingSpeed(int[] piles, int hours) {
    int lo = 1, hi = Arrays.stream(piles).max().getAsInt();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canFinish(piles, hours, mid)) hi = mid;   // works → try smaller
        else lo = mid + 1;
    }
    return lo;
}

boolean canFinish(int[] piles, int hours, int speed) {
    long needed = 0;
    for (int p : piles) needed += (p + speed - 1) / speed;   // ceil(p / speed)
    return needed <= hours;
}
```

**Complexity:** time O(n log max) · space O(1)

> ⚠️ **Trap:** You need a monotonic yes/no function.

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas · Koko Eating Bananas / capacity problems

#### 15.6 Peak finding

- **Big pattern:** Binary Search
- **Ask yourself:** *Which direction is rising?*
- **Small recipe:** `Compare mid with neighbor`
- **Memory sentence:** Binary Search → Peak finding → Compare mid with neighbor
- **Words that give it away:** sorted, half, boundary, monotonic answer

**In plain English:** Compare mid with its right neighbour. If the slope goes UP, a peak must exist to the right; otherwise mid or something to its left is a peak.

**Steps:**
1. lo = 0, hi = n − 1.
2. nums[mid] < nums[mid + 1] → lo = mid + 1.
3. Else hi = mid. Stop when lo == hi.

**Tiny example:**

```text
[1, 2, 3, 1]
mid 1: 2 < 3 → go right; mid 2: 3 > 1 → hi = 2
peak at index 2
```

**Java:**

```java
int findPeakElement(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] < nums[mid + 1]) lo = mid + 1;   // climbing → peak on the right
        else hi = mid;
    }
    return lo;
}
```

**Complexity:** time O(log n) · space O(1)

**Practise on:** Binary Search · Search in Rotated Sorted Array · Koko Eating Bananas

**Runnable in SkillForge (Binary Search):** Binary Search (Easy) · Find First and Last Position of an Element in a Sorted Array (Medium) · Search Insert Position (Easy) · Search in Rotated Sorted Array (Medium) · Peak Index in a Mountain Array (Medium) · Sqrt(x) (Easy) · Koko Eating Bananas (Binary Search on Answer) (Medium) · Median of Two Sorted Arrays (Hard)

---

### 16. Greedy

**The big idea:** Make the choice that looks best RIGHT NOW and never go back. It works only when you can explain why the local best choice never blocks the overall best answer.

**Think of it like this:** Paying with the fewest notes in rupees: always hand over the biggest note that fits — it works because of how the note values are designed.

**How to spot it:**
- Scheduling / intervals (sort by end)
- Farthest reach / jumps
- A simple rule you can prove with an "exchange" argument

**Template:**

```java
// 1. sort by the key that makes the greedy choice safe (often END time)
// 2. walk once, taking every item that fits
// 3. never undo a choice
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Interval scheduling](#161-interval-scheduling) | Which choice leaves the most room for future choices? | `Sort by end → take earliest finish` | O(n log n) |
| [Jump Game](#162-jump-game) | What is the farthest index reachable so far? | `Track farthest reachable` | O(n) |
| [Gas station](#163-gas-station) | If starting here fails, can any point inside this failed segment work? | `Reset after failed prefix` | O(n) |
| [Minimum arrows / overlap removal](#164-minimum-arrows--overlap-removal) | Can one local endpoint cover as much future work as possible? | `Sort intervals strategically` | O(n log n) |
| [Activity selection](#165-activity-selection) | Can I safely commit to the earliest-finishing activity? | `Earliest compatible finish` | O(n log n) |

#### 16.1 Interval scheduling

- **Big pattern:** Greedy
- **Ask yourself:** *Which choice leaves the most room for future choices?*
- **Small recipe:** `Sort by end → take earliest finish`
- **Memory sentence:** Greedy → Sort by End Time → keep interval ending earliest
- **Words that give it away:** local best, earliest finish, minimum removals, farthest reach

**In plain English:** To fit the most meetings, always pick the meeting that ENDS earliest among those that still fit. Finishing early leaves the most time for the rest.

**Steps:**
1. Sort by end time.
2. Take a meeting if it starts after the last chosen one ends.
3. Count what you took.

**Tiny example:**

```text
[[1,4], [3,5], [0,6], [5,7], [8,9]]
take [1,4], [5,7], [8,9] → 3 meetings
```

**Java:**

```java
int maxMeetings(int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> Integer.compare(a[1], b[1]));
    int count = 0;
    long lastEnd = Long.MIN_VALUE;
    for (int[] m : meetings) {
        if (m[0] >= lastEnd) {
            count++;
            lastEnd = m[1];
        }
    }
    return count;
}
```

**Complexity:** time O(n log n) · space O(1) extra

> ⚠️ **Trap:** Greedy needs a reason/proof why local choice is safe.

**Practise on:** Jump Game · Non-overlapping Intervals · Gas Station · Activity Selection

#### 16.2 Jump Game

- **Big pattern:** Greedy
- **Ask yourself:** *What is the farthest index reachable so far?*
- **Small recipe:** `Track farthest reachable`
- **Memory sentence:** Greedy → Farthest Reach → track maximum reachable index
- **Words that give it away:** local best, earliest finish, minimum removals, farthest reach

**In plain English:** Track the farthest index you can reach so far. If you ever stand on an index beyond that, you are stuck.

**Steps:**
1. farthest = 0.
2. For each i: if i > farthest → false.
3. farthest = max(farthest, i + nums[i]).

**Tiny example:**

```text
[3, 2, 1, 0, 4]
farthest stays 3; index 4 > 3 → cannot reach
```

**Java:**

```java
boolean canJump(int[] nums) {
    int farthest = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > farthest) return false;
        farthest = Math.max(farthest, i + nums[i]);
    }
    return true;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** If current index is beyond farthest, you are stuck.

**Practise on:** Jump Game · Non-overlapping Intervals · Gas Station

#### 16.3 Gas station

- **Big pattern:** Greedy
- **Ask yourself:** *If starting here fails, can any point inside this failed segment work?*
- **Small recipe:** `Reset after failed prefix`
- **Memory sentence:** Greedy → Gas station → Reset after failed prefix
- **Words that give it away:** local best, earliest finish, minimum removals, farthest reach

**In plain English:** If the total gas is less than the total cost, it is impossible. Otherwise drive from a candidate start; whenever your tank goes negative, no station between the start and here can work — so start again from the next station.

**Steps:**
1. total += gas − cost for every station.
2. tank += gas − cost; if tank < 0 → start = i + 1, tank = 0.
3. Answer = total ≥ 0 ? start : −1.

**Tiny example:**

```text
gas [1,2,3,4,5], cost [3,4,5,1,2]
tank fails at 0,1,2 → start = 3 → works
```

**Java:**

```java
int canCompleteCircuit(int[] gas, int[] cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < gas.length; i++) {
        int gain = gas[i] - cost[i];
        total += gain;
        tank += gain;
        if (tank < 0) {          // cannot reach i + 1 from start
            start = i + 1;
            tank = 0;
        }
    }
    return total >= 0 ? start : -1;
}
```

**Complexity:** time O(n) · space O(1)

**Practise on:** Jump Game · Non-overlapping Intervals · Gas Station

#### 16.4 Minimum arrows / overlap removal

- **Big pattern:** Greedy
- **Ask yourself:** *Can one local endpoint cover as much future work as possible?*
- **Small recipe:** `Sort intervals strategically`
- **Memory sentence:** Greedy → Minimum arrows / overlap removal → Sort intervals strategically
- **Words that give it away:** local best, earliest finish, minimum removals, farthest reach

**In plain English:** Balloons are ranges. Sort by END. Shoot an arrow at the end of the first balloon — it pops every balloon that starts before that point. The next arrow goes at the end of the first balloon it missed.

**Steps:**
1. Sort by end.
2. arrowAt = end of the first balloon, arrows = 1.
3. If a balloon starts after arrowAt → new arrow at its end.

**Tiny example:**

```text
[[10,16], [2,8], [1,6], [7,12]]
arrow at 6 pops [1,6], [2,8]; arrow at 12 pops [7,12], [10,16] → 2
```

**Java:**

```java
int findMinArrowShots(int[][] points) {
    Arrays.sort(points, (a, b) -> Integer.compare(a[1], b[1]));
    int arrows = 1;
    int arrowAt = points[0][1];
    for (int[] p : points) {
        if (p[0] > arrowAt) {    // this balloon was not popped
            arrows++;
            arrowAt = p[1];
        }
    }
    return arrows;
}
```

**Complexity:** time O(n log n) · space O(1) extra

**Practise on:** Jump Game · Non-overlapping Intervals · Gas Station

#### 16.5 Activity selection

- **Big pattern:** Greedy
- **Ask yourself:** *Can I safely commit to the earliest-finishing activity?*
- **Small recipe:** `Earliest compatible finish`
- **Memory sentence:** Greedy → Activity selection → Earliest compatible finish
- **Words that give it away:** local best, earliest finish, minimum removals, farthest reach

**In plain English:** The classic proof: if the best plan starts with some activity A, swapping A for the activity that finishes earliest never makes the plan worse. So picking the earliest finish first is always safe.

**Steps:**
1. Sort activities by finish time.
2. Pick the first; then pick each next one that starts after the last picked finishes.
3. Return the picked list.

**Tiny example:**

```text
start [1,3,0,5,8,5], finish [2,4,6,7,9,9]
picked activities 0, 1, 3, 4
```

**Java:**

```java
List<Integer> selectActivities(int[] start, int[] finish) {
    Integer[] order = new Integer[start.length];
    for (int i = 0; i < order.length; i++) order[i] = i;
    Arrays.sort(order, (a, b) -> Integer.compare(finish[a], finish[b]));
    List<Integer> picked = new ArrayList<>();
    int lastFinish = Integer.MIN_VALUE;
    for (int i : order) {
        if (start[i] >= lastFinish) {
            picked.add(i);
            lastFinish = finish[i];
        }
    }
    return picked;
}
```

**Complexity:** time O(n log n) · space O(n)

**Practise on:** Jump Game · Non-overlapping Intervals · Gas Station

**Runnable in SkillForge (Greedy):** Jump Game (Medium) · Best Time to Buy and Sell Stock II (Medium) · Largest Number (Medium)

---

## Phase 4 — Linked Lists, Trees & Tries

### 17. Linked List

**The big idea:** A linked list is a chain of nodes; each node knows only the NEXT node. Most tricks are about moving pointers carefully so you never lose the rest of the chain.

**Think of it like this:** A treasure hunt: each clue tells you only where the next clue is.

**How to spot it:**
- Reverse / reorder a list
- Middle node, N-th from end
- Cycle detection
- Merge sorted lists

**Template:**

```java
class ListNode { int val; ListNode next; ListNode(int v) { val = v; } }

ListNode dummy = new ListNode(0);   // avoids special cases for the head
ListNode tail = dummy;
// ... tail.next = node; tail = tail.next;
return dummy.next;
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Reverse list](#171-reverse-list) | Can I safely flip one arrow at a time? | `prev / curr / next` | O(n) |
| [Middle node](#172-middle-node) | Can one pointer move twice as fast? | `Slow + fast` | O(n) |
| [Cycle detection](#173-cycle-detection) | Will runners meet if the track loops? | `Floyd slow/fast` | O(n) |
| [Merge sorted lists](#174-merge-sorted-lists) | Can I always attach the smaller front node? | `Dummy head + tail` | O(n + m) |
| [Remove nth from end](#175-remove-nth-from-end) | Can I keep pointers N nodes apart? | `Two pointers with gap` | O(n) |
| [Reorder list](#176-reorder-list) | Can I combine three known recipes? | `Middle → reverse → merge` | O(n) |

#### 17.1 Reverse list

- **Big pattern:** Linked List
- **Ask yourself:** *Can I safely flip one arrow at a time?*
- **Small recipe:** `prev / curr / next`
- **Memory sentence:** Linked List → Reverse Linked List → save next → reverse arrow → advance
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** Walk the list flipping one arrow at a time. Always save curr.next BEFORE changing it, or you lose the rest of the list.

**Steps:**
1. prev = null, curr = head.
2. next = curr.next; curr.next = prev.
3. prev = curr; curr = next. Return prev.

**Tiny example:**

```text
1 → 2 → 3
null ← 1   2 → 3
null ← 1 ← 2   3
3 → 2 → 1
```

**Java:**

```java
ListNode reverseList(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;   // save before rewiring
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Save next BEFORE changing curr.next.

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List

#### 17.2 Middle node

- **Big pattern:** Linked List
- **Ask yourself:** *Can one pointer move twice as fast?*
- **Small recipe:** `Slow + fast`
- **Memory sentence:** Linked List → Middle Node → slow 1 step, fast 2
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** Two runners: slow moves 1 step, fast moves 2. When fast reaches the end, slow is in the middle.

**Steps:**
1. slow = fast = head.
2. while (fast != null && fast.next != null): slow = slow.next; fast = fast.next.next.
3. Return slow.

**Tiny example:**

```text
1 → 2 → 3 → 4 → 5
slow 1,2,3 / fast 1,3,5 → middle 3
```

**Java:**

```java
ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;       // for even length this is the second middle
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** For even length this returns the second middle.

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List · Middle of Linked List

#### 17.3 Cycle detection

- **Big pattern:** Linked List
- **Ask yourself:** *Will runners meet if the track loops?*
- **Small recipe:** `Floyd slow/fast`
- **Memory sentence:** Linked List → Cycle Detection → if runners meet → cycle
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** If the list loops, the fast runner will eventually lap the slow runner and they meet. If fast reaches null, there is no loop.

**Steps:**
1. slow = fast = head.
2. Move slow 1, fast 2.
3. slow == fast → cycle. fast reaches null → no cycle.

**Tiny example:**

```text
3 → 2 → 0 → −4 → (back to 2)
runners meet inside the loop → true
```

**Java:**

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;   // compare nodes, not values
    }
    return false;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Compare node references, not node values.

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List

#### 17.4 Merge sorted lists

- **Big pattern:** Linked List
- **Ask yourself:** *Can I always attach the smaller front node?*
- **Small recipe:** `Dummy head + tail`
- **Memory sentence:** Linked List → Dummy Head → dummy simplifies head edge cases
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** Start with a dummy node. Repeatedly attach the smaller of the two front nodes to the tail. When one list runs out, attach the other.

**Steps:**
1. dummy + tail pointer.
2. Attach the smaller front node; advance that list and the tail.
3. Attach whatever is left; return dummy.next.

**Tiny example:**

```text
1 → 3 and 2 → 4
take 1, 2, 3, 4 → 1 → 2 → 3 → 4
```

**Java:**

```java
ListNode mergeTwoLists(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (a != null && b != null) {
        if (a.val <= b.val) { tail.next = a; a = a.next; }
        else { tail.next = b; b = b.next; }
        tail = tail.next;
    }
    tail.next = (a != null) ? a : b;
    return dummy.next;
}
```

**Complexity:** time O(n + m) · space O(1)

> ⚠️ **Trap:** Dummy nodes remove special handling for the first result node.

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List · Merge / delete / construct lists

#### 17.5 Remove nth from end

- **Big pattern:** Linked List
- **Ask yourself:** *Can I keep pointers N nodes apart?*
- **Small recipe:** `Two pointers with gap`
- **Memory sentence:** Linked List → Remove nth from end → Two pointers with gap
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** Move a fast pointer N steps ahead. Then move both together; when fast reaches the last node, slow is just before the node to delete.

**Steps:**
1. dummy → head; fast = slow = dummy.
2. Move fast N steps.
3. Move both until fast.next == null; slow.next = slow.next.next.

**Tiny example:**

```text
1 → 2 → 3 → 4 → 5, N = 2
slow stops at 3 → delete 4 → 1 → 2 → 3 → 5
```

**Java:**

```java
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i < n; i++) fast = fast.next;
    while (fast.next != null) {
        fast = fast.next;
        slow = slow.next;
    }
    slow.next = slow.next.next;
    return dummy.next;
}
```

**Complexity:** time O(n) · space O(1)

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List

#### 17.6 Reorder list

- **Big pattern:** Linked List
- **Ask yourself:** *Can I combine three known recipes?*
- **Small recipe:** `Middle → reverse → merge`
- **Memory sentence:** Linked List → Reorder list → Middle → reverse → merge
- **Words that give it away:** next pointer, cycle, middle, reverse, nth from end

**In plain English:** Combine three recipes you already know: find the middle, reverse the second half, then weave the two halves together.

**Steps:**
1. Middle with slow/fast.
2. Reverse the second half.
3. Alternate nodes: first, last, second, second-last…

**Tiny example:**

```text
1 → 2 → 3 → 4 → 5
halves 1 → 2 → 3 and 5 → 4
woven 1 → 5 → 2 → 4 → 3
```

**Java:**

```java
void reorderList(ListNode head) {
    if (head == null || head.next == null) return;
    ListNode slow = head, fast = head;
    while (fast.next != null && fast.next.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    ListNode second = null, curr = slow.next;   // reverse the second half
    slow.next = null;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = second;
        second = curr;
        curr = next;
    }
    ListNode first = head;                      // weave
    while (second != null) {
        ListNode n1 = first.next, n2 = second.next;
        first.next = second;
        second.next = n1;
        first = n1;
        second = n2;
    }
}
```

**Complexity:** time O(n) · space O(1)

**Practise on:** Reverse Linked List · Linked List Cycle · Reorder List

**Runnable in SkillForge (Linked List):** Reverse Linked List (Easy) · Merge Two Sorted Lists (Easy) · Linked List Cycle (Easy) · Add Two Numbers (Linked Lists) (Medium)

---

### 18. Trees

**The big idea:** A binary tree node has a value and up to two children. Nearly every tree problem is a recursive function: solve the left subtree, solve the right subtree, combine the answers.

**Think of it like this:** A family tree: to count everyone, each person asks their two children "how many are in your branch?" and adds 1 for themselves.

**How to spot it:**
- Anything with TreeNode
- Depth / height / diameter
- Paths from root to leaf
- BST (left < node < right)

**Template:**

```java
class TreeNode { int val; TreeNode left, right; TreeNode(int v) { val = v; } }

int solve(TreeNode node) {
    if (node == null) return BASE;        // always handle null first
    int left = solve(node.left);
    int right = solve(node.right);
    return combine(node.val, left, right);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Preorder](#181-preorder) | Do I need the parent before its children? | `Node → left → right` | O(n) |
| [Inorder](#182-inorder) | Do I want BST values in sorted order? | `Left → node → right` | O(n) |
| [Postorder](#183-postorder) | Does parent depend on child results? | `Left → right → node` | O(n) |
| [Height / depth](#184-height--depth) | Can each subtree report a small answer upward? | `DFS returns child result` | O(n) |
| [Path problems](#185-path-problems) | Do I carry a running state from root to leaf? | `DFS + path state` | O(n) |
| [Lowest common ancestor](#186-lowest-common-ancestor) | Where do two targets diverge? | `Recursive split / BST ordering` | O(n) |

#### 18.1 Preorder

- **Big pattern:** Trees
- **Ask yourself:** *Do I need the parent before its children?*
- **Small recipe:** `Node → left → right`
- **Memory sentence:** Trees → Preorder Traversal → process before children
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** Visit the node FIRST, then its left subtree, then its right subtree. Useful when a parent must be handled before its children (copying, serialising).

**Steps:**
1. If node is null → return.
2. Process node.
3. Recurse left, then right.

**Tiny example:**

```text
tree 1 / (2, 3)
order: 1, 2, 3
```

**Java:**

```java
void preorder(TreeNode node, List<Integer> out) {
    if (node == null) return;
    out.add(node.val);           // node first
    preorder(node.left, out);
    preorder(node.right, out);
}
```

**Complexity:** time O(n) · space O(height)

> ⚠️ **Trap:** Good for serialization / copying-style logic.

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor · Preorder

#### 18.2 Inorder

- **Big pattern:** Trees
- **Ask yourself:** *Do I want BST values in sorted order?*
- **Small recipe:** `Left → node → right`
- **Memory sentence:** Trees → Inorder Traversal → BST gives sorted order
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** Left subtree, then the node, then the right subtree. For a Binary Search Tree this gives the values in sorted order.

**Steps:**
1. Recurse left.
2. Process node.
3. Recurse right.

**Tiny example:**

```text
BST 2 / (1, 3)
order: 1, 2, 3 (sorted)
```

**Java:**

```java
void inorder(TreeNode node, List<Integer> out) {
    if (node == null) return;
    inorder(node.left, out);
    out.add(node.val);           // node in the middle
    inorder(node.right, out);
}
```

**Complexity:** time O(n) · space O(height)

> ⚠️ **Trap:** Inorder of a BST is sorted.

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor · BST traversal

#### 18.3 Postorder

- **Big pattern:** Trees
- **Ask yourself:** *Does parent depend on child results?*
- **Small recipe:** `Left → right → node`
- **Memory sentence:** Trees → Postorder Traversal → children first
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** Children first, node last. Use it when the node’s answer depends on its children’s answers (sizes, heights, deleting a tree).

**Steps:**
1. Recurse left.
2. Recurse right.
3. Process node using the children’s results.

**Tiny example:**

```text
tree 1 / (2, 3)
order: 2, 3, 1
```

**Java:**

```java
int subtreeSize(TreeNode node) {
    if (node == null) return 0;
    int left = subtreeSize(node.left);
    int right = subtreeSize(node.right);
    return left + right + 1;     // node last, using child answers
}
```

**Complexity:** time O(n) · space O(height)

> ⚠️ **Trap:** Use when parent answer depends on completed child answers.

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor · Subtree aggregation

#### 18.4 Height / depth

- **Big pattern:** Trees
- **Ask yourself:** *Can each subtree report a small answer upward?*
- **Small recipe:** `DFS returns child result`
- **Memory sentence:** Trees → Recursive DFS → solve children → combine
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** An empty tree has height 0. Any other tree is 1 (this node) + the taller of its two subtrees.

**Steps:**
1. null → 0.
2. left = height(left), right = height(right).
3. Return 1 + max(left, right).

**Tiny example:**

```text
3 / (9, 20 / (15, 7))
height = 1 + max(1, 2) = 3
```

**Java:**

```java
int maxDepth(TreeNode node) {
    if (node == null) return 0;
    return 1 + Math.max(maxDepth(node.left), maxDepth(node.right));
}
```

**Complexity:** time O(n) · space O(height)

> ⚠️ **Trap:** Every recursive tree solution needs a clear null base case.

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor

#### 18.5 Path problems

- **Big pattern:** Trees
- **Ask yourself:** *Do I carry a running state from root to leaf?*
- **Small recipe:** `DFS + path state`
- **Memory sentence:** Trees → Path problems → DFS + path state
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** Carry information DOWN the tree as you recurse (e.g. the remaining sum). At a leaf, check whether the path worked.

**Steps:**
1. Subtract node.val from the remaining target.
2. At a leaf: success if remaining == 0.
3. Otherwise try the left or right child.

**Tiny example:**

```text
target 22: 5 → 4 → 11 → 2
remaining 22 → 17 → 13 → 2 → 0 at a leaf ✓
```

**Java:**

```java
boolean hasPathSum(TreeNode node, int target) {
    if (node == null) return false;
    int remaining = target - node.val;
    if (node.left == null && node.right == null) return remaining == 0;
    return hasPathSum(node.left, remaining) || hasPathSum(node.right, remaining);
}
```

**Complexity:** time O(n) · space O(height)

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor

#### 18.6 Lowest common ancestor

- **Big pattern:** Trees
- **Ask yourself:** *Where do two targets diverge?*
- **Small recipe:** `Recursive split / BST ordering`
- **Memory sentence:** Trees → Lowest common ancestor → Recursive split / BST ordering
- **Words that give it away:** root, subtree, height, path, BST

**In plain English:** Ask each subtree "did you find p or q?". If both sides say yes, this node is where they split — the lowest common ancestor. Otherwise pass up whichever side found something.

**Steps:**
1. null, p or q → return node.
2. left = lca(left), right = lca(right).
3. Both non-null → return node; else return the non-null one.

**Tiny example:**

```text
p = 5, q = 1 under root 3
left finds 5, right finds 1 → answer 3
```

**Java:**

```java
TreeNode lowestCommonAncestor(TreeNode node, TreeNode p, TreeNode q) {
    if (node == null || node == p || node == q) return node;
    TreeNode left = lowestCommonAncestor(node.left, p, q);
    TreeNode right = lowestCommonAncestor(node.right, p, q);
    if (left != null && right != null) return node;   // p and q split here
    return left != null ? left : right;
}
```

**Complexity:** time O(n) · space O(height)

**Practise on:** Maximum Depth · Validate BST · Lowest Common Ancestor

---

### 19. Trie (Prefix Tree)

**The big idea:** A trie stores words letter by letter, so words that start the same share the same path. Checking a word or a prefix takes time proportional to its length only.

**Think of it like this:** A dictionary’s thumb index: to find "car" you go to C, then A, then R — words starting with "ca" all live on that same path.

**How to spot it:**
- Prefix search / autocomplete
- Many words sharing prefixes
- Word search on a board with a dictionary

**Template:**

```java
class TrieNode {
    TrieNode[] next = new TrieNode[26];
    boolean isWord;
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Insert word](#191-insert-word) | Can shared prefixes share the same path? | `Create child per character` | O(word length) |
| [Exact search](#192-exact-search) | Did I reach a complete stored word? | `Walk + end marker` | O(word length) |
| [Prefix search](#193-prefix-search) | Does any stored word begin with this prefix? | `Walk prefix only` | O(prefix length) |
| [Autocomplete](#194-autocomplete) | After reaching prefix, what words live below it? | `Prefix node + DFS descendants` | O(prefix + size of the subtree) |
| [Word Search II](#195-word-search-ii) | Can prefix pruning avoid useless grid paths? | `Trie + grid DFS` | O(cells · 4 · 3^(L−1)) worst case, much less with pruning |

#### 19.1 Insert word

- **Big pattern:** Trie (Prefix Tree)
- **Ask yourself:** *Can shared prefixes share the same path?*
- **Small recipe:** `Create child per character`
- **Memory sentence:** Trie → Trie Insert → follow/create child per char
- **Words that give it away:** prefix, dictionary, autocomplete, word search

**In plain English:** Walk from the root letter by letter, creating a child node whenever it is missing. Mark the last node as the end of a word.

**Steps:**
1. cur = root.
2. For each char: create next[c] if null; move there.
3. cur.isWord = true.

**Tiny example:**

```text
insert "cat", then "car"
"car" reuses c → a and adds r
```

**Java:**

```java
class Trie {
    private final TrieNode root = new TrieNode();

    void insert(String word) {
        TrieNode cur = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (cur.next[i] == null) cur.next[i] = new TrieNode();
            cur = cur.next[i];
        }
        cur.isWord = true;
    }
}
```

**Complexity:** time O(word length) · space O(word length) new nodes at most

> ⚠️ **Trap:** Prefix existence and full-word existence are different.

**Practise on:** Implement Trie · Word Search II

#### 19.2 Exact search

- **Big pattern:** Trie (Prefix Tree)
- **Ask yourself:** *Did I reach a complete stored word?*
- **Small recipe:** `Walk + end marker`
- **Memory sentence:** Trie → Trie Search → missing child → false
- **Words that give it away:** prefix, dictionary, autocomplete, word search

**In plain English:** Follow the letters. A missing child means the word is not there. Reaching the end is not enough — the last node must be marked as a word.

**Steps:**
1. Walk the letters; missing child → false.
2. At the end return node.isWord.

**Tiny example:**

```text
trie has "cat"
search("ca") → path exists but isWord is false → false
```

**Java:**

```java
boolean search(TrieNode root, String word) {
    TrieNode cur = root;
    for (char c : word.toCharArray()) {
        cur = cur.next[c - 'a'];
        if (cur == null) return false;
    }
    return cur.isWord;
}
```

**Complexity:** time O(word length) · space O(1)

> ⚠️ **Trap:** startsWith would return true without requiring isWord.

**Practise on:** Implement Trie · Word Search II · Exact word search

#### 19.3 Prefix search

- **Big pattern:** Trie (Prefix Tree)
- **Ask yourself:** *Does any stored word begin with this prefix?*
- **Small recipe:** `Walk prefix only`
- **Memory sentence:** Trie → Prefix search → Walk prefix only
- **Words that give it away:** prefix, dictionary, autocomplete, word search

**In plain English:** Same walk as search, but as soon as the whole prefix path exists, the answer is true — no end marker needed.

**Steps:**
1. Walk the prefix letters.
2. Missing child → false.
3. Finished walking → true.

**Tiny example:**

```text
trie has "cat"
startsWith("ca") → true
```

**Java:**

```java
boolean startsWith(TrieNode root, String prefix) {
    TrieNode cur = root;
    for (char c : prefix.toCharArray()) {
        cur = cur.next[c - 'a'];
        if (cur == null) return false;
    }
    return true;
}
```

**Complexity:** time O(prefix length) · space O(1)

**Practise on:** Implement Trie · Word Search II

#### 19.4 Autocomplete

- **Big pattern:** Trie (Prefix Tree)
- **Ask yourself:** *After reaching prefix, what words live below it?*
- **Small recipe:** `Prefix node + DFS descendants`
- **Memory sentence:** Trie → Autocomplete → Prefix node + DFS descendants
- **Words that give it away:** prefix, dictionary, autocomplete, word search

**In plain English:** Walk to the node for the prefix, then explore everything below it (DFS), collecting each word you pass.

**Steps:**
1. Walk the prefix; if it breaks → no suggestions.
2. DFS from that node with a StringBuilder.
3. Whenever node.isWord → add the current string.

**Tiny example:**

```text
words cat, car, cart, dog; prefix "ca"
suggestions: car, cart, cat
```

**Java:**

```java
List<String> autocomplete(TrieNode root, String prefix) {
    List<String> out = new ArrayList<>();
    TrieNode cur = root;
    for (char c : prefix.toCharArray()) {
        cur = cur.next[c - 'a'];
        if (cur == null) return out;
    }
    collect(cur, new StringBuilder(prefix), out);
    return out;
}

void collect(TrieNode node, StringBuilder path, List<String> out) {
    if (node.isWord) out.add(path.toString());
    for (int i = 0; i < 26; i++) {
        if (node.next[i] == null) continue;
        path.append((char) ('a' + i));
        collect(node.next[i], path, out);
        path.deleteCharAt(path.length() - 1);
    }
}
```

**Complexity:** time O(prefix + size of the subtree) · space O(longest word)

**Practise on:** Implement Trie · Word Search II

#### 19.5 Word Search II

- **Big pattern:** Trie (Prefix Tree)
- **Ask yourself:** *Can prefix pruning avoid useless grid paths?*
- **Small recipe:** `Trie + grid DFS`
- **Memory sentence:** Trie → Word Search II → Trie + grid DFS
- **Words that give it away:** prefix, dictionary, autocomplete, word search

**In plain English:** Put all dictionary words in a trie. DFS over the grid, but move into a neighbouring cell only if the trie has a child for that letter — dead prefixes are pruned immediately.

**Steps:**
1. Build the trie; store the full word at its last node.
2. DFS from every cell, following trie children.
3. When a node holds a word → add it and clear it (avoid duplicates).

**Tiny example:**

```text
board with "oath" path, words [oath, pea]
DFS stops early on "pe…" when no path exists → finds only "oath"
```

**Java:**

```java
List<String> findWords(char[][] board, String[] words) {
    WordNode root = new WordNode();
    for (String w : words) {
        WordNode cur = root;
        for (char c : w.toCharArray()) {
            if (cur.next[c - 'a'] == null) cur.next[c - 'a'] = new WordNode();
            cur = cur.next[c - 'a'];
        }
        cur.word = w;
    }
    List<String> found = new ArrayList<>();
    for (int r = 0; r < board.length; r++)
        for (int c = 0; c < board[0].length; c++) dfs(board, r, c, root, found);
    return found;
}

void dfs(char[][] b, int r, int c, WordNode node, List<String> found) {
    if (r < 0 || c < 0 || r >= b.length || c >= b[0].length || b[r][c] == '#') return;
    WordNode next = node.next[b[r][c] - 'a'];
    if (next == null) return;                       // prune: no word continues here
    if (next.word != null) { found.add(next.word); next.word = null; }
    char saved = b[r][c];
    b[r][c] = '#';                                   // mark visited
    dfs(b, r + 1, c, next, found);
    dfs(b, r - 1, c, next, found);
    dfs(b, r, c + 1, next, found);
    dfs(b, r, c - 1, next, found);
    b[r][c] = saved;                                 // undo
}

static class WordNode {
    WordNode[] next = new WordNode[26];
    String word;
}
```

**Complexity:** time O(cells · 4 · 3^(L−1)) worst case, much less with pruning · space O(total letters in words)

**Practise on:** Implement Trie · Word Search II

**Runnable in SkillForge (Trie (Prefix Tree)):** Count Words With Prefix (Medium)

---

## Phase 5 — Graphs

### 20. BFS / DFS

**The big idea:** Two ways to explore anything connected (graphs, grids, trees). DFS goes deep down one path before backing up; BFS spreads out level by level. Always keep a "visited" mark so you never loop forever.

**Think of it like this:** Exploring a maze: DFS follows one corridor to its end before turning back; BFS sends explorers one step in every direction at once.

**How to spot it:**
- Grid of cells / islands
- "Can I reach…"
- Count separate groups
- Fewest steps when every step costs the same (BFS)

**Template:**

```java
int[][] dirs = {{1, 0}, {-1, 0}, {0, 1}, {0, -1}};
void dfs(char[][] g, int r, int c) {
    if (r < 0 || c < 0 || r >= g.length || c >= g[0].length || g[r][c] != '1') return;
    g[r][c] = '#';                       // mark visited
    for (int[] d : dirs) dfs(g, r + d[0], c + d[1]);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Reachability](#201-reachability) | Can I mark everything reachable from a start? | `DFS or BFS + visited` | O(V + E) |
| [Shortest unweighted path](#202-shortest-unweighted-path) | Does each edge cost the same? | `BFS` | O(V + E) |
| [Connected components](#203-connected-components) | How many separate groups exist? | `Traversal from every unvisited node` | O(V + E) |
| [Flood fill / islands](#204-flood-fill--islands) | Can I treat neighboring cells as graph edges? | `Grid DFS/BFS` | O(rows · cols) |
| [Tree levels](#205-tree-levels) | Do I need one depth at a time? | `BFS by queue size` | O(n) |

#### 20.1 Reachability

- **Big pattern:** BFS / DFS
- **Ask yourself:** *Can I mark everything reachable from a start?*
- **Small recipe:** `DFS or BFS + visited`
- **Memory sentence:** BFS/DFS → Graph DFS → visit → mark → recurse neighbors
- **Words that give it away:** reachability, islands, connected, shortest unweighted

**In plain English:** Start at the source and visit every neighbour you have not seen yet, marking each one. When the search stops, everything marked is reachable.

**Steps:**
1. visited[start] = true.
2. DFS: for each unvisited neighbour → mark and recurse.
3. Target reachable ⇔ visited[target].

**Tiny example:**

```text
edges 0-1, 1-2, 3-4
from 0 reach {0, 1, 2}; 4 is not reachable
```

**Java:**

```java
boolean canReach(List<List<Integer>> graph, int start, int target) {
    boolean[] visited = new boolean[graph.size()];
    dfs(graph, start, visited);
    return visited[target];
}

void dfs(List<List<Integer>> graph, int node, boolean[] visited) {
    visited[node] = true;
    for (int next : graph.get(node)) {
        if (!visited[next]) dfs(graph, next, visited);
    }
}
```

**Complexity:** time O(V + E) · space O(V)

> ⚠️ **Trap:** Visited prevents infinite loops.

**Practise on:** Number of Islands · Clone Graph · Rotting Oranges · Reachability / components

#### 20.2 Shortest unweighted path

- **Big pattern:** BFS / DFS
- **Ask yourself:** *Does each edge cost the same?*
- **Small recipe:** `BFS`
- **Memory sentence:** BFS/DFS → Graph BFS → mark when enqueuing
- **Words that give it away:** reachability, islands, connected, shortest unweighted

**In plain English:** When every edge costs the same, BFS reaches each node for the first time using the fewest edges. Mark nodes when you add them to the queue.

**Steps:**
1. dist[start] = 0; queue = [start].
2. Poll; for each unvisited neighbour set dist + 1 and enqueue.
3. The first time you reach the target is the shortest.

**Tiny example:**

```text
0 → 1 → 3 and 0 → 2 → 3
dist(3) = 2
```

**Java:**

```java
int shortestPath(List<List<Integer>> graph, int start, int target) {
    int[] dist = new int[graph.size()];
    Arrays.fill(dist, -1);
    Deque<Integer> queue = new ArrayDeque<>();
    dist[start] = 0;
    queue.offer(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        if (node == target) return dist[node];
        for (int next : graph.get(node)) {
            if (dist[next] == -1) {           // mark on enqueue
                dist[next] = dist[node] + 1;
                queue.offer(next);
            }
        }
    }
    return -1;
}
```

**Complexity:** time O(V + E) · space O(V)

> ⚠️ **Trap:** Mark visited when enqueueing to avoid duplicate queue entries.

**Practise on:** Number of Islands · Clone Graph · Rotting Oranges · Shortest path in unweighted graph

#### 20.3 Connected components

- **Big pattern:** BFS / DFS
- **Ask yourself:** *How many separate groups exist?*
- **Small recipe:** `Traversal from every unvisited node`
- **Memory sentence:** BFS/DFS → Connected components → Traversal from every unvisited node
- **Words that give it away:** reachability, islands, connected, shortest unweighted

**In plain English:** Loop over every node. Each time you find one that is still unvisited, you have found a NEW group — count it and mark its whole group with a DFS.

**Steps:**
1. count = 0.
2. For each node: if not visited → count++, DFS from it.
3. Return count.

**Tiny example:**

```text
5 nodes, edges 0-1, 1-2, 3-4
groups {0,1,2} and {3,4} → 2
```

**Java:**

```java
int countComponents(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        graph.get(e[1]).add(e[0]);
    }
    boolean[] visited = new boolean[n];
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            count++;
            mark(graph, i, visited);
        }
    }
    return count;
}

void mark(List<List<Integer>> graph, int node, boolean[] visited) {
    visited[node] = true;
    for (int next : graph.get(node)) if (!visited[next]) mark(graph, next, visited);
}
```

**Complexity:** time O(V + E) · space O(V + E)

**Practise on:** Number of Islands · Clone Graph · Rotting Oranges

#### 20.4 Flood fill / islands

- **Big pattern:** BFS / DFS
- **Ask yourself:** *Can I treat neighboring cells as graph edges?*
- **Small recipe:** `Grid DFS/BFS`
- **Memory sentence:** BFS/DFS → Grid DFS → 4 directions
- **Words that give it away:** reachability, islands, connected, shortest unweighted

**In plain English:** Treat each grid cell as a node connected to its 4 neighbours. Every time you find unvisited land, count an island and "sink" the whole island with DFS.

**Steps:**
1. Scan every cell.
2. Land found → islands++, DFS turns it and its connected land into water.
3. Check bounds before touching grid[r][c].

**Tiny example:**

```text
11000 / 11000 / 00100 / 00011
→ 3 islands
```

**Java:**

```java
int numIslands(char[][] grid) {
    int islands = 0;
    for (int r = 0; r < grid.length; r++) {
        for (int c = 0; c < grid[0].length; c++) {
            if (grid[r][c] == '1') {
                islands++;
                sink(grid, r, c);
            }
        }
    }
    return islands;
}

void sink(char[][] g, int r, int c) {
    if (r < 0 || c < 0 || r >= g.length || c >= g[0].length || g[r][c] != '1') return;
    g[r][c] = '0';
    sink(g, r + 1, c);
    sink(g, r - 1, c);
    sink(g, r, c + 1);
    sink(g, r, c - 1);
}
```

**Complexity:** time O(rows · cols) · space O(rows · cols) recursion in the worst case

> ⚠️ **Trap:** Mutating grid can replace a separate visited array if allowed.

**Practise on:** Number of Islands · Clone Graph · Rotting Oranges

#### 20.5 Tree levels

- **Big pattern:** BFS / DFS
- **Ask yourself:** *Do I need one depth at a time?*
- **Small recipe:** `BFS by queue size`
- **Memory sentence:** BFS/DFS → Tree levels → BFS by queue size
- **Words that give it away:** reachability, islands, connected, shortest unweighted

**In plain English:** BFS with the queue-size trick: the number of nodes in the queue at the start of a round is exactly one level. Great for "right side view", "average of each level", "minimum depth".

**Steps:**
1. queue = [root].
2. size = queue.size(); process size nodes.
3. The last node processed in a round is the right-side view.

**Tiny example:**

```text
1 / (2, 3) / (5 under 2, 4 under 3)
right side view: 1, 3, 4
```

**Java:**

```java
List<Integer> rightSideView(TreeNode root) {
    List<Integer> view = new ArrayList<>();
    if (root == null) return view;
    Deque<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (i == size - 1) view.add(node.val);   // last node of this level
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }
    return view;
}
```

**Complexity:** time O(n) · space O(width)

**Practise on:** Number of Islands · Clone Graph · Rotting Oranges

**Runnable in SkillForge (BFS / DFS):** Number of Islands (Medium) · Rotting Oranges (Multi-source BFS) (Medium)

---

### 21. Graphs

**The big idea:** A graph is a set of nodes connected by edges. Store it as an adjacency list, then choose the right tool: BFS/DFS to explore, Dijkstra for weighted shortest paths, Union-Find for grouping.

**Think of it like this:** A city map: places are nodes, roads are edges, and road lengths are weights.

**How to spot it:**
- Cities / flights / networks / friends
- Edges given as pairs
- Shortest or cheapest route
- Loops (cycles)

**Template:**

```java
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
for (int[] e : edges) {
    graph.get(e[0]).add(e[1]);
    graph.get(e[1]).add(e[0]);   // skip this line for a directed graph
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Adjacency list](#211-adjacency-list) | How should I store sparse edges? | `List of neighbors` | O(V + E) |
| [Undirected connectivity](#212-undirected-connectivity) | Do I need exploration or dynamic grouping? | `DFS/BFS or Union Find` | Almost O(1) per query |
| [Weighted shortest path](#213-weighted-shortest-path) | Are weights non-negative? | `Dijkstra` | O((V + E) log V) |
| [Negative edges](#214-negative-edges) | Can edges have negative weights? | `Bellman-Ford` | O(V · E) |
| [All-pairs shortest path](#215-all-pairs-shortest-path) | Do I need distance between every pair? | `Floyd-Warshall` | O(V³) |
| [Cycle detection](#216-cycle-detection) | Is there a loop? | `DFS colors / Union Find` | O(V + E) |

#### 21.1 Adjacency list

- **Big pattern:** Graphs
- **Ask yourself:** *How should I store sparse edges?*
- **Small recipe:** `List of neighbors`
- **Memory sentence:** Graphs → Adjacency List → edge → neighbor list
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Make one list per node and add each edge to the list of its start node (and the end node too if the graph is undirected).

**Steps:**
1. Create n empty lists.
2. For edge (u, v): graph[u].add(v).
3. Undirected → also graph[v].add(u).

**Tiny example:**

```text
edges [[0,1], [0,2], [1,3]]
graph[0] = [1, 2], graph[1] = [0, 3]
```

**Java:**

```java
List<List<Integer>> buildGraph(int n, int[][] edges, boolean directed) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        if (!directed) graph.get(e[1]).add(e[0]);
    }
    return graph;
}
```

**Complexity:** time O(V + E) · space O(V + E)

> ⚠️ **Trap:** Know whether graph is directed or undirected.

**Practise on:** Network Delay Time · Number of Connected Components · Graph representation

#### 21.2 Undirected connectivity

- **Big pattern:** Graphs
- **Ask yourself:** *Do I need exploration or dynamic grouping?*
- **Small recipe:** `DFS/BFS or Union Find`
- **Memory sentence:** Graphs → Undirected connectivity → DFS/BFS or Union Find
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Two good choices: explore with BFS/DFS when the graph is fixed, or use Union-Find when edges keep arriving and you keep asking "are these connected?".

**Steps:**
1. Static graph → BFS/DFS from each unvisited node.
2. Edges arrive over time → union(a, b) for each edge.
3. Connected? → find(a) == find(b).

**Tiny example:**

```text
friends (0,1), (1,2); is 0 connected to 2? → yes
```

**Java:**

```java
boolean connected(int n, int[][] edges, int a, int b) {
    int[] parent = new int[n];
    for (int i = 0; i < n; i++) parent[i] = i;
    for (int[] e : edges) parent[root(parent, e[0])] = root(parent, e[1]);
    return root(parent, a) == root(parent, b);
}

int root(int[] parent, int x) {
    while (parent[x] != x) {
        parent[x] = parent[parent[x]];   // path halving
        x = parent[x];
    }
    return x;
}
```

**Complexity:** time Almost O(1) per query · space O(V)

**Practise on:** Network Delay Time · Number of Connected Components

#### 21.3 Weighted shortest path

- **Big pattern:** Graphs
- **Ask yourself:** *Are weights non-negative?*
- **Small recipe:** `Dijkstra`
- **Memory sentence:** Graphs → Dijkstra → pop cheapest unprocessed state
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Dijkstra: keep a min-heap of (distance, node). Always take the closest node not yet finalised and try to improve its neighbours. Works only when no weight is negative.

**Steps:**
1. dist[] = ∞, dist[src] = 0, heap = [(0, src)].
2. Poll the smallest; skip it if it is outdated.
3. Relax edges: if dist[u] + w < dist[v] → update and push.

**Tiny example:**

```text
A–B 1, B–D 1, A–C 4, C–D 1
dist: A 0, B 1, D 2, C 3
```

**Java:**

```java
int[] dijkstra(List<List<int[]>> graph, int src) {   // graph.get(u) = list of {v, weight}
    int[] dist = new int[graph.size()];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));
    heap.offer(new int[]{0, src});
    while (!heap.isEmpty()) {
        int[] top = heap.poll();
        int d = top[0], u = top[1];
        if (d > dist[u]) continue;                  // stale entry
        for (int[] edge : graph.get(u)) {
            int v = edge[0], w = edge[1];
            if (d + w < dist[v]) {
                dist[v] = d + w;
                heap.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

**Complexity:** time O((V + E) log V) · space O(V + E)

> ⚠️ **Trap:** Dijkstra requires non-negative edge weights.

**Practise on:** Network Delay Time · Number of Connected Components · Weighted shortest path

#### 21.4 Negative edges

- **Big pattern:** Graphs
- **Ask yourself:** *Can edges have negative weights?*
- **Small recipe:** `Bellman-Ford`
- **Memory sentence:** Graphs → Negative edges → Bellman-Ford
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Bellman-Ford handles negative weights: "relax" every edge V − 1 times. If an edge can STILL be improved after that, there is a negative cycle.

**Steps:**
1. dist[src] = 0, others ∞.
2. Repeat V − 1 times: for every edge (u, v, w) try dist[u] + w.
3. One more pass improves something → negative cycle.

**Tiny example:**

```text
edges 0→1 (4), 0→2 (5), 2→1 (−3)
dist[1] improves from 4 to 2
```

**Java:**

```java
long[] bellmanFord(int n, int[][] edges, int src) {   // edge = {u, v, w}
    long INF = Long.MAX_VALUE / 4;
    long[] dist = new long[n];
    Arrays.fill(dist, INF);
    dist[src] = 0;
    for (int round = 0; round < n - 1; round++) {
        for (int[] e : edges) {
            if (dist[e[0]] != INF && dist[e[0]] + e[2] < dist[e[1]]) {
                dist[e[1]] = dist[e[0]] + e[2];
            }
        }
    }
    for (int[] e : edges) {
        if (dist[e[0]] != INF && dist[e[0]] + e[2] < dist[e[1]]) {
            throw new IllegalStateException("negative cycle");
        }
    }
    return dist;
}
```

**Complexity:** time O(V · E) · space O(V)

**Practise on:** Network Delay Time · Number of Connected Components

#### 21.5 All-pairs shortest path

- **Big pattern:** Graphs
- **Ask yourself:** *Do I need distance between every pair?*
- **Small recipe:** `Floyd-Warshall`
- **Memory sentence:** Graphs → All-pairs shortest path → Floyd-Warshall
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Floyd-Warshall: for every possible middle node k, check whether going i → k → j is shorter than the best known i → j. Three nested loops, k on the outside.

**Steps:**
1. dist[i][j] = edge weight, 0 on the diagonal, ∞ otherwise.
2. for k, for i, for j: dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]).
3. Read any pair instantly.

**Tiny example:**

```text
0→1 (3), 1→2 (1), 0→2 (10)
via k = 1: 0→2 becomes 4
```

**Java:**

```java
long[][] floydWarshall(long[][] dist) {   // dist[i][j] = weight or INF, 0 on the diagonal
    int n = dist.length;
    long INF = Long.MAX_VALUE / 4;
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            if (dist[i][k] >= INF) continue;
            for (int j = 0; j < n; j++) {
                if (dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }
    return dist;
}
```

**Complexity:** time O(V³) · space O(V²)

**Practise on:** Network Delay Time · Number of Connected Components

#### 21.6 Cycle detection

- **Big pattern:** Graphs
- **Ask yourself:** *Is there a loop?*
- **Small recipe:** `DFS colors / Union Find`
- **Memory sentence:** Graphs → Cycle detection → DFS colors / Union Find
- **Words that give it away:** nodes, edges, weights, connectivity, cycle

**In plain English:** Directed graph: DFS with three colours — white (new), grey (on the current path), black (finished). Reaching a grey node again means you went in a circle.

**Steps:**
1. colour[] = 0 (white).
2. Enter a node → grey. A grey neighbour → cycle!
3. Leave the node → black.

**Tiny example:**

```text
0 → 1 → 2 → 0
at 2 we see 0 still grey → cycle
```

**Java:**

```java
boolean hasCycle(List<List<Integer>> graph) {
    int[] colour = new int[graph.size()];      // 0 new, 1 on path, 2 done
    for (int i = 0; i < graph.size(); i++) {
        if (colour[i] == 0 && dfs(graph, i, colour)) return true;
    }
    return false;
}

boolean dfs(List<List<Integer>> graph, int u, int[] colour) {
    colour[u] = 1;
    for (int v : graph.get(u)) {
        if (colour[v] == 1) return true;                    // back edge → cycle
        if (colour[v] == 0 && dfs(graph, v, colour)) return true;
    }
    colour[u] = 2;
    return false;
}
```

**Complexity:** time O(V + E) · space O(V)

**Practise on:** Network Delay Time · Number of Connected Components

**Runnable in SkillForge (Graphs):** Network Delay Time (Medium)

---

### 22. Topological Sort

**The big idea:** Put tasks in an order where every task comes after the tasks it depends on. Repeatedly pick a task with no remaining prerequisites (in-degree 0).

**Think of it like this:** Getting dressed: socks before shoes, shirt before tie. Anything with nothing left to wait for can go next.

**How to spot it:**
- Prerequisites / dependencies
- Build or course order
- "Is it possible to finish all?" (cycle check)

**Template:**

```java
int[] indegree = new int[n];
for (int[] e : edges) { graph.get(e[0]).add(e[1]); indegree[e[1]]++; }
Deque<Integer> q = new ArrayDeque<>();
for (int i = 0; i < n; i++) if (indegree[i] == 0) q.offer(i);
while (!q.isEmpty()) {
    int u = q.poll();  order.add(u);
    for (int v : graph.get(u)) if (--indegree[v] == 0) q.offer(v);
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Course schedule possible?](#221-course-schedule-possible) | Can prerequisites be satisfied without a cycle? | `Kahn / DFS cycle detection` | O(V + E) |
| [Produce dependency order](#222-produce-dependency-order) | Which items have zero prerequisites now? | `Indegree queue` | O(V + E) |
| [Build order](#223-build-order) | What must happen before what? | `Topological ordering` | O(V + E) |
| [Cycle in directed graph](#224-cycle-in-directed-graph) | Did some nodes remain blocked forever? | `Processed count / DFS colors` | O(V + E) |

#### 22.1 Course schedule possible?

- **Big pattern:** Topological Sort
- **Ask yourself:** *Can prerequisites be satisfied without a cycle?*
- **Small recipe:** `Kahn / DFS cycle detection`
- **Memory sentence:** Topological Sort → Kahn's Algorithm → zero indegree → process → reduce neighbors
- **Words that give it away:** dependency, prerequisite, build order, course schedule

**In plain English:** Run Kahn’s algorithm and count how many courses get processed. If some never reach in-degree 0, they are stuck in a cycle → impossible.

**Steps:**
1. Edge b → a for "a needs b".
2. Process zero in-degree courses, reducing their neighbours.
3. possible ⇔ processed == n.

**Tiny example:**

```text
2 courses, 1 needs 0 → order 0, 1 → possible
0 needs 1 and 1 needs 0 → nothing starts → impossible
```

**Java:**

```java
boolean canFinish(int n, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    int[] indegree = new int[n];
    for (int[] p : prerequisites) {       // p = {course, prerequisite}
        graph.get(p[1]).add(p[0]);
        indegree[p[0]]++;
    }
    Deque<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);
    int processed = 0;
    while (!queue.isEmpty()) {
        int u = queue.poll();
        processed++;
        for (int v : graph.get(u)) if (--indegree[v] == 0) queue.offer(v);
    }
    return processed == n;
}
```

**Complexity:** time O(V + E) · space O(V + E)

> ⚠️ **Trap:** If processed < n, a cycle blocked some nodes.

**Practise on:** Course Schedule · Course Schedule II

#### 22.2 Produce dependency order

- **Big pattern:** Topological Sort
- **Ask yourself:** *Which items have zero prerequisites now?*
- **Small recipe:** `Indegree queue`
- **Memory sentence:** Topological Sort → Produce dependency order → Indegree queue
- **Words that give it away:** dependency, prerequisite, build order, course schedule

**In plain English:** Same algorithm, but record the order in which nodes leave the queue. That list is a valid order.

**Steps:**
1. Build graph + in-degrees.
2. Queue all zero in-degree nodes.
3. Poll → append to order → reduce neighbours.

**Tiny example:**

```text
4 courses: 1←0, 2←0, 3←1, 3←2
order 0, 1, 2, 3
```

**Java:**

```java
int[] findOrder(int n, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    int[] indegree = new int[n];
    for (int[] p : prerequisites) {
        graph.get(p[1]).add(p[0]);
        indegree[p[0]]++;
    }
    Deque<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);
    int[] order = new int[n];
    int idx = 0;
    while (!queue.isEmpty()) {
        int u = queue.poll();
        order[idx++] = u;
        for (int v : graph.get(u)) if (--indegree[v] == 0) queue.offer(v);
    }
    return idx == n ? order : new int[0];     // empty → there was a cycle
}
```

**Complexity:** time O(V + E) · space O(V + E)

**Practise on:** Course Schedule · Course Schedule II

#### 22.3 Build order

- **Big pattern:** Topological Sort
- **Ask yourself:** *What must happen before what?*
- **Small recipe:** `Topological ordering`
- **Memory sentence:** Topological Sort → Build order → Topological ordering
- **Words that give it away:** dependency, prerequisite, build order, course schedule

**In plain English:** Items often have names, not numbers. Use maps for the graph and in-degrees; otherwise the recipe is identical.

**Steps:**
1. Map<String, List<String>> graph; Map<String, Integer> indegree.
2. Start with every project whose in-degree is 0.
3. Process like Kahn’s algorithm.

**Tiny example:**

```text
projects a, b, c; deps (a → c), (b → c)
order a, b, c
```

**Java:**

```java
List<String> buildOrder(List<String> projects, List<String[]> deps) {   // dep = {before, after}
    Map<String, List<String>> graph = new HashMap<>();
    Map<String, Integer> indegree = new HashMap<>();
    for (String p : projects) {
        graph.put(p, new ArrayList<>());
        indegree.put(p, 0);
    }
    for (String[] d : deps) {
        graph.get(d[0]).add(d[1]);
        indegree.merge(d[1], 1, Integer::sum);
    }
    Deque<String> queue = new ArrayDeque<>();
    for (String p : projects) if (indegree.get(p) == 0) queue.offer(p);
    List<String> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        String p = queue.poll();
        order.add(p);
        for (String next : graph.get(p)) {
            if (indegree.merge(next, -1, Integer::sum) == 0) queue.offer(next);
        }
    }
    return order.size() == projects.size() ? order : Collections.emptyList();
}
```

**Complexity:** time O(V + E) · space O(V + E)

**Practise on:** Course Schedule · Course Schedule II

#### 22.4 Cycle in directed graph

- **Big pattern:** Topological Sort
- **Ask yourself:** *Did some nodes remain blocked forever?*
- **Small recipe:** `Processed count / DFS colors`
- **Memory sentence:** Topological Sort → Cycle in directed graph → Processed count / DFS colors
- **Words that give it away:** dependency, prerequisite, build order, course schedule

**In plain English:** Kahn’s algorithm doubles as a cycle detector: nodes on a cycle always keep in-degree ≥ 1, so they are never processed. processed < n ⇒ a cycle exists.

**Steps:**
1. Run Kahn’s algorithm.
2. Count processed nodes.
3. processed < n → cycle.

**Tiny example:**

```text
0 → 1 → 2 → 1
only 0 is processed → cycle between 1 and 2
```

**Java:**

```java
boolean hasDirectedCycle(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    int[] indegree = new int[n];
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        indegree[e[1]]++;
    }
    Deque<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);
    int processed = 0;
    while (!queue.isEmpty()) {
        int u = queue.poll();
        processed++;
        for (int v : graph.get(u)) if (--indegree[v] == 0) queue.offer(v);
    }
    return processed < n;
}
```

**Complexity:** time O(V + E) · space O(V + E)

**Practise on:** Course Schedule · Course Schedule II

**Runnable in SkillForge (Topological Sort):** Course Schedule (Medium) · Course Schedule II (Smallest Order) (Medium)

---

### 23. Union-Find (Disjoint Set)

**The big idea:** Each group has a leader (root). find(x) returns x’s leader; union(a, b) joins two groups by making one leader follow the other. With two small tricks, both are almost O(1).

**Think of it like this:** Friend circles: to know whether two people are in the same circle, compare who their circle leader is.

**How to spot it:**
- "Are these connected?" asked many times
- Groups merge over time
- Detect a redundant edge / cycle in an undirected graph
- Count groups

**Template:**

```java
int[] parent, size;
int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); }
boolean union(int a, int b) {
    a = find(a); b = find(b);
    if (a == b) return false;                 // already together
    if (size[a] < size[b]) { int t = a; a = b; b = t; }
    parent[b] = a; size[a] += size[b];
    return true;
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Connectivity query](#231-connectivity-query) | Do two items belong to the same group? | `find(a) == find(b)` | O(height) per find — see path compression |
| [Merge groups](#232-merge-groups) | Can I join two components quickly? | `union(a,b)` | O(height) |
| [Path compression](#233-path-compression) | Can future find operations become almost constant? | `Flatten find path` | Almost O(1) amortised (with union by size) |
| [Union by rank/size](#234-union-by-ranksize) | Can I keep trees shallow? | `Attach smaller tree under bigger` | Almost O(1) amortised |
| [Redundant edge](#235-redundant-edge) | Would this edge create a cycle? | `Already connected?` | Almost O(n) |
| [Count components](#236-count-components) | How many groups remain after merges? | `Start n, decrement on union` | Almost O(n + E) |

#### 23.1 Connectivity query

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *Do two items belong to the same group?*
- **Small recipe:** `find(a) == find(b)`
- **Memory sentence:** Union Find → Connectivity query → find(a) == find(b)
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** Two items are in the same group exactly when they have the same leader: find(a) == find(b).

**Steps:**
1. Build parent[] with unions.
2. connected(a, b) = find(a) == find(b).

**Tiny example:**

```text
after union(0,1), union(1,2)
connected(0, 2) → find(0) = find(2) → true
```

**Java:**

```java
class DSU {
    int[] parent;

    DSU(int n) {
        parent = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        while (parent[x] != x) x = parent[x];
        return x;
    }

    void union(int a, int b) {
        parent[find(a)] = find(b);
    }

    boolean connected(int a, int b) {
        return find(a) == find(b);
    }
}
```

**Complexity:** time O(height) per find — see path compression · space O(n)

**Practise on:** Redundant Connection · Number of Provinces

#### 23.2 Merge groups

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *Can I join two components quickly?*
- **Small recipe:** `union(a,b)`
- **Memory sentence:** Union Find → Merge groups → union(a,b)
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** To merge two groups, find both leaders and point one leader at the other. If the leaders are the same, they were already one group.

**Steps:**
1. ra = find(a), rb = find(b).
2. ra == rb → nothing to do.
3. Else parent[ra] = rb.

**Tiny example:**

```text
groups {0,1} and {2,3}: union(1, 3)
leader of 1 → 0, leader of 3 → 2 → parent[0] = 2
```

**Java:**

```java
boolean union(int[] parent, int a, int b) {
    int ra = find(parent, a), rb = find(parent, b);
    if (ra == rb) return false;     // already in the same group
    parent[ra] = rb;
    return true;
}

int find(int[] parent, int x) {
    while (parent[x] != x) x = parent[x];
    return x;
}
```

**Complexity:** time O(height) · space O(1)

**Practise on:** Redundant Connection · Number of Provinces

#### 23.3 Path compression

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *Can future find operations become almost constant?*
- **Small recipe:** `Flatten find path`
- **Memory sentence:** Union Find → Find with Path Compression → make nodes point closer to root
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** While finding the leader, make every node you pass point DIRECTLY to the leader. Later finds on those nodes become one step.

**Steps:**
1. Recursive: parent[x] = find(parent[x]).
2. Return parent[x].

**Tiny example:**

```text
chain 4 → 3 → 2 → 1 → 0
after find(4): 4, 3, 2, 1 all point to 0
```

**Java:**

```java
int find(int[] parent, int x) {
    if (parent[x] != x) {
        parent[x] = find(parent, parent[x]);   // flatten the path
    }
    return parent[x];
}
```

**Complexity:** time Almost O(1) amortised (with union by size) · space O(height) recursion

> ⚠️ **Trap:** Path compression makes repeated finds very fast.

**Practise on:** Redundant Connection · Number of Provinces · Connectivity

#### 23.4 Union by rank/size

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *Can I keep trees shallow?*
- **Small recipe:** `Attach smaller tree under bigger`
- **Memory sentence:** Union Find → Union by Size → smaller root joins larger root
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** Always attach the SMALLER group under the bigger one. Trees stay shallow, so finds stay fast.

**Steps:**
1. size[] starts at 1.
2. After finding both roots, make the smaller root point to the larger.
3. size[larger] += size[smaller].

**Tiny example:**

```text
size 5 group + size 2 group
the 2-group joins the 5-group → height barely grows
```

**Java:**

```java
boolean unionBySize(int[] parent, int[] size, int a, int b) {
    int ra = find(parent, a), rb = find(parent, b);
    if (ra == rb) return false;
    if (size[ra] < size[rb]) { int t = ra; ra = rb; rb = t; }
    parent[rb] = ra;              // smaller under larger
    size[ra] += size[rb];
    return true;
}

int find(int[] parent, int x) {
    return parent[x] == x ? x : (parent[x] = find(parent, parent[x]));
}
```

**Complexity:** time Almost O(1) amortised · space O(n)

> ⚠️ **Trap:** Returning false when roots already match is handy for cycle detection.

**Practise on:** Redundant Connection · Number of Provinces · Merge components

#### 23.5 Redundant edge

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *Would this edge create a cycle?*
- **Small recipe:** `Already connected?`
- **Memory sentence:** Union Find → Redundant edge → Already connected?
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** Add edges one by one. If an edge connects two nodes that already share a leader, it closes a loop — that edge is redundant.

**Steps:**
1. For each edge (u, v):
2. find(u) == find(v) → return this edge.
3. Else union(u, v).

**Tiny example:**

```text
[[1,2], [1,3], [2,3]]
[2,3]: 2 and 3 already connected via 1 → redundant
```

**Java:**

```java
int[] findRedundantConnection(int[][] edges) {
    int[] parent = new int[edges.length + 1];
    for (int i = 0; i < parent.length; i++) parent[i] = i;
    for (int[] e : edges) {
        int a = find(parent, e[0]), b = find(parent, e[1]);
        if (a == b) return e;       // already connected → this edge makes a cycle
        parent[a] = b;
    }
    return new int[0];
}

int find(int[] parent, int x) {
    return parent[x] == x ? x : (parent[x] = find(parent, parent[x]));
}
```

**Complexity:** time Almost O(n) · space O(n)

**Practise on:** Redundant Connection · Number of Provinces

#### 23.6 Count components

- **Big pattern:** Union-Find (Disjoint Set)
- **Ask yourself:** *How many groups remain after merges?*
- **Small recipe:** `Start n, decrement on union`
- **Memory sentence:** Union Find → Count components → Start n, decrement on union
- **Words that give it away:** components, connected?, merge groups, redundant edge

**In plain English:** Start with n groups. Every union that actually joins two different groups reduces the count by one.

**Steps:**
1. groups = n.
2. For each edge: if union succeeded → groups−−.
3. Return groups.

**Tiny example:**

```text
n = 5, edges (0,1), (1,2), (3,4)
5 → 4 → 3 → 2 groups
```

**Java:**

```java
int countGroups(int n, int[][] edges) {
    int[] parent = new int[n];
    for (int i = 0; i < n; i++) parent[i] = i;
    int groups = n;
    for (int[] e : edges) {
        int a = find(parent, e[0]), b = find(parent, e[1]);
        if (a != b) {
            parent[a] = b;
            groups--;
        }
    }
    return groups;
}

int find(int[] parent, int x) {
    return parent[x] == x ? x : (parent[x] = find(parent, parent[x]));
}
```

**Complexity:** time Almost O(n + E) · space O(n)

**Practise on:** Redundant Connection · Number of Provinces

**Runnable in SkillForge (Union-Find (Disjoint Set)):** Number of Connected Components (Medium) · Redundant Connection (Medium)

---

## Phase 6 — Recursion, DP & Bits

### 24. Backtracking

**The big idea:** Build an answer one choice at a time. After exploring a choice, UNDO it and try the next one. Stop exploring early when a partial answer can no longer work (pruning).

**Think of it like this:** Trying keys on a key ring: try one, if it fails take it out and try the next.

**How to spot it:**
- "Generate all…" subsets, permutations, combinations
- Puzzles: N-Queens, Sudoku
- Search a word in a grid
- Small input size (n ≤ 20)

**Template:**

```java
void backtrack(State s) {
    if (isComplete(s)) { save(copy of s); return; }
    for (Choice c : choices(s)) {
        if (!valid(s, c)) continue;   // prune
        apply(s, c);
        backtrack(s);
        undo(s, c);
    }
}
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Subsets](#241-subsets) | For each item, do I include it or not? | `Choose / skip` | O(n · 2^n) |
| [Permutations](#242-permutations) | Who goes in the next position? | `Choose unused item` | O(n · n!) |
| [Combination Sum](#243-combination-sum) | Can I build target while pruning impossible branches? | `Choose candidate repeatedly` | Exponential in target / smallest candidate |
| [N-Queens](#244-n-queens) | Can I safely place one queen in this row? | `Place → validate → recurse → remove` | O(n!) worst case |
| [Sudoku](#245-sudoku) | Which digit can legally go here? | `Fill → recurse → undo` | Exponential (9^empty cells) worst case |
| [Word Search](#246-word-search) | Can this path spell the word without reusing a cell? | `Grid DFS + temporary visited` | O(cells · 3^L) |

#### 24.1 Subsets

- **Big pattern:** Backtracking
- **Ask yourself:** *For each item, do I include it or not?*
- **Small recipe:** `Choose / skip`
- **Memory sentence:** Backtracking → Subsets → binary decision per item
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** Every node of the recursion is a subset. Save a copy of the current path, then try adding each remaining element (only elements after the last one used).

**Steps:**
1. result.add(new ArrayList<>(path)).
2. For i from start: add nums[i], recurse with i + 1, remove it.

**Tiny example:**

```text
[1, 2, 3]
[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]
```

**Java:**

```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    build(nums, 0, new ArrayList<>(), result);
    return result;
}

void build(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path));        // copy!
    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        build(nums, i + 1, path, result);
        path.remove(path.size() - 1);          // undo
    }
}
```

**Complexity:** time O(n · 2^n) · space O(n) recursion

> ⚠️ **Trap:** Copy path when saving; otherwise later mutations change saved answers.

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens

#### 24.2 Permutations

- **Big pattern:** Backtracking
- **Ask yourself:** *Who goes in the next position?*
- **Small recipe:** `Choose unused item`
- **Memory sentence:** Backtracking → Permutations → choose any unused item
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** Fill positions one by one. For the next position, try every element not used yet; mark it used, recurse, then unmark.

**Steps:**
1. If path is full → save a copy.
2. For each unused i: mark, add, recurse.
3. Remove and unmark.

**Tiny example:**

```text
[1, 2, 3]
123, 132, 213, 231, 312, 321
```

**Java:**

```java
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    build(nums, new boolean[nums.length], new ArrayList<>(), result);
    return result;
}

void build(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        path.add(nums[i]);
        build(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

**Complexity:** time O(n · n!) · space O(n)

> ⚠️ **Trap:** For duplicate input values, add sorting + duplicate skip logic.

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens

#### 24.3 Combination Sum

- **Big pattern:** Backtracking
- **Ask yourself:** *Can I build target while pruning impossible branches?*
- **Small recipe:** `Choose candidate repeatedly`
- **Memory sentence:** Backtracking → Choose / Explore / Undo → do → recurse → undo
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** Choose a candidate, subtract it from the remaining target, and recurse. Pass the SAME index again if a number may be reused. Stop when remaining hits 0 (success) or goes negative (prune).

**Steps:**
1. remaining == 0 → save path.
2. For i from start: skip if candidates[i] > remaining (sorted → break).
3. add, recurse with i (reuse allowed), remove.

**Tiny example:**

```text
candidates [2, 3, 6, 7], target 7
[2, 2, 3] and [7]
```

**Java:**

```java
List<List<Integer>> combinationSum(int[] candidates, int target) {
    Arrays.sort(candidates);
    List<List<Integer>> result = new ArrayList<>();
    build(candidates, 0, target, new ArrayList<>(), result);
    return result;
}

void build(int[] c, int start, int remaining, List<Integer> path, List<List<Integer>> result) {
    if (remaining == 0) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = start; i < c.length && c[i] <= remaining; i++) {   // prune
        path.add(c[i]);
        build(c, i, remaining - c[i], path, result);   // i, not i + 1 → reuse allowed
        path.remove(path.size() - 1);
    }
}
```

**Complexity:** time Exponential in target / smallest candidate · space O(target / smallest)

> ⚠️ **Trap:** If you mutate shared state, always undo before trying next choice.

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens · Universal backtracking template

#### 24.4 N-Queens

- **Big pattern:** Backtracking
- **Ask yourself:** *Can I safely place one queen in this row?*
- **Small recipe:** `Place → validate → recurse → remove`
- **Memory sentence:** Backtracking → N-Queens → Place → validate → recurse → remove
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** Place one queen per row. For each column check the column and both diagonals are free, place it, go to the next row, then remove it and try the next column.

**Steps:**
1. Row == n → one solution found.
2. Column c is safe if cols[c], diag[r − c + n], anti[r + c] are all free.
3. Mark, recurse to row + 1, unmark.

**Tiny example:**

```text
n = 4 → 2 solutions
.Q.. / ...Q / Q... / ..Q.
```

**Java:**

```java
int totalNQueens(int n) {
    return place(0, n, new boolean[n], new boolean[2 * n], new boolean[2 * n]);
}

int place(int row, int n, boolean[] cols, boolean[] diag, boolean[] anti) {
    if (row == n) return 1;
    int count = 0;
    for (int c = 0; c < n; c++) {
        if (cols[c] || diag[row - c + n] || anti[row + c]) continue;   // attacked
        cols[c] = diag[row - c + n] = anti[row + c] = true;
        count += place(row + 1, n, cols, diag, anti);
        cols[c] = diag[row - c + n] = anti[row + c] = false;          // undo
    }
    return count;
}
```

**Complexity:** time O(n!) worst case · space O(n)

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens

#### 24.5 Sudoku

- **Big pattern:** Backtracking
- **Ask yourself:** *Which digit can legally go here?*
- **Small recipe:** `Fill → recurse → undo`
- **Memory sentence:** Backtracking → Sudoku → Fill → recurse → undo
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** Find an empty cell, try digits 1–9 that do not clash with its row, column or 3×3 box. Recurse; if the rest cannot be solved, erase the digit and try the next.

**Steps:**
1. Find the next empty cell (none → solved).
2. For each legal digit: write it, recurse.
3. Recursion failed → erase it; no digit works → return false.

**Tiny example:**

```text
a cell where only 4 fits → 4
later a dead end → back up and change an earlier digit
```

**Java:**

```java
boolean solveSudoku(char[][] board) {
    for (int r = 0; r < 9; r++) {
        for (int c = 0; c < 9; c++) {
            if (board[r][c] != '.') continue;
            for (char d = '1'; d <= '9'; d++) {
                if (canPlace(board, r, c, d)) {
                    board[r][c] = d;
                    if (solveSudoku(board)) return true;
                    board[r][c] = '.';               // undo
                }
            }
            return false;                            // no digit fits here
        }
    }
    return true;                                     // no empty cell left
}

boolean canPlace(char[][] b, int r, int c, char d) {
    for (int i = 0; i < 9; i++) {
        if (b[r][i] == d || b[i][c] == d) return false;
        if (b[3 * (r / 3) + i / 3][3 * (c / 3) + i % 3] == d) return false;
    }
    return true;
}
```

**Complexity:** time Exponential (9^empty cells) worst case · space O(81) recursion

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens

#### 24.6 Word Search

- **Big pattern:** Backtracking
- **Ask yourself:** *Can this path spell the word without reusing a cell?*
- **Small recipe:** `Grid DFS + temporary visited`
- **Memory sentence:** Backtracking → Word Search → Grid DFS + temporary visited
- **Words that give it away:** all combinations, all permutations, choose and undo, search space

**In plain English:** From each cell, DFS in 4 directions matching the word letter by letter. Mark a cell as used while it is on the current path and restore it when you back up.

**Steps:**
1. Cell out of bounds or wrong letter → false.
2. Last letter matched → true.
3. Mark cell, try 4 neighbours for the next letter, restore the cell.

**Tiny example:**

```text
board ABCE / SFCS / ADEE, word "ABCCED"
A → B → C → C → E → D ✓
```

**Java:**

```java
boolean exist(char[][] board, String word) {
    for (int r = 0; r < board.length; r++)
        for (int c = 0; c < board[0].length; c++)
            if (dfs(board, word, 0, r, c)) return true;
    return false;
}

boolean dfs(char[][] b, String word, int i, int r, int c) {
    if (r < 0 || c < 0 || r >= b.length || c >= b[0].length || b[r][c] != word.charAt(i)) return false;
    if (i == word.length() - 1) return true;
    char saved = b[r][c];
    b[r][c] = '#';                                       // temporarily visited
    boolean found = dfs(b, word, i + 1, r + 1, c) || dfs(b, word, i + 1, r - 1, c)
                 || dfs(b, word, i + 1, r, c + 1) || dfs(b, word, i + 1, r, c - 1);
    b[r][c] = saved;                                     // restore
    return found;
}
```

**Complexity:** time O(cells · 3^L) · space O(L) recursion

**Practise on:** Subsets · Permutations · Combination Sum · N-Queens

**Runnable in SkillForge (Backtracking):** Subsets (Medium) · Permutations (Medium) · Letter Combinations of a Phone Number (Medium) · Generate Parentheses (Medium) · N-Queens (Count Solutions) (Hard)

---

### 25. Dynamic Programming

**The big idea:** Break a big problem into smaller versions of itself, solve each small version ONCE, and store the answer in a table. Bigger answers are built from smaller stored ones.

**Think of it like this:** Climbing stairs: the ways to reach step 10 = ways to reach step 9 + ways to reach step 8. Remember those numbers instead of recounting.

**How to spot it:**
- "Number of ways", "minimum cost", "maximum profit"
- Choices at each step that overlap
- Greedy fails on an example
- Two strings compared (LCS, edit distance)

**Template:**

```java
// 1. State:       dp[i] means ...
// 2. Transition:  dp[i] = combine(dp[i-1], dp[i-2], ...)
// 3. Base cases:  dp[0] = ...
// 4. Order:       fill small i before big i
// 5. Answer:      dp[n]
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [1D choose / skip](#251-1d-choose--skip) | Is today's best answer based on taking or skipping current item? | `dp[i] from earlier states` | O(n) |
| [Knapsack](#252-knapsack) | Do I choose this item under a capacity constraint? | `Item × capacity state` | O(n · W) |
| [Grid DP](#253-grid-dp) | Can each cell be solved from already solved nearby cells? | `Reuse top/left/neighbors` | O(rows · cols) |
| [LCS](#254-lcs) | Do two prefixes end with matching characters? | `2D prefix DP` | O(n · m) |
| [LIS](#255-lis) | What is the best increasing sequence ending here? | `DP or tails + binary search` | O(n²) |
| [Coin Change](#256-coin-change) | Can I build this amount from smaller solved amounts? | `Amount state` | O(amount · coins) |
| [Memoization](#257-memoization) | Am I solving the same state again? | `Recursive state + cache` | O(number of states) |

#### 25.1 1D choose / skip

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Is today's best answer based on taking or skipping current item?*
- **Small recipe:** `dp[i] from earlier states`
- **Memory sentence:** Dynamic Programming → 1D DP Choose / Skip → current = max(take, skip)
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** At each item decide: TAKE it (add its value to the best answer two steps back) or SKIP it (keep the best answer one step back). Keep the better of the two.

**Steps:**
1. take = prev2 + nums[i], skip = prev1.
2. cur = max(take, skip).
3. Shift: prev2 = prev1, prev1 = cur.

**Tiny example:**

```text
House Robber [2, 7, 9, 3, 1]
best: 2, 7, 11, 11, 12 → 12
```

**Java:**

```java
int rob(int[] nums) {
    int prev2 = 0, prev1 = 0;          // best up to i-2 and i-1
    for (int x : nums) {
        int cur = Math.max(prev1, prev2 + x);   // skip vs take
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Name states by meaning, not just dp[i].

**Practise on:** House Robber · Coin Change · Longest Common Subsequence

#### 25.2 Knapsack

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Do I choose this item under a capacity constraint?*
- **Small recipe:** `Item × capacity state`
- **Memory sentence:** Dynamic Programming → Knapsack → Item × capacity state
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** 0/1 knapsack: dp[c] = best value using capacity c. For each item, go through capacities from HIGH to LOW so the item is used at most once: dp[c] = max(dp[c], dp[c − w] + v).

**Steps:**
1. dp[0..W] = 0.
2. For each item (w, v): for c = W down to w.
3. dp[c] = max(dp[c], dp[c − w] + v).

**Tiny example:**

```text
weights [1, 3, 4], values [15, 20, 30], W = 4
best = 35 (items 1 and 3)
```

**Java:**

```java
int knapsack(int[] weight, int[] value, int capacity) {
    int[] dp = new int[capacity + 1];
    for (int i = 0; i < weight.length; i++) {
        for (int c = capacity; c >= weight[i]; c--) {   // downwards → each item once
            dp[c] = Math.max(dp[c], dp[c - weight[i]] + value[i]);
        }
    }
    return dp[capacity];
}
```

**Complexity:** time O(n · W) · space O(W)

**Practise on:** House Robber · Coin Change · Longest Common Subsequence

#### 25.3 Grid DP

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Can each cell be solved from already solved nearby cells?*
- **Small recipe:** `Reuse top/left/neighbors`
- **Memory sentence:** Dynamic Programming → Grid DP → cell answer from solved neighbors
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** Each cell is solved from cells already solved (above and to the left). Fill the first row and column, then the rest row by row.

**Steps:**
1. First row and column: only one way to reach them.
2. dp[r][c] = dp[r−1][c] + dp[r][c−1].
3. Answer at the bottom-right.

**Tiny example:**

```text
3 × 3 grid
row 3: 1, 3, 6 → 6 paths
```

**Java:**

```java
int uniquePaths(int rows, int cols) {
    int[][] dp = new int[rows][cols];
    for (int r = 0; r < rows; r++) dp[r][0] = 1;
    for (int c = 0; c < cols; c++) dp[0][c] = 1;
    for (int r = 1; r < rows; r++) {
        for (int c = 1; c < cols; c++) {
            dp[r][c] = dp[r - 1][c] + dp[r][c - 1];   // from above + from left
        }
    }
    return dp[rows - 1][cols - 1];
}
```

**Complexity:** time O(rows · cols) · space O(rows · cols)

> ⚠️ **Trap:** Initialize first row/column correctly.

**Practise on:** House Robber · Coin Change · Longest Common Subsequence · Unique Paths

#### 25.4 LCS

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Do two prefixes end with matching characters?*
- **Small recipe:** `2D prefix DP`
- **Memory sentence:** Dynamic Programming → LCS → 2D prefix DP
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** dp[i][j] = LCS of the first i letters of A and the first j letters of B. If those last letters match, extend the diagonal by 1; otherwise take the better of dropping a letter from A or from B.

**Steps:**
1. Table of size (n+1) × (m+1), row 0 and column 0 are 0.
2. Match → dp[i−1][j−1] + 1.
3. No match → max(dp[i−1][j], dp[i][j−1]).

**Tiny example:**

```text
"abcde" vs "ace"
a ✓, c ✓, e ✓ → 3
```

**Java:**

```java
int longestCommonSubsequence(String a, String b) {
    int n = a.length(), m = b.length();
    int[][] dp = new int[n + 1][m + 1];
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            if (a.charAt(i - 1) == b.charAt(j - 1)) dp[i][j] = dp[i - 1][j - 1] + 1;
            else dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
        }
    }
    return dp[n][m];
}
```

**Complexity:** time O(n · m) · space O(n · m)

**Practise on:** House Robber · Coin Change · Longest Common Subsequence

#### 25.5 LIS

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *What is the best increasing sequence ending here?*
- **Small recipe:** `DP or tails + binary search`
- **Memory sentence:** Dynamic Programming → LIS → DP or tails + binary search
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** Simple version: dp[i] = longest increasing run ENDING at i = 1 + best dp[j] for any earlier smaller value. (A faster O(n log n) version keeps "tails" and uses binary search.)

**Steps:**
1. dp[i] = 1 for all i.
2. For j < i with nums[j] < nums[i]: dp[i] = max(dp[i], dp[j] + 1).
3. Answer = max of dp.

**Tiny example:**

```text
[10, 9, 2, 5, 3, 7, 101, 18]
dp: 1, 1, 1, 2, 2, 3, 4, 4 → 4
```

**Java:**

```java
int lengthOfLIS(int[] nums) {
    int[] dp = new int[nums.length];
    int best = 0;
    for (int i = 0; i < nums.length; i++) {
        dp[i] = 1;
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) dp[i] = Math.max(dp[i], dp[j] + 1);
        }
        best = Math.max(best, dp[i]);
    }
    return best;
}
```

**Complexity:** time O(n²) · space O(n)

**Practise on:** House Robber · Coin Change · Longest Common Subsequence

#### 25.6 Coin Change

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Can I build this amount from smaller solved amounts?*
- **Small recipe:** `Amount state`
- **Memory sentence:** Dynamic Programming → Coin Change → Amount state
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** dp[x] = fewest coins to make amount x. Build from 0 upwards: for each coin that fits, dp[x] = min(dp[x], dp[x − coin] + 1).

**Steps:**
1. dp[0] = 0, others = amount + 1 (acts as infinity).
2. For x = 1..amount, for each coin ≤ x: try dp[x − coin] + 1.
3. dp[amount] > amount → impossible (−1).

**Tiny example:**

```text
coins [1, 2, 5], amount 11
dp[11] = dp[6] + 1 = 3 (5 + 5 + 1)
```

**Java:**

```java
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int x = 1; x <= amount; x++) {
        for (int coin : coins) {
            if (coin <= x) dp[x] = Math.min(dp[x], dp[x - coin] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

**Complexity:** time O(amount · coins) · space O(amount)

**Practise on:** House Robber · Coin Change · Longest Common Subsequence

#### 25.7 Memoization

- **Big pattern:** Dynamic Programming
- **Ask yourself:** *Am I solving the same state again?*
- **Small recipe:** `Recursive state + cache`
- **Memory sentence:** Dynamic Programming → Memoization → state → cached answer
- **Words that give it away:** number of ways, min/max cost, choose/skip, overlapping subproblems

**In plain English:** Write the natural recursive solution, then add a cache: before computing a state, check if you already know its answer. This turns exponential recursion into DP ("top-down").

**Steps:**
1. Recursive function of the state.
2. If memo has the state → return it.
3. Compute, store in memo, return.

**Tiny example:**

```text
fib(5) without memo calls fib(2) three times
with memo each fib(k) is computed once
```

**Java:**

```java
long climbStairs(int n) {
    return ways(n, new HashMap<>());
}

long ways(int n, Map<Integer, Long> memo) {
    if (n <= 1) return 1;
    if (memo.containsKey(n)) return memo.get(n);
    long result = ways(n - 1, memo) + ways(n - 2, memo);
    memo.put(n, result);
    return result;
}
```

**Complexity:** time O(number of states) · space O(number of states)

> ⚠️ **Trap:** Your memo key must include every variable that defines the state.

**Practise on:** House Robber · Coin Change · Longest Common Subsequence · Recursive DP

**Runnable in SkillForge (Dynamic Programming):** N-th Fibonacci (mod 1e9+7) (Easy) · Climbing Stairs (Easy) · House Robber (Medium) · Coin Change (Minimum Coins) (Medium) · Unique Paths (Medium) · Longest Increasing Subsequence (Medium) · Longest Common Subsequence (Medium) · 0/1 Knapsack (Medium) · Edit Distance (Hard) · Longest Palindromic Substring (Medium)

---

### 26. Bit Manipulation

**The big idea:** Numbers are stored as bits (0s and 1s). With AND (&), OR (|), XOR (^) and shifts (<<, >>) you can read, set, clear and flip individual bits in O(1).

**Think of it like this:** A row of light switches: a "mask" lets you look at or flip exactly the switch you want.

**How to spot it:**
- "Every number appears twice except one"
- Power of two
- Count set bits
- All subsets of a small set (n ≤ 20)

**Template:**

```java
boolean on  = (x & (1 << k)) != 0;  // check bit k
x = x |  (1 << k);                  // set
x = x & ~(1 << k);                  // clear
x = x ^  (1 << k);                  // toggle
```

**Question types in this pattern:**

| Question type | Ask yourself | Small recipe | Time |
|---|---|---|---|
| [Check bit](#261-check-bit) | Is bit k ON? | `x & (1<<k)` | O(1) |
| [Set bit](#262-set-bit) | How do I turn bit k ON? | `x \| (1<<k)` | O(1) |
| [Clear bit](#263-clear-bit) | How do I turn bit k OFF? | `x & ~(1<<k)` | O(1) |
| [Toggle bit](#264-toggle-bit) | How do I flip bit k? | `x ^ (1<<k)` | O(1) |
| [Single Number](#265-single-number) | Can pairs cancel each other? | `XOR everything` | O(n) |
| [Power of two](#266-power-of-two) | Does n have exactly one set bit? | `n & (n-1)` | O(1) |
| [Enumerate subsets](#267-enumerate-subsets) | Can each bit represent include/exclude? | `Bitmask 0..2^n-1` | O(n · 2^n) |

#### 26.1 Check bit

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *Is bit k ON?*
- **Small recipe:** `x & (1<<k)`
- **Memory sentence:** Bit Manipulation → Check Bit → x & (1<<k)
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** Make a mask with only bit k on (1 << k). AND keeps just that bit: non-zero means it was on.

**Steps:**
1. mask = 1 << k.
2. (x & mask) != 0 → bit is on.
3. Bits are numbered from 0 on the right.

**Tiny example:**

```text
x = 13 (1101), k = 2
1101 & 0100 = 0100 ≠ 0 → on
```

**Java:**

```java
boolean isBitOn(int x, int k) {
    return (x & (1 << k)) != 0;
}
```

**Complexity:** time O(1) · space O(1)

> ⚠️ **Trap:** Bit positions are zero-based.

**Practise on:** Single Number · Counting Bits · Power of Two · Check bit k

#### 26.2 Set bit

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *How do I turn bit k ON?*
- **Small recipe:** `x | (1<<k)`
- **Memory sentence:** Bit Manipulation → Set Bit → x | (1<<k)
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** OR with the mask turns bit k on and leaves every other bit unchanged.

**Steps:**
1. mask = 1 << k.
2. x | mask.

**Tiny example:**

```text
x = 9 (1001), k = 1
1001 | 0010 = 1011 = 11
```

**Java:**

```java
int setBit(int x, int k) {
    return x | (1 << k);
}
```

**Complexity:** time O(1) · space O(1)

> ⚠️ **Trap:** OR preserves existing 1s.

**Practise on:** Single Number · Counting Bits · Power of Two · Turn bit on

#### 26.3 Clear bit

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *How do I turn bit k OFF?*
- **Small recipe:** `x & ~(1<<k)`
- **Memory sentence:** Bit Manipulation → Clear Bit → x & ~(1<<k)
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** Invert the mask (~) so it has a 0 only at bit k, then AND: bit k becomes 0, the rest stay.

**Steps:**
1. mask = ~(1 << k).
2. x & mask.

**Tiny example:**

```text
x = 13 (1101), k = 2
1101 & 1011 = 1001 = 9
```

**Java:**

```java
int clearBit(int x, int k) {
    return x & ~(1 << k);
}
```

**Complexity:** time O(1) · space O(1)

> ⚠️ **Trap:** Parentheses matter around 1 << k.

**Practise on:** Single Number · Counting Bits · Power of Two · Turn bit off

#### 26.4 Toggle bit

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *How do I flip bit k?*
- **Small recipe:** `x ^ (1<<k)`
- **Memory sentence:** Bit Manipulation → Toggle Bit → x ^ (1<<k)
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** XOR with the mask flips bit k: 1 becomes 0 and 0 becomes 1.

**Steps:**
1. mask = 1 << k.
2. x ^ mask.

**Tiny example:**

```text
x = 13 (1101), k = 0
1101 ^ 0001 = 1100 = 12
```

**Java:**

```java
int toggleBit(int x, int k) {
    return x ^ (1 << k);
}
```

**Complexity:** time O(1) · space O(1)

> ⚠️ **Trap:** XOR with 1 flips; XOR with 0 keeps.

**Practise on:** Single Number · Counting Bits · Power of Two · Flip bit

#### 26.5 Single Number

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *Can pairs cancel each other?*
- **Small recipe:** `XOR everything`
- **Memory sentence:** Bit Manipulation → Single Number → pairs disappear
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** XOR a number with itself gives 0, and XOR with 0 changes nothing. XOR everything together: all the pairs cancel and the single number is left.

**Steps:**
1. x = 0.
2. x ^= every number.
3. Return x.

**Tiny example:**

```text
[4, 1, 2, 1, 2]
4 ^ 1 ^ 2 ^ 1 ^ 2 = 4
```

**Java:**

```java
int singleNumber(int[] nums) {
    int x = 0;
    for (int n : nums) x ^= n;
    return x;
}
```

**Complexity:** time O(n) · space O(1)

> ⚠️ **Trap:** Works because a^a=0 and XOR is associative/commutative.

**Practise on:** Single Number · Counting Bits · Power of Two

#### 26.6 Power of two

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *Does n have exactly one set bit?*
- **Small recipe:** `n & (n-1)`
- **Memory sentence:** Bit Manipulation → Power of two → n & (n-1)
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** A power of two has exactly ONE bit set. n − 1 flips that bit and turns on all bits below it, so n & (n − 1) is 0 only for powers of two.

**Steps:**
1. n must be > 0.
2. Return (n & (n − 1)) == 0.

**Tiny example:**

```text
8 = 1000, 7 = 0111 → 1000 & 0111 = 0 ✓
6 = 110, 5 = 101 → 100 ≠ 0 ✗
```

**Java:**

```java
boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

**Complexity:** time O(1) · space O(1)

**Practise on:** Single Number · Counting Bits · Power of Two

#### 26.7 Enumerate subsets

- **Big pattern:** Bit Manipulation
- **Ask yourself:** *Can each bit represent include/exclude?*
- **Small recipe:** `Bitmask 0..2^n-1`
- **Memory sentence:** Bit Manipulation → Enumerate subsets → Bitmask 0..2^n-1
- **Words that give it away:** XOR, bit, mask, power of two, subset mask

**In plain English:** With n items, every number from 0 to 2ⁿ − 1 is a subset: bit i of the number says whether item i is included.

**Steps:**
1. for mask = 0 .. (1 << n) − 1.
2. For each i: if bit i of mask is on → include nums[i].
3. Each mask gives one subset.

**Tiny example:**

```text
[a, b, c]: mask 101 → {a, c}
mask 000 → {}, 111 → {a, b, c}
```

**Java:**

```java
List<List<Integer>> allSubsets(int[] nums) {
    int n = nums.length;
    List<List<Integer>> result = new ArrayList<>();
    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) subset.add(nums[i]);
        }
        result.add(subset);
    }
    return result;
}
```

**Complexity:** time O(n · 2^n) · space O(n · 2^n) for the output

**Practise on:** Single Number · Counting Bits · Power of Two

**Runnable in SkillForge (Bit Manipulation):** Single Number (Easy) · Counting Bits (Easy)

---
