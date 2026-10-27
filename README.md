# dsa-notes

Personal repo for LeetCode practice: Java solutions plus short notes on patterns, complexity, and mistakes.

## Layout

```
notes/                 # Pattern cheatsheet, Java collections, study log
problems/              # One folder per problem, grouped by topic
  _template/           # Copy this when you start a new problem
  arrays/
  two-pointers/
  ...
```

Cheat sheets:

- [`notes/java-collections.md`](notes/java-collections.md) — List / Set / Queue / Map Big-O, HashMap internals, conversions
- [`notes/java-dsa-cheatsheet.md`](notes/java-dsa-cheatsheet.md) — strings, regex, graphs, DP, binary search templates
- [`notes/patterns.md`](notes/patterns.md) — short pattern reminders while solving

Each problem folder should look like:

```
problems/arrays/0001-two-sum/
  Solution.java
  notes.md
```

Use the LeetCode id (4 digits) plus a kebab-case title. That keeps problems easy to find and sort.

## How to add a problem

1. Copy `problems/_template/` into the matching topic folder.
2. Rename it, for example `0001-two-sum`.
3. Paste your LeetCode Java solution into `Solution.java`.
4. Fill in `notes.md` while the problem is still fresh: approach, complexity, and what you missed.

Run a solution locally with:

```bash
javac problems/arrays/0001-two-sum/Solution.java
```

LeetCode’s editor is still the main judge. Local `javac` is only for a quick compile check.

## Topics

| Folder | Typical patterns |
| --- | --- |
| `arrays` | prefix sums, hashing, sorting |
| `two-pointers` | pair sums, in-place partition |
| `sliding-window` | subarray / substring constraints |
| `binary-search` | search space, first/last true |
| `linked-list` | reverse, fast/slow, dummy head |
| `stack` | monotonic stack, parentheses |
| `heap` | kth, merge, streaming |
| `trees` | DFS/BFS, BST, LCA |
| `graphs` | BFS/DFS, topo, union-find |
| `dp` | knapsack, LIS, grid, intervals |
| `backtracking` | subsets, permutations, combinations |
| `greedy` | intervals, jump, local choice |
| `misc` | math, bit, design, simulation |

If a problem fits two topics, put it in the one you used to solve it, and mention the other in `notes.md`.

## Git

This folder is ready to become a git repository. On this Mac, `git` currently needs Apple’s Command Line Tools:

```bash
xcode-select --install
cd ~/dsa-notes
git init -b main
git add .
git commit -m "Initial commit: Java LeetCode notes layout"
```
