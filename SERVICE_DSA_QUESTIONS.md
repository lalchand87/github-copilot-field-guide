# Service-Based Company DSA Questions

> 384 questions asked at service-based companies (TCS, Infosys, Wipro, Accenture, Cognizant, Capgemini, HCL, Tech Mahindra and more), with LeetCode numbers and links. 44 of them have a full Java version with tests in SkillForge (**Service DSA** tab) — their statements, approaches and Java solutions are in [Part 3](#part-3--the-44-questions-solved-in-java). Exported from the app on 2026-10-05.

**How to use it:** start with the highest *frequency* questions in each difficulty (they come up most often), solve them in Java, and use Part 2 to focus on the company you are interviewing with.

| | Easy | Medium | Hard | Total |
|---|---|---|---|---|
| **DSA** | 127 | 172 | 57 | 356 |
| **SQL** | 17 | 7 | 1 | 25 |
| **JavaScript** | 3 | 0 | 0 | 3 |
| **All** | 147 | 179 | 58 | 384 |

Columns: **#** = LeetCode number · **Freq** = how often it is asked (100 = most) · **Acc** = LeetCode acceptance rate · **Java** = ✅ runnable with tests in SkillForge.

## Contents

- [Part 1 — All questions](#part-1--all-questions)
  - [DSA · Easy](#dsa-easy)
  - [DSA · Medium](#dsa-medium)
  - [DSA · Hard](#dsa-hard)
  - [SQL · Easy](#sql-easy)
  - [SQL · Medium](#sql-medium)
  - [SQL · Hard](#sql-hard)
  - [JavaScript · Easy](#javascript-easy)
- [Part 2 — By company](#part-2--by-company)
- [Part 3 — The 44 questions solved in Java](#part-3--the-44-questions-solved-in-java)

## Part 1 — All questions

### DSA Easy

127 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum) | 100 | 57.5% | Accenture, Capgemini, Infosys, Wipro, HCL, Tech Mahindra, Cognizant, TCS, Accolite, Persistent Systems, Altimetrik, Mindtree |  |
| 121 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) | 100 | 56.8% | Tech Mahindra, Accenture, Infosys, HCL, Capgemini, Cognizant, TCS, Accolite | [✅](#121-best-time-to-buy-and-sell-stock) |
| 283 | [Move Zeroes](https://leetcode.com/problems/move-zeroes) | 100 | 63.8% | LTI, ADP, TCS, Accenture, Capgemini, Infosys, Accolite, Cognizant |  |
| 2180 | [Count Integers With Even Digit Sum](https://leetcode.com/problems/count-integers-with-even-digit-sum) | 100 | 70.3% | Mindtree |  |
| 2220 | [Minimum Bit Flips to Convert Number](https://leetcode.com/problems/minimum-bit-flips-to-convert-number) | 100 | 87.9% | Persistent Systems |  |
| 2341 | [Maximum Number of Pairs in Array](https://leetcode.com/problems/maximum-number-of-pairs-in-array) | 100 | 76.1% | Altimetrik |  |
| 3079 | [Find the Sum of Encrypted Integers](https://leetcode.com/problems/find-the-sum-of-encrypted-integers) | 100 | 74.9% | Larsen & Toubro |  |
| 3105 | [Longest Strictly Increasing or Strictly Decreasing Subarray](https://leetcode.com/problems/longest-strictly-increasing-or-strictly-decreasing-subarray) | 100 | 64.9% | Larsen & Toubro |  |
| 3392 | [Count Subarrays of Length Three With a Condition](https://leetcode.com/problems/count-subarrays-of-length-three-with-a-condition) | 100 | 61.4% | Cognizant |  |
| 3875 | [Construct Uniform Parity Array I](https://leetcode.com/problems/construct-uniform-parity-array-i) | 100 | 76.3% | Amdocs |  |
| 9 | [Palindrome Number](https://leetcode.com/problems/palindrome-number) | 87.5 | 60.6% | Cognizant, Accenture, TCS, Infosys, Capgemini, Wipro, Persistent Systems, Mindtree | [✅](#9-palindrome-number) |
| 14 | [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) | 87.5 | 47.6% | Wipro, Infosys, Persistent Systems, Accenture, Capgemini, TCS | [✅](#14-longest-common-prefix) |
| 20 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses) | 87.5 | 44.2% | HCL, Infosys, Wipro, Persistent Systems, Accenture, Altimetrik, Cognizant, TCS |  |
| 70 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs) | 87.5 | 54.1% | Accenture, Accolite, Cognizant, Infosys, TCS |  |
| 125 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome) | 87.5 | 53.3% | LTI, HCL, Cognizant, TCS, Accenture, Infosys |  |
| 509 | [Fibonacci Number](https://leetcode.com/problems/fibonacci-number) | 87.5 | 74.1% | LTI, Cognizant, Accenture, Infosys, TCS |  |
| 2859 | [Sum of Values at Indices With K Set Bits](https://leetcode.com/problems/sum-of-values-at-indices-with-k-set-bits) | 87.5 | 86.1% | Accenture |  |
| 3000 | [Maximum Area of Longest Diagonal Rectangle](https://leetcode.com/problems/maximum-area-of-longest-diagonal-rectangle) | 87.5 | 45.9% | Accenture |  |
| 3146 | [Permutation Difference between Two Strings](https://leetcode.com/problems/permutation-difference-between-two-strings) | 87.5 | 87.8% | Accenture |  |
| 3498 | [Reverse Degree of a String](https://leetcode.com/problems/reverse-degree-of-a-string) | 87.5 | 88.7% | Capgemini |  |
| 88 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) | 75 | 54.9% | Cognizant, HCL, Accenture, Wipro, Persistent Systems, TCS, Infosys | [✅](#88-merge-sorted-array) |
| 169 | [Majority Element](https://leetcode.com/problems/majority-element) | 75 | 66.3% | Accenture, TCS, Cognizant, Infosys | [✅](#169-majority-element) |
| 202 | [Happy Number](https://leetcode.com/problems/happy-number) | 75 | 59.7% | Accenture, TCS |  |
| 242 | [Valid Anagram](https://leetcode.com/problems/valid-anagram) | 75 | 68.1% | Wipro, Accenture, Capgemini, Cognizant, TCS, Infosys |  |
| 2099 | [Find Subsequence of Length K With the Largest Sum](https://leetcode.com/problems/find-subsequence-of-length-k-with-the-largest-sum) | 75 | 57.4% | Accenture, TCS |  |
| 2418 | [Sort the People](https://leetcode.com/problems/sort-the-people) | 75 | 84.8% | Infosys |  |
| 2855 | [Minimum Right Shifts to Sort the Array](https://leetcode.com/problems/minimum-right-shifts-to-sort-the-array) | 75 | 57.4% | Accenture |  |
| 2960 | [Count Tested Devices After Test Operations](https://leetcode.com/problems/count-tested-devices-after-test-operations) | 75 | 78.9% | Accenture |  |
| 3005 | [Count Elements With Maximum Frequency](https://leetcode.com/problems/count-elements-with-maximum-frequency) | 75 | 79.8% | Capgemini |  |
| 3028 | [Ant on the Boundary](https://leetcode.com/problems/ant-on-the-boundary) | 75 | 74.4% | Accenture |  |
| 3345 | [Smallest Divisible Digit Product I](https://leetcode.com/problems/smallest-divisible-digit-product-i) | 75 | 64.4% | Accenture |  |
| 3467 | [Transform Array by Parity](https://leetcode.com/problems/transform-array-by-parity) | 75 | 89.8% | Infosys |  |
| 3667 | [Sort Array By Absolute Value](https://leetcode.com/problems/sort-array-by-absolute-value) | 75 | 86.5% | Cognizant |  |
| 13 | [Roman to Integer](https://leetcode.com/problems/roman-to-integer) | 62.5 | 66.6% | Accenture, Capgemini, Cognizant, TCS, Infosys | [✅](#13-roman-to-integer) |
| 21 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) | 62.5 | 68.3% | Capgemini, Infosys, Accenture, TCS | [✅](#21-merge-two-sorted-lists) |
| 26 | [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) | 62.5 | 62.8% | Accenture, Capgemini, Cognizant, TCS, Infosys | [✅](#26-remove-duplicates-from-sorted-array) |
| 28 | [Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) | 62.5 | 46.7% | Capgemini, TCS, Infosys |  |
| 66 | [Plus One](https://leetcode.com/problems/plus-one) | 62.5 | 50% | Accenture, Capgemini, TCS | [✅](#66-plus-one) |
| 67 | [Add Binary](https://leetcode.com/problems/add-binary) | 62.5 | 58.1% | Wipro, TCS, Infosys | [✅](#67-add-binary) |
| 118 | [Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) | 62.5 | 78.9% | Wipro, Accenture, TCS, Infosys | [✅](#118-pascals-triangle) |
| 136 | [Single Number](https://leetcode.com/problems/single-number) | 62.5 | 77.7% | Amdocs, TCS, Cognizant, Accenture |  |
| 217 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate) | 62.5 | 64.4% | Accenture, Capgemini, TCS, Infosys | [✅](#217-contains-duplicate) |
| 268 | [Missing Number](https://leetcode.com/problems/missing-number) | 62.5 | 72% | HCL, TCS |  |
| 344 | [Reverse String](https://leetcode.com/problems/reverse-string) | 62.5 | 80.8% | HCL, Accenture, TCS, Infosys |  |
| 349 | [Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) | 62.5 | 77.8% | Accenture, Capgemini, Infosys, TCS |  |
| 455 | [Assign Cookies](https://leetcode.com/problems/assign-cookies) | 62.5 | 55% | Accenture, TCS |  |
| 485 | [Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) | 62.5 | 65.2% | Accenture, Cognizant, TCS | [✅](#485-max-consecutive-ones) |
| 766 | [Toeplitz Matrix](https://leetcode.com/problems/toeplitz-matrix) | 62.5 | 69.7% | Wipro, TCS |  |
| 876 | [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list) | 62.5 | 81.8% | Accenture, TCS |  |
| 1518 | [Water Bottles](https://leetcode.com/problems/water-bottles) | 62.5 | 72.6% | Accenture, TCS |  |
| 1800 | [Maximum Ascending Subarray Sum](https://leetcode.com/problems/maximum-ascending-subarray-sum) | 62.5 | 66.3% | TCS |  |
| 2259 | [Remove Digit From Number to Maximize Result](https://leetcode.com/problems/remove-digit-from-number-to-maximize-result) | 62.5 | 48.5% | Infosys |  |
| 2423 | [Remove Letter To Equalize Frequency](https://leetcode.com/problems/remove-letter-to-equalize-frequency) | 62.5 | 19.4% | TCS |  |
| 2520 | [Count the Digits That Divide a Number](https://leetcode.com/problems/count-the-digits-that-divide-a-number) | 62.5 | 85.9% | TCS |  |
| 3432 | [Count Partitions with Even Sum Difference](https://leetcode.com/problems/count-partitions-with-even-sum-difference) | 62.5 | 85.3% | Accenture |  |
| 3637 | [Trionic Array I](https://leetcode.com/problems/trionic-array-i) | 62.5 | 49.5% | Infosys |  |
| 35 | [Search Insert Position](https://leetcode.com/problems/search-insert-position) | 50 | 51.3% | Cognizant, Accenture, TCS | [✅](#35-search-insert-position) |
| 141 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) | 50 | 54.3% | Cognizant, Infosys, Accenture, TCS | [✅](#141-linked-list-cycle) |
| 219 | [Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii) | 50 | 51.3% | Accenture, TCS |  |
| 303 | [Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable) | 50 | 72.2% | Infosys, TCS |  |
| 412 | [Fizz Buzz](https://leetcode.com/problems/fizz-buzz) | 50 | 75.5% | Cognizant, TCS |  |
| 496 | [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i) | 50 | 76.1% | Accenture, TCS |  |
| 704 | [Binary Search](https://leetcode.com/problems/binary-search) | 50 | 60.9% | Cognizant, Infosys, TCS |  |
| 1200 | [Minimum Absolute Difference](https://leetcode.com/problems/minimum-absolute-difference) | 50 | 75.1% | Cognizant |  |
| 1295 | [Find Numbers with Even Number of Digits](https://leetcode.com/problems/find-numbers-with-even-number-of-digits) | 50 | 79.8% | Cognizant |  |
| 1732 | [Find the Highest Altitude](https://leetcode.com/problems/find-the-highest-altitude) | 50 | 83.9% | Cognizant |  |
| 1752 | [Check if Array Is Sorted and Rotated](https://leetcode.com/problems/check-if-array-is-sorted-and-rotated) | 50 | 57.3% | TCS |  |
| 2481 | [Minimum Cuts to Divide a Circle](https://leetcode.com/problems/minimum-cuts-to-divide-a-circle) | 50 | 56.2% | TCS |  |
| 3065 | [Minimum Operations to Exceed Threshold Value I](https://leetcode.com/problems/minimum-operations-to-exceed-threshold-value-i) | 50 | 86.8% | TCS |  |
| 3417 | [Zigzag Grid Traversal With Skip](https://leetcode.com/problems/zigzag-grid-traversal-with-skip) | 50 | 65.3% | TCS |  |
| 27 | [Remove Element](https://leetcode.com/problems/remove-element) | 37.5 | 61.8% | TCS |  |
| 58 | [Length of Last Word](https://leetcode.com/problems/length-of-last-word) | 37.5 | 58.8% | TCS |  |
| 69 | [Sqrt(x)](https://leetcode.com/problems/sqrtx) | 37.5 | 41.8% | TCS, Infosys | [✅](#69-sqrtx) |
| 100 | [Same Tree](https://leetcode.com/problems/same-tree) | 37.5 | 67.1% | Accenture |  |
| 104 | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree) | 37.5 | 78.2% | Accenture, Infosys |  |
| 108 | [Convert Sorted Array to Binary Search Tree](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree) | 37.5 | 75.6% | Accenture |  |
| 190 | [Reverse Bits](https://leetcode.com/problems/reverse-bits) | 37.5 | 68.4% | Accenture |  |
| 206 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list) | 37.5 | 80.6% | Accenture, TCS | [✅](#206-reverse-linked-list) |
| 231 | [Power of Two](https://leetcode.com/problems/power-of-two) | 37.5 | 50.1% | Infosys, TCS |  |
| 232 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks) | 37.5 | 69.8% | Infosys |  |
| 234 | [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list) | 37.5 | 58% | Infosys |  |
| 258 | [Add Digits](https://leetcode.com/problems/add-digits) | 37.5 | 68.9% | Infosys |  |
| 263 | [Ugly Number](https://leetcode.com/problems/ugly-number) | 37.5 | 43.5% | TCS |  |
| 345 | [Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string) | 37.5 | 61.3% | Accenture |  |
| 387 | [First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string) | 37.5 | 65.5% | Accenture, TCS | [✅](#387-first-unique-character-in-a-string) |
| 389 | [Find the Difference](https://leetcode.com/problems/find-the-difference) | 37.5 | 60.3% | Accenture |  |
| 392 | [Is Subsequence](https://leetcode.com/problems/is-subsequence) | 37.5 | 49% | Infosys |  |
| 409 | [Longest Palindrome](https://leetcode.com/problems/longest-palindrome) | 37.5 | 56% | Accenture, TCS |  |
| 414 | [Third Maximum Number](https://leetcode.com/problems/third-maximum-number) | 37.5 | 39.4% | TCS |  |
| 507 | [Perfect Number](https://leetcode.com/problems/perfect-number) | 37.5 | 48.9% | Accenture |  |
| 541 | [Reverse String II](https://leetcode.com/problems/reverse-string-ii) | 37.5 | 53.8% | Accenture, Infosys |  |
| 543 | [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree) | 37.5 | 65.5% | TCS |  |
| 628 | [Maximum Product of Three Numbers](https://leetcode.com/problems/maximum-product-of-three-numbers) | 37.5 | 45.9% | TCS, Infosys |  |
| 643 | [Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i) | 37.5 | 47.8% | TCS |  |
| 724 | [Find Pivot Index](https://leetcode.com/problems/find-pivot-index) | 37.5 | 62.7% | Accenture |  |
| 796 | [Rotate String](https://leetcode.com/problems/rotate-string) | 37.5 | 66.6% | Accenture, TCS |  |
| 977 | [Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) | 37.5 | 73.8% | Accenture, TCS, Infosys | [✅](#977-squares-of-a-sorted-array) |
| 1431 | [Kids With the Greatest Number of Candies](https://leetcode.com/problems/kids-with-the-greatest-number-of-candies) | 37.5 | 88% | Infosys |  |
| 1512 | [Number of Good Pairs](https://leetcode.com/problems/number-of-good-pairs) | 37.5 | 89.8% | Accenture, TCS |  |
| 1636 | [Sort Array by Increasing Frequency](https://leetcode.com/problems/sort-array-by-increasing-frequency) | 37.5 | 80.8% | Accenture, TCS |  |
| 1742 | [Maximum Number of Balls in a Box](https://leetcode.com/problems/maximum-number-of-balls-in-a-box) | 37.5 | 74.9% | Accenture |  |
| 1903 | [Largest Odd Number in String](https://leetcode.com/problems/largest-odd-number-in-string) | 37.5 | 67.4% | Accenture |  |
| 1929 | [Concatenation of Array](https://leetcode.com/problems/concatenation-of-array) | 37.5 | 90.3% | Infosys |  |
| 2006 | [Count Number of Pairs With Absolute Difference K](https://leetcode.com/problems/count-number-of-pairs-with-absolute-difference-k) | 37.5 | 85.4% | TCS |  |
| 2011 | [Final Value of Variable After Performing Operations](https://leetcode.com/problems/final-value-of-variable-after-performing-operations) | 37.5 | 90.6% | Accenture |  |
| 2215 | [Find the Difference of Two Arrays](https://leetcode.com/problems/find-the-difference-of-two-arrays) | 37.5 | 81.4% | Accenture |  |
| 2235 | [Add Two Integers](https://leetcode.com/problems/add-two-integers) | 37.5 | 88% | Accenture |  |
| 3184 | [Count Pairs That Form a Complete Day I](https://leetcode.com/problems/count-pairs-that-form-a-complete-day-i) | 37.5 | 78.2% | Infosys |  |
| 83 | [Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list) | 25 | 56.7% | TCS |  |
| 94 | [Binary Tree Inorder Traversal](https://leetcode.com/problems/binary-tree-inorder-traversal) | 25 | 80.1% | TCS |  |
| 160 | [Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists) | 25 | 63.7% | TCS |  |
| 350 | [Intersection of Two Arrays II](https://leetcode.com/problems/intersection-of-two-arrays-ii) | 25 | 59.9% | TCS |  |
| 374 | [Guess Number Higher or Lower](https://leetcode.com/problems/guess-number-higher-or-lower) | 25 | 57.6% | TCS |  |
| 405 | [Convert a Number to Hexadecimal](https://leetcode.com/problems/convert-a-number-to-hexadecimal) | 25 | 54.1% | TCS |  |
| 459 | [Repeated Substring Pattern](https://leetcode.com/problems/repeated-substring-pattern) | 25 | 48.2% | TCS |  |
| 495 | [Teemo Attacking](https://leetcode.com/problems/teemo-attacking) | 25 | 57.7% | TCS |  |
| 566 | [Reshape the Matrix](https://leetcode.com/problems/reshape-the-matrix) | 25 | 65% | TCS |  |
| 746 | [Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs) | 25 | 68.3% | TCS |  |
| 1021 | [Remove Outermost Parentheses](https://leetcode.com/problems/remove-outermost-parentheses) | 25 | 87.1% | TCS |  |
| 1025 | [Divisor Game](https://leetcode.com/problems/divisor-game) | 25 | 71.9% | TCS |  |
| 1365 | [How Many Numbers Are Smaller Than the Current Number](https://leetcode.com/problems/how-many-numbers-are-smaller-than-the-current-number) | 25 | 87.4% | TCS |  |
| 1455 | [Check If a Word Occurs As a Prefix of Any Word in a Sentence](https://leetcode.com/problems/check-if-a-word-occurs-as-a-prefix-of-any-word-in-a-sentence) | 25 | 68.8% | TCS |  |
| 1480 | [Running Sum of 1d Array](https://leetcode.com/problems/running-sum-of-1d-array) | 25 | 86.9% | TCS |  |
| 1614 | [Maximum Nesting Depth of the Parentheses](https://leetcode.com/problems/maximum-nesting-depth-of-the-parentheses) | 25 | 84.9% | TCS |  |
| 3136 | [Valid Word](https://leetcode.com/problems/valid-word) | 25 | 50.9% | TCS |  |
| 3190 | [Find Minimum Operations to Make All Elements Divisible by Three](https://leetcode.com/problems/find-minimum-operations-to-make-all-elements-divisible-by-three) | 25 | 90.8% | TCS |  |
| 3330 | [Find the Original Typed String I](https://leetcode.com/problems/find-the-original-typed-string-i) | 25 | 72% | TCS |  |

### DSA Medium

172 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 5 | [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) | 100 | 37.8% | Mphasis, Accenture, Cognizant, TCS, Infosys, Accolite, HCL, Persistent Systems | [✅](#5-longest-palindromic-substring) |
| 7 | [Reverse Integer](https://leetcode.com/problems/reverse-integer) | 100 | 31.9% | Tech Mahindra, Accenture, LTI, Wipro, Cognizant, Capgemini, TCS, Infosys | [✅](#7-reverse-integer) |
| 53 | [Maximum Subarray](https://leetcode.com/problems/maximum-subarray) | 100 | 53.3% | Tech Mahindra, Accenture, TCS, Infosys, Cognizant, HCL, Persistent Systems, Accolite |  |
| 1234 | [Replace the Substring for Balanced String](https://leetcode.com/problems/replace-the-substring-for-balanced-string) | 100 | 40.9% | Accolite |  |
| 1551 | [Minimum Operations to Make Array Equal](https://leetcode.com/problems/minimum-operations-to-make-array-equal) | 100 | 82.7% | Brillio |  |
| 2271 | [Maximum White Tiles Covered by a Carpet](https://leetcode.com/problems/maximum-white-tiles-covered-by-a-carpet) | 100 | 35.9% | LTI |  |
| 2950 | [Number of Divisible Substrings](https://leetcode.com/problems/number-of-divisible-substrings) | 100 | 74.6% | Amdocs |  |
| 3001 | [Minimum Moves to Capture The Queen](https://leetcode.com/problems/minimum-moves-to-capture-the-queen) | 100 | 22.5% | Wipro |  |
| 3391 | [Design a 3D Binary Matrix with Efficient Layer Tracking](https://leetcode.com/problems/design-a-3d-binary-matrix-with-efficient-layer-tracking) | 100 | 66.4% | Amdocs |  |
| 3876 | [Construct Uniform Parity Array II](https://leetcode.com/problems/construct-uniform-parity-array-ii) | 100 | 49.9% | Amdocs |  |
| 3 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) | 87.5 | 39.1% | Infosys, Accenture, Accolite, Cognizant, Wipro, HCL, Capgemini, TCS, Persistent Systems |  |
| 2165 | [Smallest Value of the Rearranged Number](https://leetcode.com/problems/smallest-value-of-the-rearranged-number) | 87.5 | 53.6% | Cognizant |  |
| 2422 | [Merge Operations to Turn Array Into a Palindrome](https://leetcode.com/problems/merge-operations-to-turn-array-into-a-palindrome) | 87.5 | 68.9% | Accolite |  |
| 2747 | [Count Zero Request Servers](https://leetcode.com/problems/count-zero-request-servers) | 87.5 | 35.9% | LTI |  |
| 3819 | [Rotate Non Negative Elements](https://leetcode.com/problems/rotate-non-negative-elements) | 87.5 | 48% | Accolite |  |
| 17 | [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number) | 75 | 66.1% | Accenture, TCS | [✅](#17-letter-combinations-of-a-phone-number) |
| 49 | [Group Anagrams](https://leetcode.com/problems/group-anagrams) | 75 | 72.6% | Infosys, Wipro, Persistent Systems, Capgemini, Accolite, Cognizant, TCS | [✅](#49-group-anagrams) |
| 134 | [Gas Station](https://leetcode.com/problems/gas-station) | 75 | 48% | Infosys, Accolite |  |
| 189 | [Rotate Array](https://leetcode.com/problems/rotate-array) | 75 | 44.9% | Accenture, Capgemini, Wipro, Cognizant, TCS, Infosys | [✅](#189-rotate-array) |
| 322 | [Coin Change](https://leetcode.com/problems/coin-change) | 75 | 48.4% | Infosys, Accolite, Accenture, Capgemini |  |
| 560 | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) | 75 | 47.3% | Infosys, Accenture, Cognizant, TCS, Mindtree, Capgemini |  |
| 740 | [Delete and Earn](https://leetcode.com/problems/delete-and-earn) | 75 | 57.2% | Accenture |  |
| 875 | [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) | 75 | 50% | Accenture, TCS, Infosys |  |
| 1124 | [Longest Well-Performing Interval](https://leetcode.com/problems/longest-well-performing-interval) | 75 | 37.6% | Infosys |  |
| 1540 | [Can Convert String in K Moves](https://leetcode.com/problems/can-convert-string-in-k-moves) | 75 | 37.7% | Infosys |  |
| 1798 | [Maximum Number of Consecutive Values You Can Make](https://leetcode.com/problems/maximum-number-of-consecutive-values-you-can-make) | 75 | 64.1% | Infosys |  |
| 1946 | [Largest Number After Mutating Substring](https://leetcode.com/problems/largest-number-after-mutating-substring) | 75 | 38.2% | Infosys |  |
| 2233 | [Maximum Product After K Increments](https://leetcode.com/problems/maximum-product-after-k-increments) | 75 | 44% | Infosys |  |
| 2445 | [Number of Nodes With Value One](https://leetcode.com/problems/number-of-nodes-with-value-one) | 75 | 66.1% | Infosys |  |
| 2457 | [Minimum Addition to Make Integer Beautiful](https://leetcode.com/problems/minimum-addition-to-make-integer-beautiful) | 75 | 38.7% | Infosys |  |
| 2597 | [The Number of Beautiful Subsets](https://leetcode.com/problems/the-number-of-beautiful-subsets) | 75 | 50.9% | Infosys |  |
| 2829 | [Determine the Minimum Sum of a k-avoiding Array](https://leetcode.com/problems/determine-the-minimum-sum-of-a-k-avoiding-array) | 75 | 60.9% | Infosys |  |
| 2834 | [Find the Minimum Possible Sum of a Beautiful Array](https://leetcode.com/problems/find-the-minimum-possible-sum-of-a-beautiful-array) | 75 | 35% | Infosys |  |
| 3457 | [Eat Pizzas!](https://leetcode.com/problems/eat-pizzas) | 75 | 33.4% | Infosys |  |
| 3779 | [Minimum Number of Operations to Have Distinct Elements](https://leetcode.com/problems/minimum-number-of-operations-to-have-distinct-elements) | 75 | 42.4% | Accenture |  |
| 2 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers) | 62.5 | 48.5% | Accenture, Capgemini, Accolite, Cognizant, TCS, Infosys | [✅](#2-add-two-numbers-linked-lists) |
| 11 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water) | 62.5 | 60% | Accenture, Infosys, Capgemini, Accolite, TCS |  |
| 15 | [3Sum](https://leetcode.com/problems/3sum) | 62.5 | 39.1% | Accenture, TCS, Infosys, HCL | [✅](#15-3sum) |
| 19 | [Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list) | 62.5 | 51.6% | Accenture, TCS |  |
| 22 | [Generate Parentheses](https://leetcode.com/problems/generate-parentheses) | 62.5 | 78.7% | Infosys, Accenture, TCS |  |
| 24 | [Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs) | 62.5 | 69.5% | Altimetrik, TCS |  |
| 31 | [Next Permutation](https://leetcode.com/problems/next-permutation) | 62.5 | 45.3% | Infosys, Cognizant, Accenture, TCS | [✅](#31-next-permutation) |
| 34 | [Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) | 62.5 | 48.9% | Capgemini, TCS, Accenture, Infosys |  |
| 48 | [Rotate Image](https://leetcode.com/problems/rotate-image) | 62.5 | 80.1% | Accenture, Infosys, TCS | [✅](#48-rotate-image) |
| 56 | [Merge Intervals](https://leetcode.com/problems/merge-intervals) | 62.5 | 51.8% | Accenture, Wipro, TCS, Infosys |  |
| 75 | [Sort Colors](https://leetcode.com/problems/sort-colors) | 62.5 | 69.6% | TCS, Capgemini, Infosys | [✅](#75-sort-colors-dutch-national-flag) |
| 151 | [Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) | 62.5 | 56.4% | Accenture, HCL, Infosys, TCS | [✅](#151-reverse-words-in-a-string) |
| 162 | [Find Peak Element](https://leetcode.com/problems/find-peak-element) | 62.5 | 47% | Accenture, Infosys, TCS |  |
| 179 | [Largest Number](https://leetcode.com/problems/largest-number) | 62.5 | 43.1% | Accenture, TCS, Infosys | [✅](#179-largest-number) |
| 198 | [House Robber](https://leetcode.com/problems/house-robber) | 62.5 | 53.2% | Accenture, Infosys, TCS |  |
| 200 | [Number of Islands](https://leetcode.com/problems/number-of-islands) | 62.5 | 64.4% | Accenture, Infosys, TCS |  |
| 204 | [Count Primes](https://leetcode.com/problems/count-primes) | 62.5 | 36.1% | Accenture, TCS | [✅](#204-count-primes) |
| 209 | [Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum) | 62.5 | 51.7% | HCL, TCS |  |
| 279 | [Perfect Squares](https://leetcode.com/problems/perfect-squares) | 62.5 | 56.5% | Accenture |  |
| 319 | [Bulb Switcher](https://leetcode.com/problems/bulb-switcher) | 62.5 | 55.9% | Accenture, Infosys, TCS |  |
| 400 | [Nth Digit](https://leetcode.com/problems/nth-digit) | 62.5 | 38.2% | Accenture |  |
| 451 | [Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency) | 62.5 | 75.3% | Wipro, Accenture |  |
| 647 | [Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings) | 62.5 | 72.8% | HCL |  |
| 735 | [Asteroid Collision](https://leetcode.com/problems/asteroid-collision) | 62.5 | 47.9% | Accolite |  |
| 962 | [Maximum Width Ramp](https://leetcode.com/problems/maximum-width-ramp) | 62.5 | 55.9% | Accenture |  |
| 994 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges) | 62.5 | 58.7% | Infosys, TCS |  |
| 3653 | [XOR After Range Multiplication Queries I](https://leetcode.com/problems/xor-after-range-multiplication-queries-i) | 62.5 | 73.3% | Infosys |  |
| 12 | [Integer to Roman](https://leetcode.com/problems/integer-to-roman) | 50 | 71% | Infosys, Accenture, TCS | [✅](#12-integer-to-roman) |
| 18 | [4Sum](https://leetcode.com/problems/4sum) | 50 | 40.6% | Infosys, Accenture, TCS |  |
| 33 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) | 50 | 44.9% | TCS, Infosys, Accenture |  |
| 50 | [Pow(x, n)](https://leetcode.com/problems/powx-n) | 50 | 38.7% | TCS, Accenture, Infosys |  |
| 54 | [Spiral Matrix](https://leetcode.com/problems/spiral-matrix) | 50 | 56.8% | TCS, Infosys, Accenture | [✅](#54-spiral-matrix) |
| 55 | [Jump Game](https://leetcode.com/problems/jump-game) | 50 | 40.9% | Cognizant, TCS, Infosys |  |
| 62 | [Unique Paths](https://leetcode.com/problems/unique-paths) | 50 | 66.8% | TCS, Accenture |  |
| 78 | [Subsets](https://leetcode.com/problems/subsets) | 50 | 82.3% | Infosys, TCS |  |
| 80 | [Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii) | 50 | 64.7% | Accolite, TCS |  |
| 92 | [Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii) | 50 | 51.5% | Infosys |  |
| 128 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) | 50 | 47.1% | Capgemini, Infosys, TCS | [✅](#128-longest-consecutive-sequence) |
| 152 | [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray) | 50 | 36.4% | TCS |  |
| 215 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array) | 50 | 68.9% | Accenture, Infosys |  |
| 235 | [Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree) | 50 | 70.6% | Capgemini |  |
| 238 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) | 50 | 68.9% | Infosys, Accenture, TCS | [✅](#238-product-of-array-except-self) |
| 300 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) | 50 | 59.4% | Infosys, Accenture, TCS |  |
| 343 | [Integer Break](https://leetcode.com/problems/integer-break) | 50 | 62.3% | Accenture |  |
| 347 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements) | 50 | 66.4% | Infosys |  |
| 443 | [String Compression](https://leetcode.com/problems/string-compression) | 50 | 60% | Cognizant |  |
| 621 | [Task Scheduler](https://leetcode.com/problems/task-scheduler) | 50 | 63.1% | TCS |  |
| 898 | [Bitwise ORs of Subarrays](https://leetcode.com/problems/bitwise-ors-of-subarrays) | 50 | 56.9% | TCS |  |
| 912 | [Sort an Array](https://leetcode.com/problems/sort-an-array) | 50 | 55.9% | Infosys, Accenture, TCS |  |
| 1143 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) | 50 | 59.2% | Accolite, Accenture, TCS |  |
| 1405 | [Longest Happy String](https://leetcode.com/problems/longest-happy-string) | 50 | 65.5% | Capgemini |  |
| 1654 | [Minimum Jumps to Reach Home](https://leetcode.com/problems/minimum-jumps-to-reach-home) | 50 | 30.8% | Accolite |  |
| 1823 | [Find the Winner of the Circular Game](https://leetcode.com/problems/find-the-winner-of-the-circular-game) | 50 | 82.2% | Accenture, TCS | [✅](#1823-find-the-winner-of-the-circular-game) |
| 1877 | [Minimize Maximum Pair Sum in Array](https://leetcode.com/problems/minimize-maximum-pair-sum-in-array) | 50 | 83.3% | Capgemini |  |
| 2938 | [Separate Black and White Balls](https://leetcode.com/problems/separate-black-and-white-balls) | 50 | 63.9% | Accenture |  |
| 3066 | [Minimum Operations to Exceed Threshold Value II](https://leetcode.com/problems/minimum-operations-to-exceed-threshold-value-ii) | 50 | 45.8% | TCS |  |
| 6 | [Zigzag Conversion](https://leetcode.com/problems/zigzag-conversion) | 37.5 | 54.2% | Infosys, TCS |  |
| 8 | [String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) | 37.5 | 21% | Accenture, TCS, Infosys | [✅](#8-string-to-integer-atoi) |
| 39 | [Combination Sum](https://leetcode.com/problems/combination-sum) | 37.5 | 76.5% | Infosys |  |
| 40 | [Combination Sum II](https://leetcode.com/problems/combination-sum-ii) | 37.5 | 59.5% | Infosys |  |
| 45 | [Jump Game II](https://leetcode.com/problems/jump-game-ii) | 37.5 | 42.9% | TCS |  |
| 46 | [Permutations](https://leetcode.com/problems/permutations) | 37.5 | 81.9% | Infosys, TCS | [✅](#46-permutations) |
| 72 | [Edit Distance](https://leetcode.com/problems/edit-distance) | 37.5 | 60.6% | Accenture, Infosys |  |
| 73 | [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes) | 37.5 | 62.9% | TCS, Infosys |  |
| 77 | [Combinations](https://leetcode.com/problems/combinations) | 37.5 | 74.6% | Infosys |  |
| 79 | [Word Search](https://leetcode.com/problems/word-search) | 37.5 | 47.4% | Accenture |  |
| 81 | [Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii) | 37.5 | 40.1% | Accenture, TCS |  |
| 90 | [Subsets II](https://leetcode.com/problems/subsets-ii) | 37.5 | 61.3% | TCS, Infosys |  |
| 103 | [Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal) | 37.5 | 63.7% | Accenture |  |
| 120 | [Triangle](https://leetcode.com/problems/triangle) | 37.5 | 59.9% | Infosys |  |
| 122 | [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) | 37.5 | 71.1% | Accenture, TCS, Infosys | [✅](#122-best-time-to-buy-and-sell-stock-ii) |
| 131 | [Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning) | 37.5 | 74.1% | Accenture |  |
| 142 | [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii) | 37.5 | 57.9% | Infosys, TCS |  |
| 143 | [Reorder List](https://leetcode.com/problems/reorder-list) | 37.5 | 65.3% | Infosys, TCS |  |
| 146 | [LRU Cache](https://leetcode.com/problems/lru-cache) | 37.5 | 47.4% | TCS |  |
| 150 | [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation) | 37.5 | 57.8% | Infosys |  |
| 153 | [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array) | 37.5 | 54.7% | TCS, Infosys |  |
| 155 | [Min Stack](https://leetcode.com/problems/min-stack) | 37.5 | 58.2% | TCS, Infosys | [✅](#155-min-stack) |
| 167 | [Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted) | 37.5 | 65.1% | TCS, Infosys |  |
| 199 | [Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view) | 37.5 | 70.2% | Accenture |  |
| 213 | [House Robber II](https://leetcode.com/problems/house-robber-ii) | 37.5 | 44.9% | Infosys |  |
| 287 | [Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number) | 37.5 | 64.3% | TCS |  |
| 328 | [Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list) | 37.5 | 62.5% | Infosys, TCS |  |
| 396 | [Rotate Function](https://leetcode.com/problems/rotate-function) | 37.5 | 54.1% | Accenture |  |
| 402 | [Remove K Digits](https://leetcode.com/problems/remove-k-digits) | 37.5 | 36.9% | Accenture |  |
| 416 | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum) | 37.5 | 49.5% | TCS |  |
| 438 | [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string) | 37.5 | 53.8% | Accenture |  |
| 450 | [Delete Node in a BST](https://leetcode.com/problems/delete-node-in-a-bst) | 37.5 | 54.7% | Infosys |  |
| 462 | [Minimum Moves to Equal Array Elements II](https://leetcode.com/problems/minimum-moves-to-equal-array-elements-ii) | 37.5 | 61.9% | TCS |  |
| 516 | [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence) | 37.5 | 65.4% | Accenture, TCS, Infosys |  |
| 525 | [Contiguous Array](https://leetcode.com/problems/contiguous-array) | 37.5 | 51.3% | Accenture, Infosys |  |
| 532 | [K-diff Pairs in an Array](https://leetcode.com/problems/k-diff-pairs-in-an-array) | 37.5 | 45.9% | TCS, Infosys |  |
| 540 | [Single Element in a Sorted Array](https://leetcode.com/problems/single-element-in-a-sorted-array) | 37.5 | 59.3% | TCS |  |
| 542 | [01 Matrix](https://leetcode.com/problems/01-matrix) | 37.5 | 53.9% | Accenture |  |
| 556 | [Next Greater Element III](https://leetcode.com/problems/next-greater-element-iii) | 37.5 | 35.3% | Infosys |  |
| 583 | [Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings) | 37.5 | 65.6% | Infosys |  |
| 658 | [Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements) | 37.5 | 49.7% | Infosys, TCS |  |
| 670 | [Maximum Swap](https://leetcode.com/problems/maximum-swap) | 37.5 | 52% | Accenture, TCS |  |
| 739 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures) | 37.5 | 68.7% | Accenture, Infosys, TCS |  |
| 767 | [Reorganize String](https://leetcode.com/problems/reorganize-string) | 37.5 | 57.1% | Infosys |  |
| 840 | [Magic Squares In Grid](https://leetcode.com/problems/magic-squares-in-grid) | 37.5 | 55.2% | Infosys |  |
| 841 | [Keys and Rooms](https://leetcode.com/problems/keys-and-rooms) | 37.5 | 75.8% | Infosys |  |
| 852 | [Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array) | 37.5 | 66.8% | Accenture, TCS | [✅](#852-peak-index-in-a-mountain-array) |
| 853 | [Car Fleet](https://leetcode.com/problems/car-fleet) | 37.5 | 55.1% | Infosys |  |
| 907 | [Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums) | 37.5 | 38.6% | Accenture |  |
| 945 | [Minimum Increment to Make Array Unique](https://leetcode.com/problems/minimum-increment-to-make-array-unique) | 37.5 | 60.7% | Infosys |  |
| 1004 | [Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii) | 37.5 | 67.7% | Infosys |  |
| 1094 | [Car Pooling](https://leetcode.com/problems/car-pooling) | 37.5 | 56.4% | Infosys |  |
| 1376 | [Time Needed to Inform All Employees](https://leetcode.com/problems/time-needed-to-inform-all-employees) | 37.5 | 60.5% | Infosys |  |
| 1493 | [Longest Subarray of 1's After Deleting One Element](https://leetcode.com/problems/longest-subarray-of-1s-after-deleting-one-element) | 37.5 | 71.2% | TCS |  |
| 1498 | [Number of Subsequences That Satisfy the Given Sum Condition](https://leetcode.com/problems/number-of-subsequences-that-satisfy-the-given-sum-condition) | 37.5 | 49.2% | Infosys |  |
| 1838 | [Frequency of the Most Frequent Element](https://leetcode.com/problems/frequency-of-the-most-frequent-element) | 37.5 | 44.8% | Infosys |  |
| 1910 | [Remove All Occurrences of a Substring](https://leetcode.com/problems/remove-all-occurrences-of-a-substring) | 37.5 | 78.5% | Infosys, TCS |  |
| 1922 | [Count Good Numbers](https://leetcode.com/problems/count-good-numbers) | 37.5 | 57.7% | Infosys |  |
| 2095 | [Delete the Middle Node of a Linked List](https://leetcode.com/problems/delete-the-middle-node-of-a-linked-list) | 37.5 | 59.6% | TCS |  |
| 2149 | [Rearrange Array Elements by Sign](https://leetcode.com/problems/rearrange-array-elements-by-sign) | 37.5 | 84.6% | Infosys |  |
| 2684 | [Maximum Number of Moves in a Grid](https://leetcode.com/problems/maximum-number-of-moves-in-a-grid) | 37.5 | 58.7% | Accenture |  |
| 38 | [Count and Say](https://leetcode.com/problems/count-and-say) | 25 | 62.9% | TCS |  |
| 63 | [Unique Paths II](https://leetcode.com/problems/unique-paths-ii) | 25 | 44.5% | TCS |  |
| 74 | [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix) | 25 | 53.9% | TCS |  |
| 109 | [Convert Sorted List to Binary Search Tree](https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree) | 25 | 66.6% | TCS |  |
| 164 | [Maximum Gap](https://leetcode.com/problems/maximum-gap) | 25 | 52.1% | TCS |  |
| 229 | [Majority Element II](https://leetcode.com/problems/majority-element-ii) | 25 | 56.2% | TCS |  |
| 435 | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals) | 25 | 57.1% | TCS |  |
| 442 | [Find All Duplicates in an Array](https://leetcode.com/problems/find-all-duplicates-in-an-array) | 25 | 76.9% | TCS |  |
| 523 | [Continuous Subarray Sum](https://leetcode.com/problems/continuous-subarray-sum) | 25 | 31.4% | TCS |  |
| 567 | [Permutation in String](https://leetcode.com/problems/permutation-in-string) | 25 | 48.9% | TCS |  |
| 713 | [Subarray Product Less Than K](https://leetcode.com/problems/subarray-product-less-than-k) | 25 | 54.3% | TCS |  |
| 979 | [Distribute Coins in Binary Tree](https://leetcode.com/problems/distribute-coins-in-binary-tree) | 25 | 77.3% | TCS |  |
| 1015 | [Smallest Integer Divisible by K](https://leetcode.com/problems/smallest-integer-divisible-by-k) | 25 | 54.3% | TCS |  |
| 1492 | [The kth Factor of n](https://leetcode.com/problems/the-kth-factor-of-n) | 25 | 70.4% | TCS |  |
| 1497 | [Check If Array Pairs Are Divisible by k](https://leetcode.com/problems/check-if-array-pairs-are-divisible-by-k) | 25 | 46.2% | TCS |  |
| 2054 | [Two Best Non-Overlapping Events](https://leetcode.com/problems/two-best-non-overlapping-events) | 25 | 64% | TCS |  |
| 2461 | [Maximum Sum of Distinct Subarrays With Length K](https://leetcode.com/problems/maximum-sum-of-distinct-subarrays-with-length-k) | 25 | 43% | TCS |  |
| 2563 | [Count the Number of Fair Pairs](https://leetcode.com/problems/count-the-number-of-fair-pairs) | 25 | 52.7% | TCS |  |
| 3163 | [String Compression III](https://leetcode.com/problems/string-compression-iii) | 25 | 67.1% | TCS |  |
| 3714 | [Longest Balanced Substring II](https://leetcode.com/problems/longest-balanced-substring-ii) | 25 | 41.9% | TCS |  |

### DSA Hard

57 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 3463 | [Check If Digits Are Equal in String After Operations II](https://leetcode.com/problems/check-if-digits-are-equal-in-string-after-operations-ii) | 100 | 14.4% | ADP |  |
| 1755 | [Closest Subsequence Sum](https://leetcode.com/problems/closest-subsequence-sum) | 87.5 | 43.7% | LTI |  |
| 3348 | [Smallest Divisible Digit Product II](https://leetcode.com/problems/smallest-divisible-digit-product-ii) | 87.5 | 14.6% | Accenture |  |
| 214 | [Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome) | 75 | 42.5% | Accenture |  |
| 1872 | [Stone Game VIII](https://leetcode.com/problems/stone-game-viii) | 75 | 53.9% | Infosys |  |
| 2338 | [Count the Number of Ideal Arrays](https://leetcode.com/problems/count-the-number-of-ideal-arrays) | 75 | 56.9% | Infosys |  |
| 2382 | [Maximum Segment Sum After Removals](https://leetcode.com/problems/maximum-segment-sum-after-removals) | 75 | 49.7% | Infosys |  |
| 2463 | [Minimum Total Distance Traveled](https://leetcode.com/problems/minimum-total-distance-traveled) | 75 | 63.2% | Infosys |  |
| 2612 | [Minimum Reverse Operations](https://leetcode.com/problems/minimum-reverse-operations) | 75 | 17% | Infosys |  |
| 2827 | [Number of Beautiful Integers in the Range](https://leetcode.com/problems/number-of-beautiful-integers-in-the-range) | 75 | 22.3% | Infosys |  |
| 2940 | [Find Building Where Alice and Bob Can Meet](https://leetcode.com/problems/find-building-where-alice-and-bob-can-meet) | 75 | 52.2% | Infosys |  |
| 3165 | [Maximum Sum of Subsequence With Non-adjacent Elements](https://leetcode.com/problems/maximum-sum-of-subsequence-with-non-adjacent-elements) | 75 | 15.9% | Infosys |  |
| 3336 | [Find the Number of Subsequences With Equal GCD](https://leetcode.com/problems/find-the-number-of-subsequences-with-equal-gcd) | 75 | 31.7% | Infosys |  |
| 3449 | [Maximize the Minimum Game Score](https://leetcode.com/problems/maximize-the-minimum-game-score) | 75 | 26.5% | Infosys |  |
| 3575 | [Maximum Good Subtree Score](https://leetcode.com/problems/maximum-good-subtree-score) | 75 | 45.5% | Infosys |  |
| 3640 | [Trionic Array II](https://leetcode.com/problems/trionic-array-ii) | 75 | 47.3% | Infosys |  |
| 3661 | [Maximum Walls Destroyed by Robots](https://leetcode.com/problems/maximum-walls-destroyed-by-robots) | 75 | 48% | Infosys |  |
| 4 | [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) | 62.5 | 46.6% | Accenture, Capgemini, Cognizant, TCS | [✅](#4-median-of-two-sorted-arrays) |
| 41 | [First Missing Positive](https://leetcode.com/problems/first-missing-positive) | 62.5 | 42.9% | Cognizant, TCS, Infosys |  |
| 3655 | [XOR After Range Multiplication Queries II](https://leetcode.com/problems/xor-after-range-multiplication-queries-ii) | 62.5 | 47.7% | Infosys |  |
| 3671 | [Sum of Beautiful Subsequences](https://leetcode.com/problems/sum-of-beautiful-subsequences) | 62.5 | 31.6% | Infosys |  |
| 3901 | [Good Subsequence Queries](https://leetcode.com/problems/good-subsequence-queries) | 62.5 | 19.6% | Infosys |  |
| 42 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) | 50 | 67.4% | Capgemini, TCS, Infosys, Accenture | [✅](#42-trapping-rain-water) |
| 51 | [N-Queens](https://leetcode.com/problems/n-queens) | 50 | 75.5% | TCS, Accenture, Infosys |  |
| 312 | [Burst Balloons](https://leetcode.com/problems/burst-balloons) | 50 | 63.5% | TCS |  |
| 321 | [Create Maximum Number](https://leetcode.com/problems/create-maximum-number) | 50 | 35.5% | Accolite |  |
| 1745 | [Palindrome Partitioning IV](https://leetcode.com/problems/palindrome-partitioning-iv) | 50 | 45.5% | TCS |  |
| 10 | [Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) | 37.5 | 31% | Accenture, Infosys, TCS |  |
| 23 | [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists) | 37.5 | 59.6% | TCS |  |
| 25 | [Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group) | 37.5 | 66.1% | Infosys |  |
| 30 | [Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words) | 37.5 | 34.4% | Infosys |  |
| 32 | [Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses) | 37.5 | 38.8% | Accenture, Infosys |  |
| 44 | [Wildcard Matching](https://leetcode.com/problems/wildcard-matching) | 37.5 | 32% | Infosys |  |
| 76 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring) | 37.5 | 47.5% | Infosys |  |
| 84 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram) | 37.5 | 49.9% | Infosys, TCS |  |
| 85 | [Maximal Rectangle](https://leetcode.com/problems/maximal-rectangle) | 37.5 | 58.8% | TCS |  |
| 124 | [Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum) | 37.5 | 42.3% | TCS |  |
| 135 | [Candy](https://leetcode.com/problems/candy) | 37.5 | 48.5% | Accenture, Infosys |  |
| 239 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum) | 37.5 | 48.8% | TCS |  |
| 297 | [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree) | 37.5 | 60.8% | TCS |  |
| 410 | [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum) | 37.5 | 60.4% | Infosys |  |
| 805 | [Split Array With Same Average](https://leetcode.com/problems/split-array-with-same-average) | 37.5 | 27% | TCS |  |
| 956 | [Tallest Billboard](https://leetcode.com/problems/tallest-billboard) | 37.5 | 51.9% | TCS |  |
| 992 | [Subarrays with K Different Integers](https://leetcode.com/problems/subarrays-with-k-different-integers) | 37.5 | 68.1% | Infosys |  |
| 1125 | [Smallest Sufficient Team](https://leetcode.com/problems/smallest-sufficient-team) | 37.5 | 55.6% | TCS |  |
| 1235 | [Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling) | 37.5 | 54.7% | Infosys |  |
| 1312 | [Minimum Insertion Steps to Make a String Palindrome](https://leetcode.com/problems/minimum-insertion-steps-to-make-a-string-palindrome) | 37.5 | 73.9% | Accenture |  |
| 1819 | [Number of Different Subsequences GCDs](https://leetcode.com/problems/number-of-different-subsequences-gcds) | 37.5 | 45.3% | Infosys |  |
| 2163 | [Minimum Difference in Sums After Removal of Elements](https://leetcode.com/problems/minimum-difference-in-sums-after-removal-of-elements) | 37.5 | 69.6% | Infosys |  |
| 2493 | [Divide Nodes Into the Maximum Number of Groups](https://leetcode.com/problems/divide-nodes-into-the-maximum-number-of-groups) | 37.5 | 66.9% | Accenture |  |
| 2872 | [Maximum Number of K-Divisible Components](https://leetcode.com/problems/maximum-number-of-k-divisible-components) | 37.5 | 74% | Infosys |  |
| 3539 | [Find Sum of Array Product of Magical Sequences](https://leetcode.com/problems/find-sum-of-array-product-of-magical-sequences) | 37.5 | 61.8% | Infosys |  |
| 3666 | [Minimum Operations to Equalize Binary String](https://leetcode.com/problems/minimum-operations-to-equalize-binary-string) | 37.5 | 45.1% | Infosys |  |
| 127 | [Word Ladder](https://leetcode.com/problems/word-ladder) | 25 | 45.6% | TCS |  |
| 273 | [Integer to English Words](https://leetcode.com/problems/integer-to-english-words) | 25 | 35% | TCS |  |
| 1368 | [Minimum Cost to Make at Least One Valid Path in a Grid](https://leetcode.com/problems/minimum-cost-to-make-at-least-one-valid-path-in-a-grid) | 25 | 71% | TCS |  |
| 1751 | [Maximum Number of Events That Can Be Attended II](https://leetcode.com/problems/maximum-number-of-events-that-can-be-attended-ii) | 25 | 63.5% | TCS |  |

### SQL Easy

17 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 2356 | [Number of Unique Subjects Taught by Each Teacher](https://leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher) | 87.5 | 89.2% | Capgemini |  |
| 1661 | [Average Time of Process per Machine](https://leetcode.com/problems/average-time-of-process-per-machine) | 75 | 66.8% | Mindtree, Accenture |  |
| 175 | [Combine Two Tables](https://leetcode.com/problems/combine-two-tables) | 62.5 | 79.5% | Cognizant, Infosys |  |
| 1068 | [Product Sales Analysis I](https://leetcode.com/problems/product-sales-analysis-i) | 62.5 | 85.8% | Cognizant, TCS |  |
| 1148 | [Article Views I](https://leetcode.com/problems/article-views-i) | 62.5 | 76.6% | Cognizant, Accenture, TCS |  |
| 181 | [Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers) | 50 | 73.2% | Accenture, Cognizant, TCS |  |
| 197 | [Rising Temperature](https://leetcode.com/problems/rising-temperature) | 50 | 51.3% | Accenture, Cognizant |  |
| 1280 | [Students and Examinations](https://leetcode.com/problems/students-and-examinations) | 50 | 61.2% | Cognizant |  |
| 1378 | [Replace Employee ID With The Unique Identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier) | 50 | 83.5% | Cognizant, Infosys, TCS |  |
| 182 | [Duplicate Emails](https://leetcode.com/problems/duplicate-emails) | 37.5 | 73.8% | TCS |  |
| 196 | [Delete Duplicate Emails](https://leetcode.com/problems/delete-duplicate-emails) | 37.5 | 66% | Infosys, TCS |  |
| 584 | [Find Customer Referee](https://leetcode.com/problems/find-customer-referee) | 37.5 | 72.9% | TCS |  |
| 1075 | [Project Employees I](https://leetcode.com/problems/project-employees-i) | 37.5 | 66.8% | TCS |  |
| 1581 | [Customer Who Visited but Did Not Make Any Transactions](https://leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions) | 37.5 | 67.7% | TCS |  |
| 1757 | [Recyclable and Low Fat Products](https://leetcode.com/problems/recyclable-and-low-fat-products) | 37.5 | 88.6% | TCS |  |
| 577 | [Employee Bonus](https://leetcode.com/problems/employee-bonus) | 25 | 77.4% | TCS |  |
| 595 | [Big Countries](https://leetcode.com/problems/big-countries) | 25 | 68.5% | TCS |  |

### SQL Medium

7 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 2993 | [Friday Purchases I](https://leetcode.com/problems/friday-purchases-i) | 100 | 81.1% | TCS |  |
| 176 | [Second Highest Salary](https://leetcode.com/problems/second-highest-salary) | 87.5 | 47% | LTI, Cognizant, Infosys, TCS, HCL, Accenture, Capgemini |  |
| 626 | [Exchange Seats](https://leetcode.com/problems/exchange-seats) | 62.5 | 74.2% | Capgemini |  |
| 177 | [Nth Highest Salary](https://leetcode.com/problems/nth-highest-salary) | 37.5 | 39.2% | Accenture |  |
| 178 | [Rank Scores](https://leetcode.com/problems/rank-scores) | 37.5 | 67.7% | Accenture |  |
| 184 | [Department Highest Salary](https://leetcode.com/problems/department-highest-salary) | 25 | 58.1% | TCS |  |
| 570 | [Managers with at Least 5 Direct Reports](https://leetcode.com/problems/managers-with-at-least-5-direct-reports) | 25 | 49% | TCS |  |

### SQL Hard

1 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 185 | [Department Top Three Salaries](https://leetcode.com/problems/department-top-three-salaries) | 62.5 | 60.5% | HCL, Accenture, TCS |  |

### JavaScript Easy

3 questions, most frequent first.

| # | Question | Freq | Acc | Companies | Java |
|---|---|---|---|---|---|
| 2667 | [Create Hello World Function](https://leetcode.com/problems/create-hello-world-function) | 62.5 | 81.9% | Wipro, TCS |  |
| 2677 | [Chunk Array](https://leetcode.com/problems/chunk-array) | 50 | 84.6% | Capgemini |  |
| 2703 | [Return Length of Arguments Passed](https://leetcode.com/problems/return-length-of-arguments-passed) | 25 | 94.5% | TCS |  |

## Part 2 — By company

Questions each company has asked, most frequent first. ✅ = solvable in Java inside SkillForge.

### TCS (216)

- **Easy (95):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [169. Majority Element](https://leetcode.com/problems/majority-element) ✅ · [202. Happy Number](https://leetcode.com/problems/happy-number) · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [2099. Find Subsequence of Length K With the Largest Sum](https://leetcode.com/problems/find-subsequence-of-length-k-with-the-largest-sum) · [13. Roman to Integer](https://leetcode.com/problems/roman-to-integer) ✅ · [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) ✅ · [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) ✅ · [28. Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) · [66. Plus One](https://leetcode.com/problems/plus-one) ✅ · [67. Add Binary](https://leetcode.com/problems/add-binary) ✅ · [118. Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) ✅ · [136. Single Number](https://leetcode.com/problems/single-number) · [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate) ✅ · [268. Missing Number](https://leetcode.com/problems/missing-number) · [344. Reverse String](https://leetcode.com/problems/reverse-string) · [349. Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) · [455. Assign Cookies](https://leetcode.com/problems/assign-cookies) · [485. Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) ✅ · [766. Toeplitz Matrix](https://leetcode.com/problems/toeplitz-matrix) · [876. Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list) · [1068. Product Sales Analysis I](https://leetcode.com/problems/product-sales-analysis-i) · [1148. Article Views I](https://leetcode.com/problems/article-views-i) · [1518. Water Bottles](https://leetcode.com/problems/water-bottles) · [1800. Maximum Ascending Subarray Sum](https://leetcode.com/problems/maximum-ascending-subarray-sum) · [2423. Remove Letter To Equalize Frequency](https://leetcode.com/problems/remove-letter-to-equalize-frequency) · [2520. Count the Digits That Divide a Number](https://leetcode.com/problems/count-the-digits-that-divide-a-number) · [2667. Create Hello World Function](https://leetcode.com/problems/create-hello-world-function) · [35. Search Insert Position](https://leetcode.com/problems/search-insert-position) ✅ · [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) ✅ · [181. Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers) · [219. Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii) · [303. Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable) · [412. Fizz Buzz](https://leetcode.com/problems/fizz-buzz) · [496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i) · [704. Binary Search](https://leetcode.com/problems/binary-search) · [1378. Replace Employee ID With The Unique Identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier) · [1752. Check if Array Is Sorted and Rotated](https://leetcode.com/problems/check-if-array-is-sorted-and-rotated) · [2481. Minimum Cuts to Divide a Circle](https://leetcode.com/problems/minimum-cuts-to-divide-a-circle) · [3065. Minimum Operations to Exceed Threshold Value I](https://leetcode.com/problems/minimum-operations-to-exceed-threshold-value-i) · [3417. Zigzag Grid Traversal With Skip](https://leetcode.com/problems/zigzag-grid-traversal-with-skip) · [27. Remove Element](https://leetcode.com/problems/remove-element) · [58. Length of Last Word](https://leetcode.com/problems/length-of-last-word) · [69. Sqrt(x)](https://leetcode.com/problems/sqrtx) ✅ · [182. Duplicate Emails](https://leetcode.com/problems/duplicate-emails) · [196. Delete Duplicate Emails](https://leetcode.com/problems/delete-duplicate-emails) · [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list) ✅ · [231. Power of Two](https://leetcode.com/problems/power-of-two) · [263. Ugly Number](https://leetcode.com/problems/ugly-number) · [387. First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string) ✅ · [409. Longest Palindrome](https://leetcode.com/problems/longest-palindrome) · [414. Third Maximum Number](https://leetcode.com/problems/third-maximum-number) · [543. Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree) · [584. Find Customer Referee](https://leetcode.com/problems/find-customer-referee) · [628. Maximum Product of Three Numbers](https://leetcode.com/problems/maximum-product-of-three-numbers) · [643. Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i) · [796. Rotate String](https://leetcode.com/problems/rotate-string) · [977. Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) ✅ · [1075. Project Employees I](https://leetcode.com/problems/project-employees-i) · [1512. Number of Good Pairs](https://leetcode.com/problems/number-of-good-pairs) · [1581. Customer Who Visited but Did Not Make Any Transactions](https://leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions) · [1636. Sort Array by Increasing Frequency](https://leetcode.com/problems/sort-array-by-increasing-frequency) · [1757. Recyclable and Low Fat Products](https://leetcode.com/problems/recyclable-and-low-fat-products) · [2006. Count Number of Pairs With Absolute Difference K](https://leetcode.com/problems/count-number-of-pairs-with-absolute-difference-k) · [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list) · [94. Binary Tree Inorder Traversal](https://leetcode.com/problems/binary-tree-inorder-traversal) · [160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists) · [350. Intersection of Two Arrays II](https://leetcode.com/problems/intersection-of-two-arrays-ii) · [374. Guess Number Higher or Lower](https://leetcode.com/problems/guess-number-higher-or-lower) · [405. Convert a Number to Hexadecimal](https://leetcode.com/problems/convert-a-number-to-hexadecimal) · [459. Repeated Substring Pattern](https://leetcode.com/problems/repeated-substring-pattern) · [495. Teemo Attacking](https://leetcode.com/problems/teemo-attacking) · [566. Reshape the Matrix](https://leetcode.com/problems/reshape-the-matrix) · [577. Employee Bonus](https://leetcode.com/problems/employee-bonus) · [595. Big Countries](https://leetcode.com/problems/big-countries) · [746. Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs) · [1021. Remove Outermost Parentheses](https://leetcode.com/problems/remove-outermost-parentheses) · [1025. Divisor Game](https://leetcode.com/problems/divisor-game) · [1365. How Many Numbers Are Smaller Than the Current Number](https://leetcode.com/problems/how-many-numbers-are-smaller-than-the-current-number) · [1455. Check If a Word Occurs As a Prefix of Any Word in a Sentence](https://leetcode.com/problems/check-if-a-word-occurs-as-a-prefix-of-any-word-in-a-sentence) · [1480. Running Sum of 1d Array](https://leetcode.com/problems/running-sum-of-1d-array) · [1614. Maximum Nesting Depth of the Parentheses](https://leetcode.com/problems/maximum-nesting-depth-of-the-parentheses) · [2703. Return Length of Arguments Passed](https://leetcode.com/problems/return-length-of-arguments-passed) · [3136. Valid Word](https://leetcode.com/problems/valid-word) · [3190. Find Minimum Operations to Make All Elements Divisible by Three](https://leetcode.com/problems/find-minimum-operations-to-make-all-elements-divisible-by-three) · [3330. Find the Original Typed String I](https://leetcode.com/problems/find-the-original-typed-string-i)
- **Medium (100):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [2993. Friday Purchases I](https://leetcode.com/problems/friday-purchases-i) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number) ✅ · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) · [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water) · [15. 3Sum](https://leetcode.com/problems/3sum) ✅ · [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list) · [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses) · [24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs) · [31. Next Permutation](https://leetcode.com/problems/next-permutation) ✅ · [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) · [48. Rotate Image](https://leetcode.com/problems/rotate-image) ✅ · [56. Merge Intervals](https://leetcode.com/problems/merge-intervals) · [75. Sort Colors](https://leetcode.com/problems/sort-colors) ✅ · [151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) ✅ · [162. Find Peak Element](https://leetcode.com/problems/find-peak-element) · [179. Largest Number](https://leetcode.com/problems/largest-number) ✅ · [198. House Robber](https://leetcode.com/problems/house-robber) · [200. Number of Islands](https://leetcode.com/problems/number-of-islands) · [204. Count Primes](https://leetcode.com/problems/count-primes) ✅ · [209. Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum) · [319. Bulb Switcher](https://leetcode.com/problems/bulb-switcher) · [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges) · [12. Integer to Roman](https://leetcode.com/problems/integer-to-roman) ✅ · [18. 4Sum](https://leetcode.com/problems/4sum) · [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) · [50. Pow(x, n)](https://leetcode.com/problems/powx-n) · [54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix) ✅ · [55. Jump Game](https://leetcode.com/problems/jump-game) · [62. Unique Paths](https://leetcode.com/problems/unique-paths) · [78. Subsets](https://leetcode.com/problems/subsets) · [80. Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii) · [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) ✅ · [152. Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray) · [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) ✅ · [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) · [621. Task Scheduler](https://leetcode.com/problems/task-scheduler) · [898. Bitwise ORs of Subarrays](https://leetcode.com/problems/bitwise-ors-of-subarrays) · [912. Sort an Array](https://leetcode.com/problems/sort-an-array) · [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) · [1823. Find the Winner of the Circular Game](https://leetcode.com/problems/find-the-winner-of-the-circular-game) ✅ · [3066. Minimum Operations to Exceed Threshold Value II](https://leetcode.com/problems/minimum-operations-to-exceed-threshold-value-ii) · [6. Zigzag Conversion](https://leetcode.com/problems/zigzag-conversion) · [8. String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) ✅ · [45. Jump Game II](https://leetcode.com/problems/jump-game-ii) · [46. Permutations](https://leetcode.com/problems/permutations) ✅ · [73. Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes) · [81. Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii) · [90. Subsets II](https://leetcode.com/problems/subsets-ii) · [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) ✅ · [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii) · [143. Reorder List](https://leetcode.com/problems/reorder-list) · [146. LRU Cache](https://leetcode.com/problems/lru-cache) · [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array) · [155. Min Stack](https://leetcode.com/problems/min-stack) ✅ · [167. Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted) · [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number) · [328. Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list) · [416. Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum) · [462. Minimum Moves to Equal Array Elements II](https://leetcode.com/problems/minimum-moves-to-equal-array-elements-ii) · [516. Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence) · [532. K-diff Pairs in an Array](https://leetcode.com/problems/k-diff-pairs-in-an-array) · [540. Single Element in a Sorted Array](https://leetcode.com/problems/single-element-in-a-sorted-array) · [658. Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements) · [670. Maximum Swap](https://leetcode.com/problems/maximum-swap) · [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures) · [852. Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array) ✅ · [1493. Longest Subarray of 1's After Deleting One Element](https://leetcode.com/problems/longest-subarray-of-1s-after-deleting-one-element) · [1910. Remove All Occurrences of a Substring](https://leetcode.com/problems/remove-all-occurrences-of-a-substring) · [2095. Delete the Middle Node of a Linked List](https://leetcode.com/problems/delete-the-middle-node-of-a-linked-list) · [38. Count and Say](https://leetcode.com/problems/count-and-say) · [63. Unique Paths II](https://leetcode.com/problems/unique-paths-ii) · [74. Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix) · [109. Convert Sorted List to Binary Search Tree](https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree) · [164. Maximum Gap](https://leetcode.com/problems/maximum-gap) · [184. Department Highest Salary](https://leetcode.com/problems/department-highest-salary) · [229. Majority Element II](https://leetcode.com/problems/majority-element-ii) · [435. Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals) · [442. Find All Duplicates in an Array](https://leetcode.com/problems/find-all-duplicates-in-an-array) · [523. Continuous Subarray Sum](https://leetcode.com/problems/continuous-subarray-sum) · [567. Permutation in String](https://leetcode.com/problems/permutation-in-string) · [570. Managers with at Least 5 Direct Reports](https://leetcode.com/problems/managers-with-at-least-5-direct-reports) · [713. Subarray Product Less Than K](https://leetcode.com/problems/subarray-product-less-than-k) · [979. Distribute Coins in Binary Tree](https://leetcode.com/problems/distribute-coins-in-binary-tree) · [1015. Smallest Integer Divisible by K](https://leetcode.com/problems/smallest-integer-divisible-by-k) · [1492. The kth Factor of n](https://leetcode.com/problems/the-kth-factor-of-n) · [1497. Check If Array Pairs Are Divisible by k](https://leetcode.com/problems/check-if-array-pairs-are-divisible-by-k) · [2054. Two Best Non-Overlapping Events](https://leetcode.com/problems/two-best-non-overlapping-events) · [2461. Maximum Sum of Distinct Subarrays With Length K](https://leetcode.com/problems/maximum-sum-of-distinct-subarrays-with-length-k) · [2563. Count the Number of Fair Pairs](https://leetcode.com/problems/count-the-number-of-fair-pairs) · [3163. String Compression III](https://leetcode.com/problems/string-compression-iii) · [3714. Longest Balanced Substring II](https://leetcode.com/problems/longest-balanced-substring-ii)
- **Hard (21):** [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) ✅ · [41. First Missing Positive](https://leetcode.com/problems/first-missing-positive) · [185. Department Top Three Salaries](https://leetcode.com/problems/department-top-three-salaries) · [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) ✅ · [51. N-Queens](https://leetcode.com/problems/n-queens) · [312. Burst Balloons](https://leetcode.com/problems/burst-balloons) · [1745. Palindrome Partitioning IV](https://leetcode.com/problems/palindrome-partitioning-iv) · [10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) · [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists) · [84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram) · [85. Maximal Rectangle](https://leetcode.com/problems/maximal-rectangle) · [124. Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum) · [239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum) · [297. Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree) · [805. Split Array With Same Average](https://leetcode.com/problems/split-array-with-same-average) · [956. Tallest Billboard](https://leetcode.com/problems/tallest-billboard) · [1125. Smallest Sufficient Team](https://leetcode.com/problems/smallest-sufficient-team) · [127. Word Ladder](https://leetcode.com/problems/word-ladder) · [273. Integer to English Words](https://leetcode.com/problems/integer-to-english-words) · [1368. Minimum Cost to Make at Least One Valid Path in a Grid](https://leetcode.com/problems/minimum-cost-to-make-at-least-one-valid-path-in-a-grid) · [1751. Maximum Number of Events That Can Be Attended II](https://leetcode.com/problems/maximum-number-of-events-that-can-be-attended-ii)

### Infosys (172)

- **Easy (44):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [169. Majority Element](https://leetcode.com/problems/majority-element) ✅ · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [2418. Sort the People](https://leetcode.com/problems/sort-the-people) · [3467. Transform Array by Parity](https://leetcode.com/problems/transform-array-by-parity) · [13. Roman to Integer](https://leetcode.com/problems/roman-to-integer) ✅ · [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) ✅ · [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) ✅ · [28. Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) · [67. Add Binary](https://leetcode.com/problems/add-binary) ✅ · [118. Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) ✅ · [175. Combine Two Tables](https://leetcode.com/problems/combine-two-tables) · [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate) ✅ · [344. Reverse String](https://leetcode.com/problems/reverse-string) · [349. Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) · [2259. Remove Digit From Number to Maximize Result](https://leetcode.com/problems/remove-digit-from-number-to-maximize-result) · [3637. Trionic Array I](https://leetcode.com/problems/trionic-array-i) · [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) ✅ · [303. Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable) · [704. Binary Search](https://leetcode.com/problems/binary-search) · [1378. Replace Employee ID With The Unique Identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier) · [69. Sqrt(x)](https://leetcode.com/problems/sqrtx) ✅ · [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree) · [196. Delete Duplicate Emails](https://leetcode.com/problems/delete-duplicate-emails) · [231. Power of Two](https://leetcode.com/problems/power-of-two) · [232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks) · [234. Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list) · [258. Add Digits](https://leetcode.com/problems/add-digits) · [392. Is Subsequence](https://leetcode.com/problems/is-subsequence) · [541. Reverse String II](https://leetcode.com/problems/reverse-string-ii) · [628. Maximum Product of Three Numbers](https://leetcode.com/problems/maximum-product-of-three-numbers) · [977. Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) ✅ · [1431. Kids With the Greatest Number of Candies](https://leetcode.com/problems/kids-with-the-greatest-number-of-candies) · [1929. Concatenation of Array](https://leetcode.com/problems/concatenation-of-array) · [3184. Count Pairs That Form a Complete Day I](https://leetcode.com/problems/count-pairs-that-form-a-complete-day-i)
- **Medium (93):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [134. Gas Station](https://leetcode.com/problems/gas-station) · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [322. Coin Change](https://leetcode.com/problems/coin-change) · [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) · [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) · [1124. Longest Well-Performing Interval](https://leetcode.com/problems/longest-well-performing-interval) · [1540. Can Convert String in K Moves](https://leetcode.com/problems/can-convert-string-in-k-moves) · [1798. Maximum Number of Consecutive Values You Can Make](https://leetcode.com/problems/maximum-number-of-consecutive-values-you-can-make) · [1946. Largest Number After Mutating Substring](https://leetcode.com/problems/largest-number-after-mutating-substring) · [2233. Maximum Product After K Increments](https://leetcode.com/problems/maximum-product-after-k-increments) · [2445. Number of Nodes With Value One](https://leetcode.com/problems/number-of-nodes-with-value-one) · [2457. Minimum Addition to Make Integer Beautiful](https://leetcode.com/problems/minimum-addition-to-make-integer-beautiful) · [2597. The Number of Beautiful Subsets](https://leetcode.com/problems/the-number-of-beautiful-subsets) · [2829. Determine the Minimum Sum of a k-avoiding Array](https://leetcode.com/problems/determine-the-minimum-sum-of-a-k-avoiding-array) · [2834. Find the Minimum Possible Sum of a Beautiful Array](https://leetcode.com/problems/find-the-minimum-possible-sum-of-a-beautiful-array) · [3457. Eat Pizzas!](https://leetcode.com/problems/eat-pizzas) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water) · [15. 3Sum](https://leetcode.com/problems/3sum) ✅ · [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses) · [31. Next Permutation](https://leetcode.com/problems/next-permutation) ✅ · [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) · [48. Rotate Image](https://leetcode.com/problems/rotate-image) ✅ · [56. Merge Intervals](https://leetcode.com/problems/merge-intervals) · [75. Sort Colors](https://leetcode.com/problems/sort-colors) ✅ · [151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) ✅ · [162. Find Peak Element](https://leetcode.com/problems/find-peak-element) · [179. Largest Number](https://leetcode.com/problems/largest-number) ✅ · [198. House Robber](https://leetcode.com/problems/house-robber) · [200. Number of Islands](https://leetcode.com/problems/number-of-islands) · [319. Bulb Switcher](https://leetcode.com/problems/bulb-switcher) · [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges) · [3653. XOR After Range Multiplication Queries I](https://leetcode.com/problems/xor-after-range-multiplication-queries-i) · [12. Integer to Roman](https://leetcode.com/problems/integer-to-roman) ✅ · [18. 4Sum](https://leetcode.com/problems/4sum) · [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) · [50. Pow(x, n)](https://leetcode.com/problems/powx-n) · [54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix) ✅ · [55. Jump Game](https://leetcode.com/problems/jump-game) · [78. Subsets](https://leetcode.com/problems/subsets) · [92. Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii) · [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) ✅ · [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array) · [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) ✅ · [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) · [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements) · [912. Sort an Array](https://leetcode.com/problems/sort-an-array) · [6. Zigzag Conversion](https://leetcode.com/problems/zigzag-conversion) · [8. String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) ✅ · [39. Combination Sum](https://leetcode.com/problems/combination-sum) · [40. Combination Sum II](https://leetcode.com/problems/combination-sum-ii) · [46. Permutations](https://leetcode.com/problems/permutations) ✅ · [72. Edit Distance](https://leetcode.com/problems/edit-distance) · [73. Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes) · [77. Combinations](https://leetcode.com/problems/combinations) · [90. Subsets II](https://leetcode.com/problems/subsets-ii) · [120. Triangle](https://leetcode.com/problems/triangle) · [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) ✅ · [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii) · [143. Reorder List](https://leetcode.com/problems/reorder-list) · [150. Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation) · [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array) · [155. Min Stack](https://leetcode.com/problems/min-stack) ✅ · [167. Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted) · [213. House Robber II](https://leetcode.com/problems/house-robber-ii) · [328. Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list) · [450. Delete Node in a BST](https://leetcode.com/problems/delete-node-in-a-bst) · [516. Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence) · [525. Contiguous Array](https://leetcode.com/problems/contiguous-array) · [532. K-diff Pairs in an Array](https://leetcode.com/problems/k-diff-pairs-in-an-array) · [556. Next Greater Element III](https://leetcode.com/problems/next-greater-element-iii) · [583. Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings) · [658. Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements) · [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures) · [767. Reorganize String](https://leetcode.com/problems/reorganize-string) · [840. Magic Squares In Grid](https://leetcode.com/problems/magic-squares-in-grid) · [841. Keys and Rooms](https://leetcode.com/problems/keys-and-rooms) · [853. Car Fleet](https://leetcode.com/problems/car-fleet) · [945. Minimum Increment to Make Array Unique](https://leetcode.com/problems/minimum-increment-to-make-array-unique) · [1004. Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii) · [1094. Car Pooling](https://leetcode.com/problems/car-pooling) · [1376. Time Needed to Inform All Employees](https://leetcode.com/problems/time-needed-to-inform-all-employees) · [1498. Number of Subsequences That Satisfy the Given Sum Condition](https://leetcode.com/problems/number-of-subsequences-that-satisfy-the-given-sum-condition) · [1838. Frequency of the Most Frequent Element](https://leetcode.com/problems/frequency-of-the-most-frequent-element) · [1910. Remove All Occurrences of a Substring](https://leetcode.com/problems/remove-all-occurrences-of-a-substring) · [1922. Count Good Numbers](https://leetcode.com/problems/count-good-numbers) · [2149. Rearrange Array Elements by Sign](https://leetcode.com/problems/rearrange-array-elements-by-sign)
- **Hard (35):** [1872. Stone Game VIII](https://leetcode.com/problems/stone-game-viii) · [2338. Count the Number of Ideal Arrays](https://leetcode.com/problems/count-the-number-of-ideal-arrays) · [2382. Maximum Segment Sum After Removals](https://leetcode.com/problems/maximum-segment-sum-after-removals) · [2463. Minimum Total Distance Traveled](https://leetcode.com/problems/minimum-total-distance-traveled) · [2612. Minimum Reverse Operations](https://leetcode.com/problems/minimum-reverse-operations) · [2827. Number of Beautiful Integers in the Range](https://leetcode.com/problems/number-of-beautiful-integers-in-the-range) · [2940. Find Building Where Alice and Bob Can Meet](https://leetcode.com/problems/find-building-where-alice-and-bob-can-meet) · [3165. Maximum Sum of Subsequence With Non-adjacent Elements](https://leetcode.com/problems/maximum-sum-of-subsequence-with-non-adjacent-elements) · [3336. Find the Number of Subsequences With Equal GCD](https://leetcode.com/problems/find-the-number-of-subsequences-with-equal-gcd) · [3449. Maximize the Minimum Game Score](https://leetcode.com/problems/maximize-the-minimum-game-score) · [3575. Maximum Good Subtree Score](https://leetcode.com/problems/maximum-good-subtree-score) · [3640. Trionic Array II](https://leetcode.com/problems/trionic-array-ii) · [3661. Maximum Walls Destroyed by Robots](https://leetcode.com/problems/maximum-walls-destroyed-by-robots) · [41. First Missing Positive](https://leetcode.com/problems/first-missing-positive) · [3655. XOR After Range Multiplication Queries II](https://leetcode.com/problems/xor-after-range-multiplication-queries-ii) · [3671. Sum of Beautiful Subsequences](https://leetcode.com/problems/sum-of-beautiful-subsequences) · [3901. Good Subsequence Queries](https://leetcode.com/problems/good-subsequence-queries) · [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) ✅ · [51. N-Queens](https://leetcode.com/problems/n-queens) · [10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) · [25. Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group) · [30. Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words) · [32. Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses) · [44. Wildcard Matching](https://leetcode.com/problems/wildcard-matching) · [76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring) · [84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram) · [135. Candy](https://leetcode.com/problems/candy) · [410. Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum) · [992. Subarrays with K Different Integers](https://leetcode.com/problems/subarrays-with-k-different-integers) · [1235. Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling) · [1819. Number of Different Subsequences GCDs](https://leetcode.com/problems/number-of-different-subsequences-gcds) · [2163. Minimum Difference in Sums After Removal of Elements](https://leetcode.com/problems/minimum-difference-in-sums-after-removal-of-elements) · [2872. Maximum Number of K-Divisible Components](https://leetcode.com/problems/maximum-number-of-k-divisible-components) · [3539. Find Sum of Array Product of Magical Sequences](https://leetcode.com/problems/find-sum-of-array-product-of-magical-sequences) · [3666. Minimum Operations to Equalize Binary String](https://leetcode.com/problems/minimum-operations-to-equalize-binary-string)

### Accenture (142)

- **Easy (64):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number) · [2859. Sum of Values at Indices With K Set Bits](https://leetcode.com/problems/sum-of-values-at-indices-with-k-set-bits) · [3000. Maximum Area of Longest Diagonal Rectangle](https://leetcode.com/problems/maximum-area-of-longest-diagonal-rectangle) · [3146. Permutation Difference between Two Strings](https://leetcode.com/problems/permutation-difference-between-two-strings) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [169. Majority Element](https://leetcode.com/problems/majority-element) ✅ · [202. Happy Number](https://leetcode.com/problems/happy-number) · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [1661. Average Time of Process per Machine](https://leetcode.com/problems/average-time-of-process-per-machine) · [2099. Find Subsequence of Length K With the Largest Sum](https://leetcode.com/problems/find-subsequence-of-length-k-with-the-largest-sum) · [2855. Minimum Right Shifts to Sort the Array](https://leetcode.com/problems/minimum-right-shifts-to-sort-the-array) · [2960. Count Tested Devices After Test Operations](https://leetcode.com/problems/count-tested-devices-after-test-operations) · [3028. Ant on the Boundary](https://leetcode.com/problems/ant-on-the-boundary) · [3345. Smallest Divisible Digit Product I](https://leetcode.com/problems/smallest-divisible-digit-product-i) · [13. Roman to Integer](https://leetcode.com/problems/roman-to-integer) ✅ · [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) ✅ · [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) ✅ · [66. Plus One](https://leetcode.com/problems/plus-one) ✅ · [118. Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) ✅ · [136. Single Number](https://leetcode.com/problems/single-number) · [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate) ✅ · [344. Reverse String](https://leetcode.com/problems/reverse-string) · [349. Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) · [455. Assign Cookies](https://leetcode.com/problems/assign-cookies) · [485. Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) ✅ · [876. Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list) · [1148. Article Views I](https://leetcode.com/problems/article-views-i) · [1518. Water Bottles](https://leetcode.com/problems/water-bottles) · [3432. Count Partitions with Even Sum Difference](https://leetcode.com/problems/count-partitions-with-even-sum-difference) · [35. Search Insert Position](https://leetcode.com/problems/search-insert-position) ✅ · [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) ✅ · [181. Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers) · [197. Rising Temperature](https://leetcode.com/problems/rising-temperature) · [219. Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii) · [496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i) · [100. Same Tree](https://leetcode.com/problems/same-tree) · [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree) · [108. Convert Sorted Array to Binary Search Tree](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree) · [190. Reverse Bits](https://leetcode.com/problems/reverse-bits) · [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list) ✅ · [345. Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string) · [387. First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string) ✅ · [389. Find the Difference](https://leetcode.com/problems/find-the-difference) · [409. Longest Palindrome](https://leetcode.com/problems/longest-palindrome) · [507. Perfect Number](https://leetcode.com/problems/perfect-number) · [541. Reverse String II](https://leetcode.com/problems/reverse-string-ii) · [724. Find Pivot Index](https://leetcode.com/problems/find-pivot-index) · [796. Rotate String](https://leetcode.com/problems/rotate-string) · [977. Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) ✅ · [1512. Number of Good Pairs](https://leetcode.com/problems/number-of-good-pairs) · [1636. Sort Array by Increasing Frequency](https://leetcode.com/problems/sort-array-by-increasing-frequency) · [1742. Maximum Number of Balls in a Box](https://leetcode.com/problems/maximum-number-of-balls-in-a-box) · [1903. Largest Odd Number in String](https://leetcode.com/problems/largest-odd-number-in-string) · [2011. Final Value of Variable After Performing Operations](https://leetcode.com/problems/final-value-of-variable-after-performing-operations) · [2215. Find the Difference of Two Arrays](https://leetcode.com/problems/find-the-difference-of-two-arrays) · [2235. Add Two Integers](https://leetcode.com/problems/add-two-integers)
- **Medium (67):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number) ✅ · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [322. Coin Change](https://leetcode.com/problems/coin-change) · [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) · [740. Delete and Earn](https://leetcode.com/problems/delete-and-earn) · [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) · [3779. Minimum Number of Operations to Have Distinct Elements](https://leetcode.com/problems/minimum-number-of-operations-to-have-distinct-elements) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water) · [15. 3Sum](https://leetcode.com/problems/3sum) ✅ · [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list) · [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses) · [31. Next Permutation](https://leetcode.com/problems/next-permutation) ✅ · [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) · [48. Rotate Image](https://leetcode.com/problems/rotate-image) ✅ · [56. Merge Intervals](https://leetcode.com/problems/merge-intervals) · [151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) ✅ · [162. Find Peak Element](https://leetcode.com/problems/find-peak-element) · [179. Largest Number](https://leetcode.com/problems/largest-number) ✅ · [198. House Robber](https://leetcode.com/problems/house-robber) · [200. Number of Islands](https://leetcode.com/problems/number-of-islands) · [204. Count Primes](https://leetcode.com/problems/count-primes) ✅ · [279. Perfect Squares](https://leetcode.com/problems/perfect-squares) · [319. Bulb Switcher](https://leetcode.com/problems/bulb-switcher) · [400. Nth Digit](https://leetcode.com/problems/nth-digit) · [451. Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency) · [962. Maximum Width Ramp](https://leetcode.com/problems/maximum-width-ramp) · [12. Integer to Roman](https://leetcode.com/problems/integer-to-roman) ✅ · [18. 4Sum](https://leetcode.com/problems/4sum) · [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) · [50. Pow(x, n)](https://leetcode.com/problems/powx-n) · [54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix) ✅ · [62. Unique Paths](https://leetcode.com/problems/unique-paths) · [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array) · [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) ✅ · [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) · [343. Integer Break](https://leetcode.com/problems/integer-break) · [912. Sort an Array](https://leetcode.com/problems/sort-an-array) · [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) · [1823. Find the Winner of the Circular Game](https://leetcode.com/problems/find-the-winner-of-the-circular-game) ✅ · [2938. Separate Black and White Balls](https://leetcode.com/problems/separate-black-and-white-balls) · [8. String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) ✅ · [72. Edit Distance](https://leetcode.com/problems/edit-distance) · [79. Word Search](https://leetcode.com/problems/word-search) · [81. Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii) · [103. Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal) · [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) ✅ · [131. Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning) · [177. Nth Highest Salary](https://leetcode.com/problems/nth-highest-salary) · [178. Rank Scores](https://leetcode.com/problems/rank-scores) · [199. Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view) · [396. Rotate Function](https://leetcode.com/problems/rotate-function) · [402. Remove K Digits](https://leetcode.com/problems/remove-k-digits) · [438. Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string) · [516. Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence) · [525. Contiguous Array](https://leetcode.com/problems/contiguous-array) · [542. 01 Matrix](https://leetcode.com/problems/01-matrix) · [670. Maximum Swap](https://leetcode.com/problems/maximum-swap) · [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures) · [852. Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array) ✅ · [907. Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums) · [2684. Maximum Number of Moves in a Grid](https://leetcode.com/problems/maximum-number-of-moves-in-a-grid)
- **Hard (11):** [3348. Smallest Divisible Digit Product II](https://leetcode.com/problems/smallest-divisible-digit-product-ii) · [214. Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome) · [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) ✅ · [185. Department Top Three Salaries](https://leetcode.com/problems/department-top-three-salaries) · [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) ✅ · [51. N-Queens](https://leetcode.com/problems/n-queens) · [10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) · [32. Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses) · [135. Candy](https://leetcode.com/problems/candy) · [1312. Minimum Insertion Steps to Make a String Palindrome](https://leetcode.com/problems/minimum-insertion-steps-to-make-a-string-palindrome) · [2493. Divide Nodes Into the Maximum Number of Groups](https://leetcode.com/problems/divide-nodes-into-the-maximum-number-of-groups)

### Cognizant (46)

- **Easy (31):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [3392. Count Subarrays of Length Three With a Condition](https://leetcode.com/problems/count-subarrays-of-length-three-with-a-condition) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [169. Majority Element](https://leetcode.com/problems/majority-element) ✅ · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [3667. Sort Array By Absolute Value](https://leetcode.com/problems/sort-array-by-absolute-value) · [13. Roman to Integer](https://leetcode.com/problems/roman-to-integer) ✅ · [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) ✅ · [136. Single Number](https://leetcode.com/problems/single-number) · [175. Combine Two Tables](https://leetcode.com/problems/combine-two-tables) · [485. Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) ✅ · [1068. Product Sales Analysis I](https://leetcode.com/problems/product-sales-analysis-i) · [1148. Article Views I](https://leetcode.com/problems/article-views-i) · [35. Search Insert Position](https://leetcode.com/problems/search-insert-position) ✅ · [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) ✅ · [181. Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers) · [197. Rising Temperature](https://leetcode.com/problems/rising-temperature) · [412. Fizz Buzz](https://leetcode.com/problems/fizz-buzz) · [704. Binary Search](https://leetcode.com/problems/binary-search) · [1200. Minimum Absolute Difference](https://leetcode.com/problems/minimum-absolute-difference) · [1280. Students and Examinations](https://leetcode.com/problems/students-and-examinations) · [1295. Find Numbers with Even Number of Digits](https://leetcode.com/problems/find-numbers-with-even-number-of-digits) · [1378. Replace Employee ID With The Unique Identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier) · [1732. Find the Highest Altitude](https://leetcode.com/problems/find-the-highest-altitude)
- **Medium (13):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [2165. Smallest Value of the Rearranged Number](https://leetcode.com/problems/smallest-value-of-the-rearranged-number) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [31. Next Permutation](https://leetcode.com/problems/next-permutation) ✅ · [55. Jump Game](https://leetcode.com/problems/jump-game) · [443. String Compression](https://leetcode.com/problems/string-compression)
- **Hard (2):** [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) ✅ · [41. First Missing Positive](https://leetcode.com/problems/first-missing-positive)

### Capgemini (35)

- **Easy (17):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [2356. Number of Unique Subjects Taught by Each Teacher](https://leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher) · [3498. Reverse Degree of a String](https://leetcode.com/problems/reverse-degree-of-a-string) · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [3005. Count Elements With Maximum Frequency](https://leetcode.com/problems/count-elements-with-maximum-frequency) · [13. Roman to Integer](https://leetcode.com/problems/roman-to-integer) ✅ · [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) ✅ · [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) ✅ · [28. Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) · [66. Plus One](https://leetcode.com/problems/plus-one) ✅ · [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate) ✅ · [349. Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) · [2677. Chunk Array](https://leetcode.com/problems/chunk-array)
- **Medium (16):** [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [322. Coin Change](https://leetcode.com/problems/coin-change) · [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water) · [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) · [75. Sort Colors](https://leetcode.com/problems/sort-colors) ✅ · [626. Exchange Seats](https://leetcode.com/problems/exchange-seats) · [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) ✅ · [235. Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree) · [1405. Longest Happy String](https://leetcode.com/problems/longest-happy-string) · [1877. Minimize Maximum Pair Sum in Array](https://leetcode.com/problems/minimize-maximum-pair-sum-in-array)
- **Hard (2):** [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) ✅ · [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) ✅

### Accolite (20)

- **Easy (4):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs)
- **Medium (15):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [1234. Replace the Substring for Balanced String](https://leetcode.com/problems/replace-the-substring-for-balanced-string) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [2422. Merge Operations to Turn Array Into a Palindrome](https://leetcode.com/problems/merge-operations-to-turn-array-into-a-palindrome) · [3819. Rotate Non Negative Elements](https://leetcode.com/problems/rotate-non-negative-elements) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [134. Gas Station](https://leetcode.com/problems/gas-station) · [322. Coin Change](https://leetcode.com/problems/coin-change) · [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers) ✅ · [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water) · [735. Asteroid Collision](https://leetcode.com/problems/asteroid-collision) · [80. Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii) · [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) · [1654. Minimum Jumps to Reach Home](https://leetcode.com/problems/minimum-jumps-to-reach-home)
- **Hard (1):** [321. Create Maximum Number](https://leetcode.com/problems/create-maximum-number)

### Wipro (17)

- **Easy (10):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [242. Valid Anagram](https://leetcode.com/problems/valid-anagram) · [67. Add Binary](https://leetcode.com/problems/add-binary) ✅ · [118. Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) ✅ · [766. Toeplitz Matrix](https://leetcode.com/problems/toeplitz-matrix) · [2667. Create Hello World Function](https://leetcode.com/problems/create-hello-world-function)
- **Medium (7):** [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [3001. Minimum Moves to Capture The Queen](https://leetcode.com/problems/minimum-moves-to-capture-the-queen) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅ · [189. Rotate Array](https://leetcode.com/problems/rotate-array) ✅ · [56. Merge Intervals](https://leetcode.com/problems/merge-intervals) · [451. Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency)

### HCL (16)

- **Easy (7):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅ · [268. Missing Number](https://leetcode.com/problems/missing-number) · [344. Reverse String](https://leetcode.com/problems/reverse-string)
- **Medium (8):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [15. 3Sum](https://leetcode.com/problems/3sum) ✅ · [151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) ✅ · [209. Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum) · [647. Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings)
- **Hard (1):** [185. Department Top Three Salaries](https://leetcode.com/problems/department-top-three-salaries)

### Persistent Systems (10)

- **Easy (6):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [2220. Minimum Bit Flips to Convert Number](https://leetcode.com/problems/minimum-bit-flips-to-convert-number) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [14. Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) ✅ · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses) · [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) ✅
- **Medium (4):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray) · [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [49. Group Anagrams](https://leetcode.com/problems/group-anagrams) ✅

### LTI (8)

- **Easy (3):** [283. Move Zeroes](https://leetcode.com/problems/move-zeroes) · [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome) · [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number)
- **Medium (4):** [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [2271. Maximum White Tiles Covered by a Carpet](https://leetcode.com/problems/maximum-white-tiles-covered-by-a-carpet) · [176. Second Highest Salary](https://leetcode.com/problems/second-highest-salary) · [2747. Count Zero Request Servers](https://leetcode.com/problems/count-zero-request-servers)
- **Hard (1):** [1755. Closest Subsequence Sum](https://leetcode.com/problems/closest-subsequence-sum)

### Mindtree (5)

- **Easy (4):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [2180. Count Integers With Even Digit Sum](https://leetcode.com/problems/count-integers-with-even-digit-sum) · [9. Palindrome Number](https://leetcode.com/problems/palindrome-number) ✅ · [1661. Average Time of Process per Machine](https://leetcode.com/problems/average-time-of-process-per-machine)
- **Medium (1):** [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k)

### Amdocs (5)

- **Easy (2):** [3875. Construct Uniform Parity Array I](https://leetcode.com/problems/construct-uniform-parity-array-i) · [136. Single Number](https://leetcode.com/problems/single-number)
- **Medium (3):** [2950. Number of Divisible Substrings](https://leetcode.com/problems/number-of-divisible-substrings) · [3391. Design a 3D Binary Matrix with Efficient Layer Tracking](https://leetcode.com/problems/design-a-3d-binary-matrix-with-efficient-layer-tracking) · [3876. Construct Uniform Parity Array II](https://leetcode.com/problems/construct-uniform-parity-array-ii)

### Tech Mahindra (4)

- **Easy (2):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) ✅
- **Medium (2):** [7. Reverse Integer](https://leetcode.com/problems/reverse-integer) ✅ · [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray)

### Altimetrik (4)

- **Easy (3):** [1. Two Sum](https://leetcode.com/problems/two-sum) · [2341. Maximum Number of Pairs in Array](https://leetcode.com/problems/maximum-number-of-pairs-in-array) · [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses)
- **Medium (1):** [24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs)

### ADP (2)

- **Easy (1):** [283. Move Zeroes](https://leetcode.com/problems/move-zeroes)
- **Hard (1):** [3463. Check If Digits Are Equal in String After Operations II](https://leetcode.com/problems/check-if-digits-are-equal-in-string-after-operations-ii)

### Larsen & Toubro (2)

- **Easy (2):** [3079. Find the Sum of Encrypted Integers](https://leetcode.com/problems/find-the-sum-of-encrypted-integers) · [3105. Longest Strictly Increasing or Strictly Decreasing Subarray](https://leetcode.com/problems/longest-strictly-increasing-or-strictly-decreasing-subarray)

### Mphasis (1)

- **Medium (1):** [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) ✅

### Brillio (1)

- **Medium (1):** [1551. Minimum Operations to Make Array Equal](https://leetcode.com/problems/minimum-operations-to-make-array-equal)

## Part 3 — The 44 questions solved in Java

Each one reads its input from standard input (`Scanner`) and prints the answer, exactly like the app's tests. Try it yourself first — the approach and the code are here to check against.

[9. Palindrome Number](#9-palindrome-number) (Easy) · [13. Roman to Integer](#13-roman-to-integer) (Easy) · [14. Longest Common Prefix](#14-longest-common-prefix) (Easy) · [21. Merge Two Sorted Lists](#21-merge-two-sorted-lists) (Easy) · [26. Remove Duplicates from Sorted Array](#26-remove-duplicates-from-sorted-array) (Easy) · [35. Search Insert Position](#35-search-insert-position) (Easy) · [66. Plus One](#66-plus-one) (Easy) · [67. Add Binary](#67-add-binary) (Easy) · [69. Sqrt(x)](#69-sqrtx) (Easy) · [88. Merge Sorted Array](#88-merge-sorted-array) (Easy) · [118. Pascal's Triangle](#118-pascals-triangle) (Easy) · [121. Best Time to Buy and Sell Stock](#121-best-time-to-buy-and-sell-stock) (Easy) · [141. Linked List Cycle](#141-linked-list-cycle) (Easy) · [169. Majority Element](#169-majority-element) (Easy) · [206. Reverse Linked List](#206-reverse-linked-list) (Easy) · [217. Contains Duplicate](#217-contains-duplicate) (Easy) · [387. First Unique Character in a String](#387-first-unique-character-in-a-string) (Easy) · [485. Max Consecutive Ones](#485-max-consecutive-ones) (Easy) · [977. Squares of a Sorted Array](#977-squares-of-a-sorted-array) (Easy) · [2. Add Two Numbers (Linked Lists)](#2-add-two-numbers-linked-lists) (Medium) · [5. Longest Palindromic Substring](#5-longest-palindromic-substring) (Medium) · [7. Reverse Integer](#7-reverse-integer) (Medium) · [8. String to Integer (atoi)](#8-string-to-integer-atoi) (Medium) · [12. Integer to Roman](#12-integer-to-roman) (Medium) · [15. 3Sum](#15-3sum) (Medium) · [17. Letter Combinations of a Phone Number](#17-letter-combinations-of-a-phone-number) (Medium) · [31. Next Permutation](#31-next-permutation) (Medium) · [46. Permutations](#46-permutations) (Medium) · [48. Rotate Image](#48-rotate-image) (Medium) · [49. Group Anagrams](#49-group-anagrams) (Medium) · [54. Spiral Matrix](#54-spiral-matrix) (Medium) · [75. Sort Colors (Dutch National Flag)](#75-sort-colors-dutch-national-flag) (Medium) · [122. Best Time to Buy and Sell Stock II](#122-best-time-to-buy-and-sell-stock-ii) (Medium) · [128. Longest Consecutive Sequence](#128-longest-consecutive-sequence) (Medium) · [151. Reverse Words in a String](#151-reverse-words-in-a-string) (Medium) · [155. Min Stack](#155-min-stack) (Medium) · [179. Largest Number](#179-largest-number) (Medium) · [189. Rotate Array](#189-rotate-array) (Medium) · [204. Count Primes](#204-count-primes) (Medium) · [238. Product of Array Except Self](#238-product-of-array-except-self) (Medium) · [852. Peak Index in a Mountain Array](#852-peak-index-in-a-mountain-array) (Medium) · [1823. Find the Winner of the Circular Game](#1823-find-the-winner-of-the-circular-game) (Medium) · [4. Median of Two Sorted Arrays](#4-median-of-two-sorted-arrays) (Hard) · [42. Trapping Rain Water](#42-trapping-rain-water) (Hard)

### 9. Palindrome Number

**Easy** · Math · [LeetCode](https://leetcode.com/problems/palindrome-number) · asked at Cognizant, Accenture, TCS, Infosys, Capgemini, Wipro, Persistent Systems, Mindtree

Print YES if the integer X reads the same forwards and backwards, otherwise NO. Solve it without converting X to a string.

**Input:** A single integer X.  
**Output:** YES or NO

**Constraints:** −2^31 ≤ X ≤ 2^31 − 1

**Example 1**

| Input | Output |
|---|---|
| <pre>121</pre> | <pre>YES</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>-121</pre> | <pre>NO</pre> |

<details><summary>Hints</summary>

1. Negative numbers are never palindromes; neither are numbers ending in 0 (except 0).
2. Reverse only the second half of the digits and compare with the first half — no overflow possible.

</details>

**Approach:** Build the reversed lower half until it is ≥ the remaining upper half; compare (dropping the middle digit for odd lengths).

**Complexity:** time O(log X) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static boolean isPalindrome(int x) {
        if (x < 0 || (x % 10 == 0 && x != 0)) return false;
        int rev = 0;
        while (x > rev) {
            rev = rev * 10 + x % 10;
            x /= 10;
        }
        return x == rev || x == rev / 10;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(isPalindrome(sc.nextInt()) ? "YES" : "NO");
    }
}
```

</details>

### 13. Roman to Integer

**Easy** · Strings · [LeetCode](https://leetcode.com/problems/roman-to-integer) · asked at Accenture, Capgemini, Cognizant, TCS, Infosys

Convert a valid Roman numeral (I=1, V=5, X=10, L=50, C=100, D=500, M=1000; subtractive forms IV, IX, XL, XC, CD, CM) to an integer.

**Input:** A single Roman numeral.  
**Output:** Its integer value

**Constraints:** Value in [1, 3999]

**Example 1**

| Input | Output |
|---|---|
| <pre>III</pre> | <pre>3</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>LVIII</pre> | <pre>58</pre> |

<details><summary>Hints</summary>

1. If a symbol is smaller than the one after it, subtract it; otherwise add it.

</details>

**Approach:** Left-to-right scan with a one-symbol lookahead: subtract when value(s[i]) < value(s[i+1]).

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int value(char c) {
        switch (c) {
            case 'I': return 1;
            case 'V': return 5;
            case 'X': return 10;
            case 'L': return 50;
            case 'C': return 100;
            case 'D': return 500;
            default: return 1000;
        }
    }

    static int romanToInt(String s) {
        int total = 0;
        for (int i = 0; i < s.length(); i++) {
            int v = value(s.charAt(i));
            if (i + 1 < s.length() && v < value(s.charAt(i + 1))) total -= v;
            else total += v;
        }
        return total;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(romanToInt(sc.next()));
    }
}
```

</details>

### 14. Longest Common Prefix

**Easy** · Strings · [LeetCode](https://leetcode.com/problems/longest-common-prefix) · asked at Wipro, Infosys, Persistent Systems, Accenture, Capgemini, TCS

Print the longest common prefix of N words, wrapped in double quotes (so an empty prefix prints as "").

**Input:** Line 1: N · Line 2: N words  
**Output:** The prefix in double quotes

**Constraints:** 1 ≤ N ≤ 200 · 1 ≤ |word| ≤ 200

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>flower flow flight</pre> | <pre>"fl"</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3<br>dog racecar car</pre> | <pre>""</pre> |

<details><summary>Hints</summary>

1. Vertical scan: compare column c of every word with the first word.
2. Stop at the first mismatch or when any word ends.

</details>

**Approach:** Vertical scanning bounded by the shortest word. Alternative: sort and compare only the first and last words.

**Complexity:** time O(total characters) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String longestCommonPrefix(String[] w) {
        for (int c = 0; c < w[0].length(); c++) {
            char ch = w[0].charAt(c);
            for (int i = 1; i < w.length; i++) {
                if (c >= w[i].length() || w[i].charAt(c) != ch) return w[0].substring(0, c);
            }
        }
        return w[0];
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] w = new String[n];
        for (int i = 0; i < n; i++) w[i] = sc.next();
        System.out.println("\"" + longestCommonPrefix(w) + "\"");
    }
}
```

</details>

### 21. Merge Two Sorted Lists

**Easy** · Linked List · [LeetCode](https://leetcode.com/problems/merge-two-sorted-lists) · asked at Capgemini, Infosys, Accenture, TCS

Merge two sorted linked lists into one sorted list by splicing their nodes together. The program prints the merged values.

**Input:** Line 1: N · Line 2: N sorted values (may be empty) · Line 3: M · Line 4: M sorted values (may be empty)  
**Output:** Merged values

**Constraints:** 0 ≤ N, M ≤ 50 · 1 ≤ N + M

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 2 4<br>3<br>1 3 4</pre> | <pre>1 1 2 3 4 4</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>0<br><br>1<br>0</pre> | <pre>0</pre> |

<details><summary>Hints</summary>

1. Use a dummy head and a tail pointer.
2. Attach the smaller head each step; attach the leftover list at the end.

</details>

**Approach:** Dummy-node merge — the same merge step as merge sort, reusing existing nodes.

**Complexity:** time O(N + M) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static class ListNode {
        int val;
        ListNode next;
        ListNode(int val) { this.val = val; }
    }

    static ListNode merge(ListNode a, ListNode b) {
        ListNode dummy = new ListNode(0), tail = dummy;
        while (a != null && b != null) {
            if (a.val <= b.val) { tail.next = a; a = a.next; }
            else { tail.next = b; b = b.next; }
            tail = tail.next;
        }
        tail.next = (a != null) ? a : b;
        return dummy.next;
    }

    static ListNode read(Scanner sc) {
        int n = sc.nextInt();
        ListNode dummy = new ListNode(0), tail = dummy;
        for (int i = 0; i < n; i++) tail = tail.next = new ListNode(sc.nextInt());
        return dummy.next;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        ListNode a = read(sc), b = read(sc);
        StringJoiner sj = new StringJoiner(" ");
        for (ListNode p = merge(a, b); p != null; p = p.next) sj.add(String.valueOf(p.val));
        System.out.println(sj);
    }
}
```

</details>

### 26. Remove Duplicates from Sorted Array

**Easy** · Two Pointers · [LeetCode](https://leetcode.com/problems/remove-duplicates-from-sorted-array) · asked at Accenture, Capgemini, Cognizant, TCS, Infosys

Remove duplicates from a sorted array IN PLACE so each value appears once, keeping order. Return K, the number of unique values. The program prints K, then the first K elements.

**Input:** Line 1: N · Line 2: N sorted integers  
**Output:** Line 1: K · Line 2: first K elements

**Constraints:** 1 ≤ N ≤ 3·10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 1 2</pre> | <pre>2<br>1 2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>10<br>0 0 1 1 1 2 2 3 3 4</pre> | <pre>5<br>0 1 2 3 4</pre> |

<details><summary>Hints</summary>

1. Slow pointer = next write position; fast pointer scans.
2. Write a[fast] only when it differs from the last written value.

</details>

**Approach:** Read/write two pointers; since the array is sorted, duplicates are adjacent.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int removeDuplicates(int[] a) {
        int k = 1;
        for (int i = 1; i < a.length; i++) {
            if (a[i] != a[k - 1]) a[k++] = a[i];
        }
        return k;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        int k = removeDuplicates(a);
        StringJoiner sj = new StringJoiner(" ");
        for (int i = 0; i < k; i++) sj.add(String.valueOf(a[i]));
        System.out.println(k + "\n" + sj);
    }
}
```

</details>

### 35. Search Insert Position

**Easy** · Binary Search · [LeetCode](https://leetcode.com/problems/search-insert-position) · asked at Cognizant, Accenture, TCS

Given a sorted array of DISTINCT integers and a target X, print X’s index if present, otherwise the index where it would be inserted to keep the array sorted. O(log N).

**Input:** Line 1: N · Line 2: N sorted integers · Line 3: X  
**Output:** Index

**Constraints:** 1 ≤ N ≤ 10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>4<br>1 3 5 6<br>5</pre> | <pre>2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>4<br>1 3 5 6<br>2</pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. This is exactly the lower bound: first index with a[i] ≥ X.

</details>

**Approach:** Lower-bound binary search on the half-open range [0, N].

**Complexity:** time O(log N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int searchInsert(int[] a, int x) {
        int lo = 0, hi = a.length;
        while (lo < hi) {
            int mid = (lo + hi) >>> 1;
            if (a[mid] < x) lo = mid + 1;
            else hi = mid;
        }
        return lo;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(searchInsert(a, sc.nextInt()));
    }
}
```

</details>

### 66. Plus One

**Easy** · Arrays · [LeetCode](https://leetcode.com/problems/plus-one) · asked at Accenture, Capgemini, TCS

A large non-negative integer is given as its digits (most significant first). Add one and print the resulting digits.

**Input:** Line 1: N · Line 2: N digits  
**Output:** Digits of the result

**Constraints:** 1 ≤ N ≤ 100 · No leading zeros

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 2 3</pre> | <pre>1 2 4</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>4<br>4 3 2 1</pre> | <pre>4 3 2 2</pre> |

<details><summary>Hints</summary>

1. Walk from the last digit: if it is < 9, increment and stop; otherwise set it to 0 and carry.
2. All nines → a new array 1 followed by zeros.

</details>

**Approach:** Carry propagation from the right; only the all-9s case needs a longer array.

**Complexity:** time O(N) · space O(1) (O(N) when growing)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int[] plusOne(int[] d) {
        for (int i = d.length - 1; i >= 0; i--) {
            if (d[i] < 9) {
                d[i]++;
                return d;
            }
            d[i] = 0;
        }
        int[] out = new int[d.length + 1];
        out[0] = 1;
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] d = new int[n];
        for (int i = 0; i < n; i++) d[i] = sc.nextInt();
        StringJoiner sj = new StringJoiner(" ");
        for (int v : plusOne(d)) sj.add(String.valueOf(v));
        System.out.println(sj);
    }
}
```

</details>

### 67. Add Binary

**Easy** · Strings · [LeetCode](https://leetcode.com/problems/add-binary) · asked at Wipro, TCS, Infosys

Given two binary strings A and B, print their sum as a binary string. The strings can be long — do not convert them to integers.

**Input:** Line 1: A · Line 2: B  
**Output:** A + B in binary

**Constraints:** 1 ≤ |A|, |B| ≤ 10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>11<br>1</pre> | <pre>100</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1010<br>1011</pre> | <pre>10101</pre> |

<details><summary>Hints</summary>

1. Add from the rightmost characters with a carry, like decimal addition.
2. Build the result with StringBuilder and reverse it at the end.

</details>

**Approach:** Two indices from the end plus a carry; append (sum % 2), carry = sum / 2.

**Complexity:** time O(max(|A|, |B|)) · space O(max(|A|, |B|))

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String addBinary(String a, String b) {
        StringBuilder sb = new StringBuilder();
        int i = a.length() - 1, j = b.length() - 1, carry = 0;
        while (i >= 0 || j >= 0 || carry > 0) {
            int sum = carry;
            if (i >= 0) sum += a.charAt(i--) - '0';
            if (j >= 0) sum += b.charAt(j--) - '0';
            sb.append(sum % 2);
            carry = sum / 2;
        }
        return sb.reverse().toString();
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String a = sc.next(), b = sc.next();
        System.out.println(addBinary(a, b));
    }
}
```

</details>

### 69. Sqrt(x)

**Easy** · Binary Search · [LeetCode](https://leetcode.com/problems/sqrtx) · asked at TCS, Infosys

Print ⌊√X⌋ without using Math.sqrt or pow.

**Input:** A single integer X.  
**Output:** Integer square root

**Constraints:** 0 ≤ X ≤ 2^31 − 1

**Example 1**

| Input | Output |
|---|---|
| <pre>4</pre> | <pre>2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>8</pre> | <pre>2</pre> |

<details><summary>Hints</summary>

1. Binary search the largest m with m·m ≤ X.
2. Use long for m·m (or compare m ≤ X / m).

</details>

**Approach:** Binary search on the answer in [0, X] with overflow-safe comparison.

**Complexity:** time O(log X) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int mySqrt(int x) {
        long lo = 0, hi = x;
        while (lo < hi) {
            long mid = (lo + hi + 1) / 2;
            if (mid * mid <= x) lo = mid;
            else hi = mid - 1;
        }
        return (int) lo;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(mySqrt(sc.nextInt()));
    }
}
```

</details>

### 88. Merge Sorted Array

**Easy** · Two Pointers · [LeetCode](https://leetcode.com/problems/merge-sorted-array) · asked at Cognizant, HCL, Accenture, Wipro, Persistent Systems, TCS, Infosys

Arrays A (M elements) and B (N elements) are sorted. A has room for M + N elements. Merge B into A IN PLACE so A is sorted, then print A.

**Input:** Line 1: M N · Line 2: M integers (may be empty) · Line 3: N integers (may be empty)  
**Output:** The merged array

**Constraints:** 0 ≤ M, N ≤ 200 · 1 ≤ M + N

**Example 1**

| Input | Output |
|---|---|
| <pre>3 3<br>1 2 3<br>2 5 6</pre> | <pre>1 2 2 3 5 6</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1 0<br>1<br></pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. Filling from the front would overwrite unread values of A.
2. Fill from the BACK: compare the largest remaining of each array.

</details>

**Approach:** Three pointers from the end (i in A, j in B, k write position). Leftover B elements are copied; leftover A elements are already in place.

**Complexity:** time O(M + N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    // a has length m + n; its first m values are the real elements
    static void merge(int[] a, int m, int[] b, int n) {
        int i = m - 1, j = n - 1, k = m + n - 1;
        while (j >= 0) {
            if (i >= 0 && a[i] > b[j]) a[k--] = a[i--];
            else a[k--] = b[j--];
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int m = sc.nextInt(), n = sc.nextInt();
        int[] a = new int[m + n], b = new int[n];
        for (int i = 0; i < m; i++) a[i] = sc.nextInt();
        for (int i = 0; i < n; i++) b[i] = sc.nextInt();
        merge(a, m, b, n);
        StringJoiner sj = new StringJoiner(" ");
        for (int x : a) sj.add(String.valueOf(x));
        System.out.println(sj);
    }
}
```

</details>

### 118. Pascal's Triangle

**Easy** · Arrays · [LeetCode](https://leetcode.com/problems/pascals-triangle) · asked at Wipro, Accenture, TCS, Infosys

Print the first R rows of Pascal's triangle, one row per line.

**Input:** A single integer R.  
**Output:** R lines

**Constraints:** 1 ≤ R ≤ 30

**Example 1**

| Input | Output |
|---|---|
| <pre>5</pre> | <pre>1<br>1 1<br>1 2 1<br>1 3 3 1<br>1 4 6 4 1</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1</pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. Each row starts and ends with 1.
2. row[j] = prev[j−1] + prev[j].

</details>

**Approach:** Build each row from the previous one.

**Complexity:** time O(R²) · space O(R²)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static List<List<Integer>> generate(int rows) {
        List<List<Integer>> out = new ArrayList<>();
        for (int r = 0; r < rows; r++) {
            List<Integer> row = new ArrayList<>();
            for (int j = 0; j <= r; j++) {
                if (j == 0 || j == r) row.add(1);
                else row.add(out.get(r - 1).get(j - 1) + out.get(r - 1).get(j));
            }
            out.add(row);
        }
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        StringBuilder sb = new StringBuilder();
        for (List<Integer> row : generate(sc.nextInt())) {
            StringJoiner sj = new StringJoiner(" ");
            for (int v : row) sj.add(String.valueOf(v));
            sb.append(sj).append('\n');
        }
        System.out.print(sb);
    }
}
```

</details>

### 121. Best Time to Buy and Sell Stock

**Easy** · Arrays · [LeetCode](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) · asked at Tech Mahindra, Accenture, Infosys, HCL, Capgemini, Cognizant, TCS, Accolite

prices[i] is a stock’s price on day i. Choose one day to buy and a LATER day to sell. Print the maximum profit, or 0 if no profit is possible.

**Input:** Line 1: N · Line 2: N prices  
**Output:** Maximum profit

**Constraints:** 1 ≤ N ≤ 10^5 · 0 ≤ price ≤ 10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>7 1 5 3 6 4</pre> | <pre>5</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>5<br>7 6 4 3 1</pre> | <pre>0</pre> |

<details><summary>Hints</summary>

1. For each day, the best buy is the minimum price seen so far.

</details>

**Approach:** Single pass tracking the running minimum; profit if sold today = price − min so far.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int maxProfit(int[] prices) {
        int min = Integer.MAX_VALUE, best = 0;
        for (int p : prices) {
            min = Math.min(min, p);
            best = Math.max(best, p - min);
        }
        return best;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] p = new int[n];
        for (int i = 0; i < n; i++) p[i] = sc.nextInt();
        System.out.println(maxProfit(p));
    }
}
```

</details>

### 141. Linked List Cycle

**Easy** · Linked List · [LeetCode](https://leetcode.com/problems/linked-list-cycle) · asked at Cognizant, Infosys, Accenture, TCS

The program builds a linked list of N nodes; if POS ≥ 0 the tail’s next pointer is connected to the node at index POS, creating a cycle. Your function receives only the head: print YES if the list has a cycle, else NO. Use O(1) memory.

**Input:** Line 1: N POS · Line 2: N values  
**Output:** YES or NO

**Constraints:** 1 ≤ N ≤ 10^4 · −1 ≤ POS < N

**Example 1**

| Input | Output |
|---|---|
| <pre>4 1<br>3 2 0 -4</pre> | <pre>YES</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>2 0<br>1 2</pre> | <pre>YES</pre> |

<details><summary>Hints</summary>

1. Floyd's tortoise and hare: slow moves 1, fast moves 2.
2. If fast reaches null there is no cycle; if they meet there is one.

</details>

**Approach:** Fast/slow pointers — in a cycle the fast pointer gains one node per step and must catch up.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static class ListNode {
        int val;
        ListNode next;
        ListNode(int val) { this.val = val; }
    }

    static boolean hasCycle(ListNode head) {
        ListNode slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) return true;
        }
        return false;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), pos = sc.nextInt();
        ListNode[] nodes = new ListNode[n];
        for (int i = 0; i < n; i++) {
            nodes[i] = new ListNode(sc.nextInt());
            if (i > 0) nodes[i - 1].next = nodes[i];
        }
        if (pos >= 0) nodes[n - 1].next = nodes[pos];
        System.out.println(hasCycle(nodes[0]) ? "YES" : "NO");
    }
}
```

</details>

### 169. Majority Element

**Easy** · Arrays · [LeetCode](https://leetcode.com/problems/majority-element) · asked at Accenture, TCS, Cognizant, Infosys

The array contains a value that appears MORE than N/2 times. Print it using O(1) extra space.

**Input:** Line 1: N · Line 2: N integers  
**Output:** The majority element

**Constraints:** 1 ≤ N ≤ 5·10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>3 2 3</pre> | <pre>3</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>7<br>2 2 1 1 1 2 2</pre> | <pre>2</pre> |

<details><summary>Hints</summary>

1. Boyer–Moore voting: pair each majority vote with a different vote — the majority survives.

</details>

**Approach:** Keep a candidate and a count; when count hits 0 adopt the current value as the new candidate.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int majority(int[] a) {
        int cand = 0, count = 0;
        for (int x : a) {
            if (count == 0) cand = x;
            count += (x == cand) ? 1 : -1;
        }
        return cand;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(majority(a));
    }
}
```

</details>

### 206. Reverse Linked List

**Easy** · Linked List · [LeetCode](https://leetcode.com/problems/reverse-linked-list) · asked at Accenture, TCS

Reverse a singly linked list and return the new head. The program builds the list from the input and prints the reversed values.

**Input:** Line 1: N · Line 2: N values  
**Output:** Values of the reversed list

**Constraints:** 1 ≤ N ≤ 5000

**Example 1**

| Input | Output |
|---|---|
| <pre>5<br>1 2 3 4 5</pre> | <pre>5 4 3 2 1</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>2<br>1 2</pre> | <pre>2 1</pre> |

<details><summary>Hints</summary>

1. Keep three references: prev, cur, next.
2. Save cur.next before redirecting cur.next to prev.

</details>

**Approach:** Iterative pointer reversal. (Recursive version: reverse the rest, then head.next.next = head.)

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static class ListNode {
        int val;
        ListNode next;
        ListNode(int val) { this.val = val; }
    }

    static ListNode reverseList(ListNode head) {
        ListNode prev = null, cur = head;
        while (cur != null) {
            ListNode next = cur.next;
            cur.next = prev;
            prev = cur;
            cur = next;
        }
        return prev;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        ListNode dummy = new ListNode(0), tail = dummy;
        for (int i = 0; i < n; i++) tail = tail.next = new ListNode(sc.nextInt());
        StringJoiner sj = new StringJoiner(" ");
        for (ListNode p = reverseList(dummy.next); p != null; p = p.next) sj.add(String.valueOf(p.val));
        System.out.println(sj);
    }
}
```

</details>

### 217. Contains Duplicate

**Easy** · Hashing · [LeetCode](https://leetcode.com/problems/contains-duplicate) · asked at Accenture, Capgemini, TCS, Infosys

Print YES if any value appears at least twice in the array, otherwise NO.

**Input:** Line 1: N · Line 2: N integers  
**Output:** YES or NO

**Constraints:** 1 ≤ N ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>4<br>1 2 3 1</pre> | <pre>YES</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>4<br>1 2 3 4</pre> | <pre>NO</pre> |

<details><summary>Hints</summary>

1. HashSet.add returns false when the value is already present.

</details>

**Approach:** One pass with a HashSet (O(N) time, O(N) space); sorting gives O(1) extra space at O(N log N).

**Complexity:** time O(N) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static boolean containsDuplicate(int[] a) {
        Set<Integer> seen = new HashSet<>();
        for (int x : a) if (!seen.add(x)) return true;
        return false;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(containsDuplicate(a) ? "YES" : "NO");
    }
}
```

</details>

### 387. First Unique Character in a String

**Easy** · Hashing · [LeetCode](https://leetcode.com/problems/first-unique-character-in-a-string) · asked at Accenture, TCS

Print the index of the first character that appears exactly once in the lowercase string S, or -1 if none.

**Input:** A single line S.  
**Output:** Index or -1

**Constraints:** 1 ≤ |S| ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>leetcode</pre> | <pre>0</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>loveleetcode</pre> | <pre>2</pre> |

<details><summary>Hints</summary>

1. Count frequencies in a first pass (int[26]), find the first count-1 character in a second pass.

</details>

**Approach:** Two passes over the string with a 26-slot counter.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int firstUniqChar(String s) {
        int[] cnt = new int[26];
        for (char c : s.toCharArray()) cnt[c - 'a']++;
        for (int i = 0; i < s.length(); i++) if (cnt[s.charAt(i) - 'a'] == 1) return i;
        return -1;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(firstUniqChar(sc.next()));
    }
}
```

</details>

### 485. Max Consecutive Ones

**Easy** · Arrays · [LeetCode](https://leetcode.com/problems/max-consecutive-ones) · asked at Accenture, Cognizant, TCS

Given a binary array, print the maximum number of consecutive 1s.

**Input:** Line 1: N · Line 2: N values (0/1)  
**Output:** Longest run of 1s

**Constraints:** 1 ≤ N ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>1 1 0 1 1 1</pre> | <pre>3</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>6<br>1 0 1 1 0 1</pre> | <pre>2</pre> |

<details><summary>Hints</summary>

1. Keep a running count that resets to 0 on every 0.

</details>

**Approach:** Single pass with current and best counters.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int maxOnes(int[] a) {
        int cur = 0, best = 0;
        for (int x : a) {
            cur = (x == 1) ? cur + 1 : 0;
            best = Math.max(best, cur);
        }
        return best;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(maxOnes(a));
    }
}
```

</details>

### 977. Squares of a Sorted Array

**Easy** · Two Pointers · [LeetCode](https://leetcode.com/problems/squares-of-a-sorted-array) · asked at Accenture, TCS, Infosys

Given an array sorted in non-decreasing order (may include negatives), print the squares of each number, sorted, in O(N).

**Input:** Line 1: N · Line 2: N sorted integers  
**Output:** Sorted squares

**Constraints:** 1 ≤ N ≤ 10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>5<br>-4 -1 0 3 10</pre> | <pre>0 1 9 16 100</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>5<br>-7 -3 2 3 11</pre> | <pre>4 9 9 49 121</pre> |

<details><summary>Hints</summary>

1. The largest square is at one of the two ends.
2. Fill the output from the back with two pointers.

</details>

**Approach:** Two pointers at both ends; place the larger square at the current back position.

**Complexity:** time O(N) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int[] sortedSquares(int[] a) {
        int n = a.length;
        int[] out = new int[n];
        int l = 0, r = n - 1;
        for (int k = n - 1; k >= 0; k--) {
            if (Math.abs(a[l]) > Math.abs(a[r])) {
                out[k] = a[l] * a[l];
                l++;
            } else {
                out[k] = a[r] * a[r];
                r--;
            }
        }
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        StringJoiner sj = new StringJoiner(" ");
        for (int v : sortedSquares(a)) sj.add(String.valueOf(v));
        System.out.println(sj);
    }
}
```

</details>

### 2. Add Two Numbers (Linked Lists)

**Medium** · Linked List · [LeetCode](https://leetcode.com/problems/add-two-numbers) · asked at Accenture, Capgemini, Accolite, Cognizant, TCS, Infosys

Two non-negative numbers are stored as linked lists with digits in REVERSE order (the head is the ones digit). Return their sum as a linked list in the same format. The program prints the resulting digits from head to tail.

**Input:** Line 1: N · Line 2: N digits of list 1 · Line 3: M · Line 4: M digits of list 2  
**Output:** Digits of the result list, space-separated

**Constraints:** 1 ≤ N, M ≤ 100 · No leading zeros except the number 0 itself

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>2 4 3<br>3<br>5 6 4</pre> | <pre>7 0 8</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1<br>0<br>1<br>0</pre> | <pre>0</pre> |

<details><summary>Hints</summary>

1. Walk both lists together like grade-school addition, keeping a carry.
2. A dummy head node avoids special-casing the first digit.
3. Don’t forget a final carry.

</details>

**Approach:** Simultaneous traversal with a carry; continue while either list has nodes or carry > 0.

**Complexity:** time O(max(N, M)) · space O(max(N, M)) for the result

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static class ListNode {
        int val;
        ListNode next;
        ListNode(int val) { this.val = val; }
    }

    static ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0), tail = dummy;
        int carry = 0;
        while (l1 != null || l2 != null || carry > 0) {
            int sum = carry;
            if (l1 != null) { sum += l1.val; l1 = l1.next; }
            if (l2 != null) { sum += l2.val; l2 = l2.next; }
            tail.next = new ListNode(sum % 10);
            tail = tail.next;
            carry = sum / 10;
        }
        return dummy.next;
    }

    static ListNode read(Scanner sc) {
        int n = sc.nextInt();
        ListNode dummy = new ListNode(0), tail = dummy;
        for (int i = 0; i < n; i++) tail = tail.next = new ListNode(sc.nextInt());
        return dummy.next;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        ListNode a = read(sc), b = read(sc);
        StringJoiner sj = new StringJoiner(" ");
        for (ListNode p = addTwoNumbers(a, b); p != null; p = p.next) sj.add(String.valueOf(p.val));
        System.out.println(sj);
    }
}
```

</details>

### 5. Longest Palindromic Substring

**Medium** · Strings · [LeetCode](https://leetcode.com/problems/longest-palindromic-substring) · asked at Mphasis, Accenture, Cognizant, TCS, Infosys, Accolite, HCL, Persistent Systems

Given a string S, print its longest palindromic substring. If several have the maximum length, print the one that starts first.

**Input:** A single line S (letters and digits, no spaces).  
**Output:** The longest palindromic substring.

**Constraints:** 1 ≤ |S| ≤ 1000

**Example 1**

| Input | Output |
|---|---|
| <pre>babad</pre> | <pre>bab</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>cbbd</pre> | <pre>bb</pre> |

<details><summary>Hints</summary>

1. Every palindrome mirrors around a centre — there are 2N−1 centres (characters and gaps).
2. Expand outward from each centre while the ends match.

</details>

**Approach:** Expand around centre: for each index try an odd-length centre (i, i) and an even one (i, i+1). Update only on a strictly longer palindrome so the leftmost wins ties.

**Complexity:** time O(N²) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String longestPalindrome(String s) {
        int start = 0, best = 1;
        for (int c = 0; c < s.length(); c++) {
            for (int k = 0; k < 2; k++) {
                int l = c, r = c + k;
                while (l >= 0 && r < s.length() && s.charAt(l) == s.charAt(r)) {
                    l--;
                    r++;
                }
                int len = r - l - 1;
                if (len > best) {
                    best = len;
                    start = l + 1;
                }
            }
        }
        return s.substring(start, start + best);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(longestPalindrome(sc.next()));
    }
}
```

</details>

### 7. Reverse Integer

**Medium** · Math · [LeetCode](https://leetcode.com/problems/reverse-integer) · asked at Tech Mahindra, Accenture, LTI, Wipro, Cognizant, Capgemini, TCS, Infosys

Given a 32-bit signed integer X, print X with its digits reversed. If the reversed value falls outside the 32-bit signed range [−2^31, 2^31 − 1], print 0. Do not use `long` to store the result.

**Input:** A single integer X.  
**Output:** The reversed integer or 0.

**Constraints:** −2^31 ≤ X ≤ 2^31 − 1

**Example 1**

| Input | Output |
|---|---|
| <pre>123</pre> | <pre>321</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>-123</pre> | <pre>-321</pre> |

<details><summary>Hints</summary>

1. Pop digits with `x % 10` and `x / 10` (works for negatives in Java).
2. Before `rev * 10 + d`, check whether it would overflow by comparing rev with `Integer.MAX_VALUE / 10`.

</details>

**Approach:** Pop and push digits, checking for overflow before each multiply-add. Java’s % keeps the sign of the dividend, so negatives work naturally.

**Complexity:** time O(log |X|) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int reverse(int x) {
        int rev = 0;
        while (x != 0) {
            int d = x % 10;
            x /= 10;
            if (rev > Integer.MAX_VALUE / 10 || (rev == Integer.MAX_VALUE / 10 && d > 7)) return 0;
            if (rev < Integer.MIN_VALUE / 10 || (rev == Integer.MIN_VALUE / 10 && d < -8)) return 0;
            rev = rev * 10 + d;
        }
        return rev;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(reverse(sc.nextInt()));
    }
}
```

</details>

### 8. String to Integer (atoi)

**Medium** · Strings · [LeetCode](https://leetcode.com/problems/string-to-integer-atoi) · asked at Accenture, TCS, Infosys

Implement atoi: skip leading spaces; read an optional + or − sign; read digits until the first non-digit; ignore the rest. If no digits were read the result is 0. Clamp the result to the 32-bit signed range.

**Input:** A single line (may have leading spaces).  
**Output:** The parsed integer

**Constraints:** 1 ≤ |S| ≤ 200

**Example 1**

| Input | Output |
|---|---|
| <pre>42</pre> | <pre>42</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>   -042</pre> | <pre>-42</pre> |

<details><summary>Hints</summary>

1. Handle the steps strictly in order: spaces → sign → digits.
2. Detect overflow before multiplying (compare with Integer.MAX_VALUE / 10), or accumulate in a long and clamp.

</details>

**Approach:** Deterministic state walk over the string; clamp as soon as the value exceeds the range.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int myAtoi(String s) {
        int i = 0, n = s.length(), sign = 1;
        while (i < n && s.charAt(i) == ' ') i++;
        if (i < n && (s.charAt(i) == '+' || s.charAt(i) == '-')) sign = s.charAt(i++) == '-' ? -1 : 1;
        long val = 0;
        while (i < n && Character.isDigit(s.charAt(i))) {
            val = val * 10 + (s.charAt(i++) - '0');
            if (sign * val > Integer.MAX_VALUE) return Integer.MAX_VALUE;
            if (sign * val < Integer.MIN_VALUE) return Integer.MIN_VALUE;
        }
        return (int) (sign * val);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(myAtoi(sc.nextLine()));
    }
}
```

</details>

### 12. Integer to Roman

**Medium** · Strings · [LeetCode](https://leetcode.com/problems/integer-to-roman) · asked at Infosys, Accenture, TCS

Convert an integer in [1, 3999] to a Roman numeral (use subtractive forms IV, IX, XL, XC, CD, CM).

**Input:** A single integer N.  
**Output:** The Roman numeral

**Constraints:** 1 ≤ N ≤ 3999

**Example 1**

| Input | Output |
|---|---|
| <pre>3</pre> | <pre>III</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>58</pre> | <pre>LVIII</pre> |

<details><summary>Hints</summary>

1. List the 13 symbol values from largest to smallest, including the subtractive pairs (900 = CM, 4 = IV…).
2. Greedily subtract the largest value that fits.

</details>

**Approach:** Greedy over a fixed table of 13 value/symbol pairs.

**Complexity:** time O(1) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String intToRoman(int n) {
        int[] vals = {1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1};
        String[] syms = {"M", "CM", "D", "CD", "C", "XC", "L", "XL", "X", "IX", "V", "IV", "I"};
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < vals.length; i++) {
            while (n >= vals[i]) {
                n -= vals[i];
                sb.append(syms[i]);
            }
        }
        return sb.toString();
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(intToRoman(sc.nextInt()));
    }
}
```

</details>

### 15. 3Sum

**Medium** · Two Pointers · [LeetCode](https://leetcode.com/problems/3sum) · asked at Accenture, TCS, Infosys, HCL

Find all UNIQUE triplets that sum to zero. Print the number of triplets on the first line, then each triplet (ascending within the triplet) on its own line, triplets in lexicographic order.

**Input:** Line 1: N · Line 2: N integers  
**Output:** Count, then the triplets

**Constraints:** 3 ≤ N ≤ 3000 · −10^5 ≤ a[i] ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>-1 0 1 2 -1 -4</pre> | <pre>2<br>-1 -1 2<br>-1 0 1</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3<br>0 1 1</pre> | <pre>0</pre> |

<details><summary>Hints</summary>

1. Sort first.
2. Fix a[i], then two-pointer search the rest for −a[i].
3. Skip equal neighbours for i, left and right to avoid duplicates.

</details>

**Approach:** Sort + for each i a two-pointer scan. Sorting makes duplicate-skipping easy and yields lexicographic order automatically.

**Complexity:** time O(N²) · space O(1) extra (besides output)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static List<int[]> threeSum(int[] a) {
        Arrays.sort(a);
        List<int[]> out = new ArrayList<>();
        for (int i = 0; i < a.length - 2; i++) {
            if (i > 0 && a[i] == a[i - 1]) continue;
            int l = i + 1, r = a.length - 1;
            while (l < r) {
                int s = a[i] + a[l] + a[r];
                if (s == 0) {
                    out.add(new int[]{a[i], a[l], a[r]});
                    while (l < r && a[l] == a[l + 1]) l++;
                    while (l < r && a[r] == a[r - 1]) r--;
                    l++;
                    r--;
                } else if (s < 0) l++;
                else r--;
            }
        }
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        List<int[]> res = threeSum(a);
        StringBuilder sb = new StringBuilder().append(res.size()).append('\n');
        for (int[] t : res) sb.append(t[0]).append(' ').append(t[1]).append(' ').append(t[2]).append('\n');
        System.out.print(sb);
    }
}
```

</details>

### 17. Letter Combinations of a Phone Number

**Medium** · Backtracking · [LeetCode](https://leetcode.com/problems/letter-combinations-of-a-phone-number) · asked at Accenture, TCS

Each digit 2–9 maps to letters as on a phone keypad (2 = abc, 3 = def, 4 = ghi, 5 = jkl, 6 = mno, 7 = pqrs, 8 = tuv, 9 = wxyz). Print every letter combination the digit string could represent, one per line, in lexicographic order.

**Input:** A single line of digits (2–9).  
**Output:** All combinations

**Constraints:** 1 ≤ length ≤ 4

**Example 1**

| Input | Output |
|---|---|
| <pre>23</pre> | <pre>ad<br>ae<br>af<br>bd<br>be<br>bf<br>cd<br>ce<br>cf</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>2</pre> | <pre>a<br>b<br>c</pre> |

<details><summary>Hints</summary>

1. Backtrack over positions; at each digit try its letters in order.
2. Trying letters in keypad order already yields lexicographic output.

</details>

**Approach:** DFS building the combination one digit at a time (Cartesian product).

**Complexity:** time O(4^N · N) · space O(N) recursion

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static final String[] KEYS = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};

    static void combine(String digits, int i, StringBuilder cur, List<String> out) {
        if (i == digits.length()) {
            out.add(cur.toString());
            return;
        }
        for (char c : KEYS[digits.charAt(i) - '0'].toCharArray()) {
            cur.append(c);
            combine(digits, i + 1, cur, out);
            cur.deleteCharAt(cur.length() - 1);
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        List<String> out = new ArrayList<>();
        combine(sc.next(), 0, new StringBuilder(), out);
        System.out.println(String.join("\n", out));
    }
}
```

</details>

### 31. Next Permutation

**Medium** · Arrays · [LeetCode](https://leetcode.com/problems/next-permutation) · asked at Infosys, Cognizant, Accenture, TCS

Rearrange the array into the next lexicographically greater permutation IN PLACE. If it is the largest permutation, rearrange it into the smallest (sorted ascending). Print the result.

**Input:** Line 1: N · Line 2: N integers  
**Output:** The next permutation

**Constraints:** 1 ≤ N ≤ 100

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 2 3</pre> | <pre>1 3 2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3<br>3 2 1</pre> | <pre>1 2 3</pre> |

<details><summary>Hints</summary>

1. From the right, find the first index i with a[i] < a[i+1] (the pivot).
2. Swap a[i] with the smallest element to its right that is larger than it.
3. Reverse the suffix after i.

</details>

**Approach:** The suffix after the pivot is non-increasing; swapping in the next-larger value and reversing the suffix gives the smallest larger arrangement.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static void swap(int[] a, int i, int j) {
        int t = a[i];
        a[i] = a[j];
        a[j] = t;
    }

    static void nextPermutation(int[] a) {
        int i = a.length - 2;
        while (i >= 0 && a[i] >= a[i + 1]) i--;
        if (i >= 0) {
            int j = a.length - 1;
            while (a[j] <= a[i]) j--;
            swap(a, i, j);
        }
        for (int l = i + 1, r = a.length - 1; l < r; l++, r--) swap(a, l, r);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        nextPermutation(a);
        StringJoiner sj = new StringJoiner(" ");
        for (int x : a) sj.add(String.valueOf(x));
        System.out.println(sj);
    }
}
```

</details>

### 46. Permutations

**Medium** · Backtracking · [LeetCode](https://leetcode.com/problems/permutations) · asked at Infosys, TCS

Given N distinct integers in ascending order, print all permutations in lexicographic order, one per line (values space-separated).

**Input:** Line 1: N · Line 2: N distinct ascending integers  
**Output:** N! lines

**Constraints:** 1 ≤ N ≤ 6

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 2 3</pre> | <pre>1 2 3<br>1 3 2<br>2 1 3<br>2 3 1<br>3 1 2<br>3 2 1</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1<br>1</pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. Backtrack with a boolean used[] array.
2. Iterating candidates in ascending order produces lexicographic order.

</details>

**Approach:** DFS choosing each unused element for the next position, undoing the choice afterwards.

**Complexity:** time O(N · N!) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static void permute(int[] a, boolean[] used, List<Integer> cur, List<List<Integer>> out) {
        if (cur.size() == a.length) {
            out.add(new ArrayList<>(cur));
            return;
        }
        for (int i = 0; i < a.length; i++) {
            if (used[i]) continue;
            used[i] = true;
            cur.add(a[i]);
            permute(a, used, cur, out);
            cur.remove(cur.size() - 1);
            used[i] = false;
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        List<List<Integer>> out = new ArrayList<>();
        permute(a, new boolean[n], new ArrayList<>(), out);
        StringBuilder sb = new StringBuilder();
        for (List<Integer> p : out) {
            StringJoiner sj = new StringJoiner(" ");
            for (int v : p) sj.add(String.valueOf(v));
            sb.append(sj).append('\n');
        }
        System.out.print(sb);
    }
}
```

</details>

### 48. Rotate Image

**Medium** · Matrix · [LeetCode](https://leetcode.com/problems/rotate-image) · asked at Accenture, Infosys, TCS

Rotate an N×N matrix 90° clockwise IN PLACE, then print it.

**Input:** Line 1: N · Next N lines: N integers  
**Output:** The rotated matrix

**Constraints:** 1 ≤ N ≤ 20

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>1 2 3<br>4 5 6<br>7 8 9</pre> | <pre>7 4 1<br>8 5 2<br>9 6 3</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1<br>5</pre> | <pre>5</pre> |

<details><summary>Hints</summary>

1. Clockwise rotation = transpose, then reverse each row.

</details>

**Approach:** Transpose (swap a[i][j] with a[j][i] for j > i), then reverse every row. Both steps are in place.

**Complexity:** time O(N²) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static void rotate(int[][] a) {
        int n = a.length;
        for (int i = 0; i < n; i++)
            for (int j = i + 1; j < n; j++) {
                int t = a[i][j];
                a[i][j] = a[j][i];
                a[j][i] = t;
            }
        for (int[] row : a)
            for (int l = 0, r = n - 1; l < r; l++, r--) {
                int t = row[l];
                row[l] = row[r];
                row[r] = t;
            }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[][] a = new int[n][n];
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++) a[i][j] = sc.nextInt();
        rotate(a);
        StringBuilder sb = new StringBuilder();
        for (int[] row : a) {
            StringJoiner sj = new StringJoiner(" ");
            for (int v : row) sj.add(String.valueOf(v));
            sb.append(sj).append('\n');
        }
        System.out.print(sb);
    }
}
```

</details>

### 49. Group Anagrams

**Medium** · Hashing · [LeetCode](https://leetcode.com/problems/group-anagrams) · asked at Infosys, Wipro, Persistent Systems, Capgemini, Accolite, Cognizant, TCS

Group the given lowercase words into anagram groups. Print one group per line: words inside a group sorted ascending and separated by spaces; groups ordered by their first (smallest) word.

**Input:** Line 1: N · Line 2: N words  
**Output:** One group per line

**Constraints:** 1 ≤ N ≤ 10^4 · 1 ≤ |word| ≤ 100

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>eat tea tan ate nat bat</pre> | <pre>ate eat tea<br>bat<br>nat tan</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>1<br>a</pre> | <pre>a</pre> |

<details><summary>Hints</summary>

1. Two words are anagrams iff their sorted letters are equal — use that as a HashMap key.
2. A 26-count signature is an O(L) alternative to sorting.

</details>

**Approach:** HashMap from canonical key (sorted characters) → list of words. Then sort within and across groups for deterministic output.

**Complexity:** time O(N · L log L) · space O(N · L)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static List<List<String>> groupAnagrams(String[] words) {
        Map<String, List<String>> groups = new HashMap<>();
        for (String w : words) {
            char[] c = w.toCharArray();
            Arrays.sort(c);
            groups.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(w);
        }
        List<List<String>> out = new ArrayList<>(groups.values());
        for (List<String> g : out) Collections.sort(g);
        out.sort((a, b) -> a.get(0).compareTo(b.get(0)));
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        String[] w = new String[n];
        for (int i = 0; i < n; i++) w[i] = sc.next();
        StringBuilder sb = new StringBuilder();
        for (List<String> g : groupAnagrams(w)) sb.append(String.join(" ", g)).append('\n');
        System.out.print(sb);
    }
}
```

</details>

### 54. Spiral Matrix

**Medium** · Matrix · [LeetCode](https://leetcode.com/problems/spiral-matrix) · asked at TCS, Infosys, Accenture

Print all elements of an R×C matrix in clockwise spiral order, starting at the top-left.

**Input:** Line 1: R C · Next R lines: C integers  
**Output:** R·C space-separated values

**Constraints:** 1 ≤ R, C ≤ 10

**Example 1**

| Input | Output |
|---|---|
| <pre>3 3<br>1 2 3<br>4 5 6<br>7 8 9</pre> | <pre>1 2 3 6 9 8 7 4 5</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3 4<br>1 2 3 4<br>5 6 7 8<br>9 10 11 12</pre> | <pre>1 2 3 4 8 12 11 10 9 5 6 7</pre> |

<details><summary>Hints</summary>

1. Keep four boundaries: top, bottom, left, right.
2. After walking the top row and right column, re-check top ≤ bottom and left ≤ right before walking back.

</details>

**Approach:** Layer-by-layer traversal shrinking the four boundaries; the re-checks handle single rows/columns.

**Complexity:** time O(R·C) · space O(1) extra

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static List<Integer> spiral(int[][] m) {
        List<Integer> out = new ArrayList<>();
        int top = 0, bottom = m.length - 1, left = 0, right = m[0].length - 1;
        while (top <= bottom && left <= right) {
            for (int j = left; j <= right; j++) out.add(m[top][j]);
            top++;
            for (int i = top; i <= bottom; i++) out.add(m[i][right]);
            right--;
            if (top <= bottom) {
                for (int j = right; j >= left; j--) out.add(m[bottom][j]);
                bottom--;
            }
            if (left <= right) {
                for (int i = bottom; i >= top; i--) out.add(m[i][left]);
                left++;
            }
        }
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int r = sc.nextInt(), c = sc.nextInt();
        int[][] m = new int[r][c];
        for (int i = 0; i < r; i++)
            for (int j = 0; j < c; j++) m[i][j] = sc.nextInt();
        StringJoiner sj = new StringJoiner(" ");
        for (int v : spiral(m)) sj.add(String.valueOf(v));
        System.out.println(sj);
    }
}
```

</details>

### 75. Sort Colors (Dutch National Flag)

**Medium** · Two Pointers · [LeetCode](https://leetcode.com/problems/sort-colors) · asked at TCS, Capgemini, Infosys

The array contains only 0s, 1s and 2s. Sort it in place in ONE pass without a library sort or counting, then print it.

**Input:** Line 1: N · Line 2: N values (0/1/2)  
**Output:** The sorted array

**Constraints:** 1 ≤ N ≤ 300

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>2 0 2 1 1 0</pre> | <pre>0 0 1 1 2 2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3<br>2 0 1</pre> | <pre>0 1 2</pre> |

<details><summary>Hints</summary>

1. Three regions: [0, low) = 0s, [low, mid) = 1s, (high, N) = 2s.
2. When swapping a 2 to the end, do NOT advance mid — the swapped-in value is unexamined.

</details>

**Approach:** Dijkstra's three-way partition with low / mid / high pointers.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static void sortColors(int[] a) {
        int low = 0, mid = 0, high = a.length - 1;
        while (mid <= high) {
            if (a[mid] == 0) {
                int t = a[low]; a[low++] = a[mid]; a[mid++] = t;
            } else if (a[mid] == 1) {
                mid++;
            } else {
                int t = a[high]; a[high--] = a[mid]; a[mid] = t;
            }
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        sortColors(a);
        StringJoiner sj = new StringJoiner(" ");
        for (int x : a) sj.add(String.valueOf(x));
        System.out.println(sj);
    }
}
```

</details>

### 122. Best Time to Buy and Sell Stock II

**Medium** · Greedy · [LeetCode](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) · asked at Accenture, TCS, Infosys

You may complete as many transactions as you like (buy then sell, holding at most one share at a time). Print the maximum total profit.

**Input:** Line 1: N · Line 2: N prices  
**Output:** Maximum profit

**Constraints:** 1 ≤ N ≤ 3·10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>7 1 5 3 6 4</pre> | <pre>7</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>5<br>1 2 3 4 5</pre> | <pre>4</pre> |

<details><summary>Hints</summary>

1. Any rising segment can be split into day-to-day gains.
2. Sum every positive difference prices[i] − prices[i−1].

</details>

**Approach:** Greedy: collecting every upward step equals the best set of transactions.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int maxProfit(int[] p) {
        int profit = 0;
        for (int i = 1; i < p.length; i++) profit += Math.max(0, p[i] - p[i - 1]);
        return profit;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] p = new int[n];
        for (int i = 0; i < n; i++) p[i] = sc.nextInt();
        System.out.println(maxProfit(p));
    }
}
```

</details>

### 128. Longest Consecutive Sequence

**Medium** · Hashing · [LeetCode](https://leetcode.com/problems/longest-consecutive-sequence) · asked at Capgemini, Infosys, TCS

Print the length of the longest run of consecutive integers that can be formed from the array values (order in the array does not matter). Must be O(N).

**Input:** Line 1: N · Line 2: N integers  
**Output:** Length of the longest consecutive sequence

**Constraints:** 1 ≤ N ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>6<br>100 4 200 1 3 2</pre> | <pre>4</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>10<br>0 3 7 2 5 8 4 6 0 1</pre> | <pre>9</pre> |

<details><summary>Hints</summary>

1. Put everything in a HashSet.
2. Only start counting from x when x − 1 is NOT in the set — that makes x the start of a run.

</details>

**Approach:** Each run is walked exactly once from its start, so total work is O(N) despite the nested loop.

**Complexity:** time O(N) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int longestConsecutive(int[] a) {
        Set<Integer> set = new HashSet<>();
        for (int x : a) set.add(x);
        int best = 0;
        for (int x : set) {
            if (set.contains(x - 1)) continue;
            int len = 1;
            while (set.contains(x + len)) len++;
            best = Math.max(best, len);
        }
        return best;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(longestConsecutive(a));
    }
}
```

</details>

### 151. Reverse Words in a String

**Medium** · Strings · [LeetCode](https://leetcode.com/problems/reverse-words-in-a-string) · asked at Accenture, HCL, Infosys, TCS

Reverse the order of words in a line. The input may contain leading, trailing or multiple spaces; the output must have single spaces and no leading/trailing spaces.

**Input:** A single line.  
**Output:** Words in reverse order

**Constraints:** 1 ≤ |S| ≤ 10^4 · At least one word

**Example 1**

| Input | Output |
|---|---|
| <pre>the sky is blue</pre> | <pre>blue is sky the</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>  hello world  </pre> | <pre>world hello</pre> |

<details><summary>Hints</summary>

1. Scan from the end: skip spaces, find the start of the word, append it.
2. Avoid split(" ") — it creates empty strings with multiple spaces (split("\\s+") after trim works too).

</details>

**Approach:** Right-to-left scan collecting words into a StringBuilder — O(N) and handles any spacing.

**Complexity:** time O(N) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String reverseWords(String s) {
        StringBuilder sb = new StringBuilder();
        int i = s.length() - 1;
        while (i >= 0) {
            while (i >= 0 && s.charAt(i) == ' ') i--;
            if (i < 0) break;
            int end = i;
            while (i >= 0 && s.charAt(i) != ' ') i--;
            if (sb.length() > 0) sb.append(' ');
            sb.append(s, i + 1, end + 1);
        }
        return sb.toString();
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(reverseWords(sc.nextLine()));
    }
}
```

</details>

### 155. Min Stack

**Medium** · Stack · [LeetCode](https://leetcode.com/problems/min-stack) · asked at TCS, Infosys

Design a stack supporting push, pop, top and getMin — ALL in O(1). Process Q operations; print the result of every `top` and `getMin` on its own line. Operations are always valid (never on an empty stack).

**Input:** Line 1: Q · Next Q lines: "push x", "pop", "top" or "getMin"  
**Output:** One line per top / getMin

**Constraints:** 1 ≤ Q ≤ 3·10^4

**Example 1**

| Input | Output |
|---|---|
| <pre>7<br>push -2<br>push 0<br>push -3<br>getMin<br>pop<br>top<br>getMin</pre> | <pre>-3<br>0<br>-2</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>3<br>push 5<br>top<br>getMin</pre> | <pre>5<br>5</pre> |

<details><summary>Hints</summary>

1. Store, with each pushed value, the minimum of the stack at that moment.
2. Popping then automatically restores the previous minimum.

</details>

**Approach:** Each stack entry is a pair (value, minSoFar). Alternative: a second stack holding only the minima.

**Complexity:** time O(1) per operation · space O(Q)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static class MinStack {
        private final Deque<int[]> st = new ArrayDeque<>(); // {value, min so far}

        void push(int x) {
            int min = st.isEmpty() ? x : Math.min(x, st.peek()[1]);
            st.push(new int[]{x, min});
        }

        void pop() {
            st.pop();
        }

        int top() {
            return st.peek()[0];
        }

        int getMin() {
            return st.peek()[1];
        }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int q = sc.nextInt();
        MinStack st = new MinStack();
        StringBuilder sb = new StringBuilder();
        while (q-- > 0) {
            String op = sc.next();
            switch (op) {
                case "push" -> st.push(sc.nextInt());
                case "pop" -> st.pop();
                case "top" -> sb.append(st.top()).append('\n');
                default -> sb.append(st.getMin()).append('\n');
            }
        }
        System.out.print(sb);
    }
}
```

</details>

### 179. Largest Number

**Medium** · Sorting · [LeetCode](https://leetcode.com/problems/largest-number) · asked at Accenture, TCS, Infosys

Arrange N non-negative integers so that their concatenation forms the largest possible number. Print it as a string (print "0" if the result is all zeros).

**Input:** Line 1: N · Line 2: N integers  
**Output:** The largest number

**Constraints:** 1 ≤ N ≤ 100 · 0 ≤ a[i] ≤ 10^9

**Example 1**

| Input | Output |
|---|---|
| <pre>2<br>10 2</pre> | <pre>210</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>5<br>3 30 34 5 9</pre> | <pre>9534330</pre> |

<details><summary>Hints</summary>

1. Sort the numbers as strings with a custom comparator: x before y if x+y > y+x.
2. If the first element after sorting is "0", the answer is "0".

</details>

**Approach:** Custom comparator on concatenations defines a valid total order; greedy by that order is optimal.

**Complexity:** time O(N log N · L) · space O(N · L)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static String largestNumber(int[] a) {
        String[] s = new String[a.length];
        for (int i = 0; i < a.length; i++) s[i] = String.valueOf(a[i]);
        Arrays.sort(s, (x, y) -> (y + x).compareTo(x + y));
        if (s[0].equals("0")) return "0";
        return String.join("", s);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(largestNumber(a));
    }
}
```

</details>

### 189. Rotate Array

**Medium** · Arrays · [LeetCode](https://leetcode.com/problems/rotate-array) · asked at Accenture, Capgemini, Wipro, Cognizant, TCS, Infosys

Rotate an array to the RIGHT by K steps in place (K may exceed N), then print it.

**Input:** Line 1: N K · Line 2: N integers  
**Output:** The rotated array

**Constraints:** 1 ≤ N ≤ 10^5 · 0 ≤ K ≤ 10^9

**Example 1**

| Input | Output |
|---|---|
| <pre>7 3<br>1 2 3 4 5 6 7</pre> | <pre>5 6 7 1 2 3 4</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>4 2<br>-1 -100 3 99</pre> | <pre>3 99 -1 -100</pre> |

<details><summary>Hints</summary>

1. K %= N first.
2. Reverse the whole array, then reverse the first K and the remaining N−K elements.

</details>

**Approach:** Triple reversal gives O(1) extra space: reverse(all), reverse(0..K−1), reverse(K..N−1).

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static void reverse(int[] a, int i, int j) {
        while (i < j) {
            int t = a[i];
            a[i++] = a[j];
            a[j--] = t;
        }
    }

    static void rotate(int[] a, int k) {
        int n = a.length;
        k %= n;
        reverse(a, 0, n - 1);
        reverse(a, 0, k - 1);
        reverse(a, k, n - 1);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), k = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        rotate(a, k);
        StringJoiner sj = new StringJoiner(" ");
        for (int x : a) sj.add(String.valueOf(x));
        System.out.println(sj);
    }
}
```

</details>

### 204. Count Primes

**Medium** · Math · [LeetCode](https://leetcode.com/problems/count-primes) · asked at Accenture, TCS

Print the number of prime numbers strictly less than N.

**Input:** A single integer N.  
**Output:** Count of primes < N

**Constraints:** 0 ≤ N ≤ 5·10^6

**Example 1**

| Input | Output |
|---|---|
| <pre>10</pre> | <pre>4</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>0</pre> | <pre>0</pre> |

<details><summary>Hints</summary>

1. Sieve of Eratosthenes.
2. Start crossing out multiples of p from p·p (use long to avoid overflow).

</details>

**Approach:** Sieve: every composite below N is crossed out by its smallest prime factor p ≤ √N.

**Complexity:** time O(N log log N) · space O(N)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int countPrimes(int n) {
        if (n < 3) return 0;
        boolean[] composite = new boolean[n];
        int count = 0;
        for (int p = 2; p < n; p++) {
            if (composite[p]) continue;
            count++;
            for (long m = (long) p * p; m < n; m += p) composite[(int) m] = true;
        }
        return count;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println(countPrimes(sc.nextInt()));
    }
}
```

</details>

### 238. Product of Array Except Self

**Medium** · Arrays · [LeetCode](https://leetcode.com/problems/product-of-array-except-self) · asked at Infosys, Accenture, TCS

For every index i print the product of all elements except a[i]. Do it in O(N) WITHOUT using division.

**Input:** Line 1: N · Line 2: N integers  
**Output:** N space-separated products

**Constraints:** 2 ≤ N ≤ 10^5 · Every product fits in a 32-bit integer

**Example 1**

| Input | Output |
|---|---|
| <pre>4<br>1 2 3 4</pre> | <pre>24 12 8 6</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>5<br>-1 1 0 -3 3</pre> | <pre>0 0 9 0 0</pre> |

<details><summary>Hints</summary>

1. answer[i] = (product of everything left of i) × (product of everything right of i).
2. Fill left products in one pass, then multiply by a running right product in a backward pass.

</details>

**Approach:** Prefix and suffix products; reusing the output array for prefixes gives O(1) extra space.

**Complexity:** time O(N) · space O(1) extra

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int[] productExceptSelf(int[] a) {
        int n = a.length;
        int[] out = new int[n];
        out[0] = 1;
        for (int i = 1; i < n; i++) out[i] = out[i - 1] * a[i - 1];
        int right = 1;
        for (int i = n - 1; i >= 0; i--) {
            out[i] *= right;
            right *= a[i];
        }
        return out;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        StringJoiner sj = new StringJoiner(" ");
        for (int v : productExceptSelf(a)) sj.add(String.valueOf(v));
        System.out.println(sj);
    }
}
```

</details>

### 852. Peak Index in a Mountain Array

**Medium** · Binary Search · [LeetCode](https://leetcode.com/problems/peak-index-in-a-mountain-array) · asked at Accenture, TCS

A mountain array strictly increases to a single peak and then strictly decreases. Print the index of the peak in O(log N).

**Input:** Line 1: N · Line 2: N integers  
**Output:** Peak index

**Constraints:** 3 ≤ N ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>3<br>0 1 0</pre> | <pre>1</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>4<br>0 2 1 0</pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. Compare a[mid] with a[mid + 1]: rising means the peak is to the right.

</details>

**Approach:** Binary search on the monotonic predicate "a[i] > a[i+1]" (false on the ascent, true after the peak).

**Complexity:** time O(log N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int peakIndex(int[] a) {
        int lo = 0, hi = a.length - 1;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] < a[mid + 1]) lo = mid + 1;
            else hi = mid;
        }
        return lo;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        System.out.println(peakIndex(a));
    }
}
```

</details>

### 1823. Find the Winner of the Circular Game

**Medium** · Math · [LeetCode](https://leetcode.com/problems/find-the-winner-of-the-circular-game) · asked at Accenture, TCS

N friends (numbered 1..N) sit in a circle. Starting from friend 1, count K friends clockwise (including the start); the K-th friend leaves, and counting restarts from the next friend. Print the last friend remaining.

**Input:** A single line: N K  
**Output:** The winner’s number

**Constraints:** 1 ≤ K ≤ N ≤ 500

**Example 1**

| Input | Output |
|---|---|
| <pre>5 2</pre> | <pre>3</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>6 5</pre> | <pre>1</pre> |

<details><summary>Hints</summary>

1. Simulating with a queue works in O(N·K).
2. Josephus recurrence (0-indexed): f(1) = 0, f(n) = (f(n−1) + K) % n. Answer f(N) + 1.

</details>

**Approach:** Josephus recurrence: after one removal the circle of n becomes a relabelled circle of n − 1 shifted by K.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static int findWinner(int n, int k) {
        int f = 0;
        for (int i = 2; i <= n; i++) f = (f + k) % i;
        return f + 1;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), k = sc.nextInt();
        System.out.println(findWinner(n, k));
    }
}
```

</details>

### 4. Median of Two Sorted Arrays

**Hard** · Binary Search · [LeetCode](https://leetcode.com/problems/median-of-two-sorted-arrays) · asked at Accenture, Capgemini, Cognizant, TCS

Given two sorted arrays of sizes M and N, print the median of all M + N values with exactly one decimal place. Aim for O(log(min(M, N))).

**Input:** Line 1: M · Line 2: M sorted integers (may be empty) · Line 3: N · Line 4: N sorted integers (may be empty)  
**Output:** Median, e.g. 2.5

**Constraints:** 0 ≤ M, N ≤ 1000 · 1 ≤ M + N

**Example 1**

| Input | Output |
|---|---|
| <pre>2<br>1 3<br>1<br>2</pre> | <pre>2.0</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>2<br>1 2<br>2<br>3 4</pre> | <pre>2.5</pre> |

<details><summary>Hints</summary>

1. Binary-search a cut in the SMALLER array; the cut in the other array is then fixed so the left halves hold (M+N+1)/2 elements.
2. A cut is valid when maxLeftA ≤ minRightB and maxLeftB ≤ minRightA.

</details>

**Approach:** Partition binary search: move the cut in A left if A’s left max is too big, right otherwise. Sentinels ±∞ handle empty sides.

**Complexity:** time O(log min(M, N)) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static double findMedian(int[] a, int[] b) {
        if (a.length > b.length) return findMedian(b, a);
        int m = a.length, n = b.length, lo = 0, hi = m, half = (m + n + 1) / 2;
        while (lo <= hi) {
            int i = (lo + hi) / 2, j = half - i;
            int aL = i == 0 ? Integer.MIN_VALUE : a[i - 1], aR = i == m ? Integer.MAX_VALUE : a[i];
            int bL = j == 0 ? Integer.MIN_VALUE : b[j - 1], bR = j == n ? Integer.MAX_VALUE : b[j];
            if (aL <= bR && bL <= aR) {
                if ((m + n) % 2 == 1) return Math.max(aL, bL);
                return (Math.max(aL, bL) + (double) Math.min(aR, bR)) / 2.0;
            }
            if (aL > bR) hi = i - 1;
            else lo = i + 1;
        }
        return 0;
    }

    static int[] read(Scanner sc) {
        int n = sc.nextInt();
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = sc.nextInt();
        return a;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int[] a = read(sc), b = read(sc);
        System.out.println(String.format(Locale.ROOT, "%.1f", findMedian(a, b)));
    }
}
```

</details>

### 42. Trapping Rain Water

**Hard** · Two Pointers · [LeetCode](https://leetcode.com/problems/trapping-rain-water) · asked at Capgemini, TCS, Infosys, Accenture

Given N bar heights (width 1 each), print how much rain water is trapped between the bars.

**Input:** Line 1: N · Line 2: N heights  
**Output:** Units of trapped water

**Constraints:** 1 ≤ N ≤ 2·10^4 · 0 ≤ h ≤ 10^5

**Example 1**

| Input | Output |
|---|---|
| <pre>12<br>0 1 0 2 1 0 1 3 2 1 2 1</pre> | <pre>6</pre> |

**Example 2**

| Input | Output |
|---|---|
| <pre>6<br>4 2 0 3 2 5</pre> | <pre>9</pre> |

<details><summary>Hints</summary>

1. Water above bar i = min(maxLeft, maxRight) − h[i].
2. Two pointers: the side with the smaller max is the limiting side — process it and move inward.

</details>

**Approach:** Two pointers with running leftMax/rightMax. Whichever max is smaller bounds the water on that side, so it can be settled immediately.

**Complexity:** time O(N) · space O(1)

<details><summary>Java solution</summary>

```java
import java.util.*;

public class Main {
    static long trap(int[] h) {
        int l = 0, r = h.length - 1, leftMax = 0, rightMax = 0;
        long water = 0;
        while (l < r) {
            if (h[l] < h[r]) {
                leftMax = Math.max(leftMax, h[l]);
                water += leftMax - h[l++];
            } else {
                rightMax = Math.max(rightMax, h[r]);
                water += rightMax - h[r--];
            }
        }
        return water;
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] h = new int[n];
        for (int i = 0; i < n; i++) h[i] = sc.nextInt();
        System.out.println(trap(h));
    }
}
```

</details>
