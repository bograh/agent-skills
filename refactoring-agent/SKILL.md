---
name: refactoring-agent
description: >
  Expert code refactoring agent based on refactoring.guru methodology. Use this skill whenever
  a user wants to improve, clean up, or restructure existing code — even if they don't use the
  word "refactor". Triggers include: "this code is messy", "can you clean this up", "make this
  better", "reduce duplication", "this function is too long", "improve readability", "fix code
  smells", "simplify this", "restructure this class/module", "this is hard to maintain", or
  any time the user shares code and asks for improvements without wanting to change behavior.
  Also trigger when reviewing pull requests for quality, doing code reviews, or analyzing
  technical debt. Always use this skill before suggesting rewrites — prefer targeted refactoring
  over full replacement.
---

# Refactoring Agent

A structured agent for analyzing and refactoring code using the refactoring.guru methodology.
Covers **21 code smells** and **66 refactoring techniques** organized into 6 groups.

## Agent Workflow

Follow this sequence for every refactoring task:

### Step 1 — Smell Detection
Scan the code for code smells before proposing any changes. See `references/code-smells.md`
for the full taxonomy. Identify and name each smell explicitly.

### Step 2 — Prioritize
Rank smells by impact: correctness > maintainability > readability > style.
Focus on the highest-impact issues first. Don't refactor everything at once.

### Step 3 — Select Techniques
For each smell, identify the applicable refactoring technique(s) from `references/techniques.md`.
Each technique has a well-defined recipe — follow it precisely, don't improvise.

### Step 4 — Apply Incrementally
Apply one refactoring at a time. Each step must:
- Not change observable behavior (no feature additions, no bug fixes mixed in)
- Leave the code in a runnable/compilable state
- Be independently reviewable

### Step 5 — Explain
For each change, state:
- **What** changed (technique name)
- **Why** (which smell it fixes)
- **How** to verify behavior is preserved (tests to run, assertions to check)

---

## Quick-Reference: Smell → Technique Mapping

### Bloaters (code that grows too big)
| Smell | Primary Technique |
|---|---|
| Long Method | Extract Method, Replace Temp with Query |
| Large Class | Extract Class, Extract Subclass |
| Primitive Obsession | Replace Data Value with Object, Introduce Parameter Object |
| Long Parameter List | Introduce Parameter Object, Preserve Whole Object |
| Data Clumps | Extract Class, Introduce Parameter Object |

### Object-Orientation Abusers
| Smell | Primary Technique |
|---|---|
| Switch Statements | Replace Conditional with Polymorphism, Replace Type Code with Subclasses |
| Temporary Field | Extract Class, Introduce Null Object |
| Refused Bequest | Replace Inheritance with Delegation |
| Alternative Classes w/ Different Interfaces | Rename Method, Move Method |

### Change Preventers (make changes hard to make)
| Smell | Primary Technique |
|---|---|
| Divergent Change | Extract Class |
| Shotgun Surgery | Move Method, Move Field, Inline Class |
| Parallel Inheritance Hierarchies | Move Method, Move Field |

### Dispensables (unnecessary code)
| Smell | Primary Technique |
|---|---|
| Comments | Extract Method, Rename Method |
| Duplicate Code | Extract Method, Pull Up Method, Form Template Method |
| Lazy Class | Inline Class, Collapse Hierarchy |
| Data Class | Move Method, Encapsulate Field |
| Dead Code | Delete it |
| Speculative Generality | Inline Class, Collapse Hierarchy, Remove Parameter |

### Couplers (excessive coupling between classes)
| Smell | Primary Technique |
|---|---|
| Feature Envy | Move Method, Extract Method |
| Inappropriate Intimacy | Move Method, Move Field, Extract Class |
| Message Chains | Hide Delegate, Extract Method |
| Middle Man | Remove Middle Man, Inline Method |
| Incomplete Library Class | Introduce Foreign Method, Introduce Local Extension |

---

## Output Format

When presenting refactored code, use this structure:

```
## Smells Detected
1. [Smell Name] — [brief description of where/why]
2. ...

## Refactoring Plan
1. [Technique] to fix [Smell] → affects [lines/function/class]
2. ...

## Refactored Code
[code block with changes]

## What Changed
- **[Technique name]**: [1-sentence explanation]
- ...

## Verification
- Run [specific tests/assertions] to confirm behavior is unchanged
```

---

## Key Principles (from refactoring.guru)

1. **Refactoring ≠ rewriting** — change structure, not behavior
2. **Tests first** — if there are no tests, writing them before refactoring is strongly recommended
3. **Small steps** — each refactoring should be atomic and reversible
4. **Don't mix concerns** — a refactoring PR should contain zero feature changes
5. **Code smells are symptoms** — treat the root cause, not just the symptom

---

## Reference Files

- `references/code-smells.md` — Full descriptions of all 21 code smells with examples
- `references/techniques.md` — All 66 refactoring techniques with step-by-step recipes

Load these when you need detailed instructions for a specific smell or technique.
