# Coding Interview — Conceptual Depth (Staff Lens)

Reported practice: [PracHub rotation + CSV](https://prachub.com/interview-questions/implement-array-rotations-and-csv-keyword-search).

---

## 1. How Staff coding rounds differ (expectations)

You may still write code **or** sketch on whiteboard/CoderPad. Evaluators look for:

- **Requirement clarity** before typing.  
- **Correctness** on edge cases.  
- **Communication** while thinking.  
- **Clean** decomposition (functions, naming).  
- **Complexity** stated correctly.  
- **Calm** refactor when bug found.

**Not** the only signal at Staff—but **sloppy** code contradicts “raises the bar.”

---

## 2. Array rotation — algorithm depth

**Left rotate by k:** move first **k** elements to end.

**Algorithm 1 — extra array (O(n) time, O(n) space):**  
Copy `nums[k..n-1]` then `nums[0..k-1]`. Easiest to **get right** under pressure.

**Algorithm 2 — triple reverse (O(n) time, O(1) extra):**  
Reverse `0..k-1`, reverse `k..n-1`, reverse whole array.  
**Why it works:** Relationship between **left rotate** and **reversals** is a standard invariant proof—memorize **pattern**, proof optional in interview.

**Edge cases:**

- `n == 0` — return early.  
- `k %= n` — **`k` can be huge**.  
- `k == 0` — no-op.

**Right rotate by k:** equivalent to **left** rotate by `n - (k % n)` (with modulo care).

---

## 3. CSV keyword search — parsing depth

**Assumptions:** newline rows, comma columns, **no** embedded commas or quotes (as in prompt). State that **explicitly**.

**Algorithm:**

1. `split('\n')` → rows (watch **trailing** newline producing **empty** last row—**trim** policy?).  
2. For each row, `split(',')` → cells.  
3. For each cell, scan with `indexOf` from **start**, advancing **start+1** for **overlapping** matches (or `indexOf` loop). Clarify if **non-overlapping** matches only.

**Complexity:** R rows, C cols, average cell length L, keyword length K.

- Naive substring per position: **O(L × K)** per cell worst case; **O(R × C × L × K)** total. Acceptable for interview if sizes modest.  
- **KMP** or **Rabin-Karp** reduce repeated work for **many** positions—mention if interviewer pushes.

### Multi-keyword extension

| Approach | Idea | When |
|----------|------|------|
| Loop keywords | For each keyword, scan | Few keywords, small grid |
| **Aho-Corasick** | Build trie + failure links, single pass | Many keywords, long document |

**Aho-Corasick intuition:** Generalized trie where **failure links** act like “if mismatch, jump to longest proper suffix that is also prefix”—**one pass** over text.

**Tradeoff:** Build time + memory for automaton vs **simple** code.

---

## 4. What to say out loud (script)

1. “I’ll clarify **empty string**, **overlapping** matches, **Unicode**—assume ASCII unless told.”  
2. “Brute force first for clarity, then optimize if needed.”  
3. “Testing: example from prompt + no match + multiple rows + k > n.”  
4. “Time **O(…)**, space **O(…)** because …”

---

## 5. Staff-level review commentary

After solving, **volunteer**: how you’d **unit test**, how this maps to **production** (streaming CSV ≠ tiny string—**buffered** parsing), **memory** if file huge.

---

## 6. Worked examples (step-by-step on paper)

### Example A — Left rotate `nums = [1,2,3,4,5]`, `k = 2`

**Effective** `k = 2 % 5 = 2`.

**Extra-array:** new = `[3,4,5]` + `[1,2]` → `[3,4,5,1,2]`.

**Triple-reverse:**

1. Reverse `0..1`: `[2,1,3,4,5]`  
2. Reverse `2..4`: `[2,1,5,4,3]`  
3. Reverse whole: **`[3,4,5,1,2]`** ✓

**Right rotate by `k=2`:** equivalent to left rotate by `5-2=3` → `[4,5,1,2,3]`.

### Example B — `k > n`

`nums = [10,20]`, `k = 5` → `k % 2 = 1` → left rotate **1** → `[20,10]`.

### Example C — CSV keyword `foo`

**Input string:**

```text
foo,bar
xxfood,barfoo
```

Search **`foo`** (overlapping **allowed**):

| Location | Match |
|----------|--------|
| Row 0, Col 0, pos 0 | `foo` |
| Row 1, Col 0, pos 2 | `foo` inside `xxfood` |
| Row 1, Col 1, pos 3 | `foo` at end of `barfoo` |

**Row-major order:** list as above top-to-bottom, left-to-right within row.

### Example D — Overlapping in one cell

Cell = `aaaa`, key = `aa`.

- Positions **0** and **1** and **2** if **overlapping** allowed.  
- If interviewer says **non-overlapping**, after match at **0**, **next** start at **2** → matches at **0, 2** only—**clarify**.

### Example E — Test cases you name aloud

- `nums=[]`, any `k`  
- `nums=[7]`, `k=100`  
- CSV: empty file; row with **empty** last cell after trailing comma `a,b,`  
- Unicode: “assume UTF-8 code points unless asked”

---

## Related

- System design on same tool: [03](./03-system-design-ad-serving-experiments.md)
