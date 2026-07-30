# Why these four modules, and what they actually have in common

[notes index](README.md)

Read individually, modules 2 through 4 look like four unrelated exercises: three classifiers, a set of red-team labs, and a prompt injection module. They aren't unrelated. Two vulnerabilities show up again and again, in every module, regardless of whether the target is a scikit-learn model or an LLM:

## 1. Class imbalance is a security vulnerability, not a statistics footnote

Every dataset in Module 2 was imbalanced, and every time, the underrepresented class became the exploitable blind spot:

| Model | Underrepresented class | Result |
|---|---|---|
| Network anomaly detector | privilege escalation — 108 samples vs 77,207 normal | 24% recall — an attacker who knows this targets `buffer_overflow`/`rootkit` specifically |
| Malware image classifier | rarest family (Skintrim.N) — 80 samples vs 2,949 for the largest | model concept still generalizes, but the family-level accuracy gap follows the sample-count gap |

The pattern: **accuracy hides the gap, recall-per-class reveals it.** A model can post 99%+ overall accuracy while being nearly blind to the one class an attacker actually cares about. See [module2-notes.md](module2-notes.md) for the full breakdown and the confusion matrix images.

## 2. LLMs and Naive Bayes fail the same way: no trust boundary between instruction and data

This sounds like a stretch until you look at the actual mechanics:

- **Naive Bayes (Module 2/3)** sums word-probability contributions across the *entire* input with no concept of "this part is the real message, this part is an attacker's addition." Padding a spam message with enough ham-signal words dilutes the spam score below threshold — [Module 3, Lab 1](module3-notes.md#lab-1-input-manipulation-ml01).
- **LLMs (Module 4)** concatenate system prompt and user input into one token stream with no structural trust boundary either. The "operator note" technique — framing an injection as a *post-processing verification step* rather than an override — worked almost everywhere for exactly this reason: the model has no reliable way to tell "legitimate system instruction" from "attacker-supplied text that looks like one." See [module4-notes.md](module4-notes.md#the-operator-note--system-framing--highest-success-rate-technique-of-the-entire-module).

Different math, same root cause: **the model has no architectural concept of which part of its input it should trust less.**

## Other patterns worth carrying forward

- **Denylists can prime the exact behavior they try to prevent.** Listing forbidden actions by name ("do not spell-check, repeat...") made a defended LLM *more* likely to associate those words with something to do, especially when the attack used the same vocabulary. Redefining the ambiguous reference (what does "the above" mean?) instead of prohibiting actions proved more robust — [Defense 3](module4-notes.md#defense-3--where-denylist-style-defenses-backfired-critical-lesson).
- **Fusing techniques into one message beats splitting them across turns.** Every time a "permission update" (Skeleton Key) was issued as a separate message before the harmful request, the model's safety training re-engaged and caught the second message — even after agreeing to the first. Fusing them into a single prompt prevented that reset.
- **Simple web vulnerabilities often beat sophisticated ML-specific attacks.** Model theft in Module 3 didn't need adversarial querying to reconstruct a decision boundary — it needed an unauthenticated `/model` endpoint. The ML sophistication of an attack and its real-world likelihood are not correlated.
- **Output filters can be bypassed by changing format, not content.** When direct key disclosure was blocked, ROT13 encoding and piece-by-piece character extraction both exfiltrated the same data, because the filter matched literal strings rather than semantic meaning.

Full technical detail, every payload, every dead end: [module2-notes.md](module2-notes.md) · [module3-notes.md](module3-notes.md) · [module4-notes.md](module4-notes.md)
