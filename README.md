**[Download CLAUDE.md](./CLAUDE.md)**

# Bringing Opus's Behavior Closer to Fable's

**The goal of this project is to make Claude Opus, running as the main session in [Claude Code](https://claude.com/claude-code), behave like the higher-capability Claude Fable** — as an **architect / reviewer / subagent-manager** that designs, delegates implementation to subagents, and audits the result, rather than grinding through code itself and asking permission at every step. (Fable is a Claude model that sits a tier above Opus in capability.)

Fable exhibits that lead-engineer discipline on its own. Opus has the raw capability but does not reliably *fire* the same behaviors unsolicited. This [`CLAUDE.md`](./CLAUDE.md) is the scaffolding that closes that gap.

It was built as one clear arc, and this README follows it:

**Goal** → **Method** (measure the Fable–Opus gap) → **Findings** (what the gap actually is) → **Iteration** (develop a prompt that closes it) → **Scores** (validate across four measurement systems).

> 日本語版: [README.ja.md](./README.ja.md)

---

## 1. Goal

Make Opus, as a main session, behave like Fable: **design, delegate, audit — not implement-and-ask.** Close the specific behavioral gaps identified below, while preserving everything Opus already does well on a single task.

## 2. Method — how the gap was measured

To decide *what* to change rather than guess, the Fable–Opus gap was triangulated from three independent sources, which converged on the same conclusions:

1. **A natural experiment.** The same developer ran a same-family project once with Fable as the main session and once with Opus. The Opus run produced a catalog of dozens of user corrections — repeated permission-asking, approximate implementations, verification handed back to the user, regressions on rewrites. The Fable run showed none of these. This is the direct evidence of *what to fix*.
2. **Published tier differences.** Comparing leaked/published system prompts across model tiers of the same product showed that the higher tier (Fable's) is given explicit autonomy and turn-ending discipline that the lower tier (Opus's) is not — the vendor itself scaffolds the exact behaviors the natural experiment flagged as missing. Those passages were adapted here.
3. **Mining the Fable sessions** for the concrete behaviors worth porting: the shape of a good delegation brief, the audit chain run on a subagent's output, the habit of judging a subagent's deviation against *principles* rather than instruction-matching.

## 3. Findings — what the gap actually is

Two findings drove the entire design.

**Finding 1 — the gap is discipline-firing, not capability.** On a single fresh task, Opus matches Fable closely: it audits code, verifies completion, designs architecture, and comprehends a large codebase at a high level. What differs is *which disciplines fire without being asked.* Compared with Fable, Opus as a main session tends to:

- **stop and ask permission** at every batch boundary ("Shall I continue?") instead of running a well-scoped task to completion;
- **declare work done** on a passing type-check, then ask *you* to verify the runtime behavior;
- **stop debugging at the first cause it finds**, rationalizing the remaining symptom away instead of checking every layer;
- **silently make user-visible breaking changes** (renaming a documented API) while asking permission for trivial internal choices — the escalation axis is inverted;
- **drift** over a long session: a rule stated early erodes as the context fills.

**Finding 2 — therefore the prompt must be procedural, not aspirational.** Because the capability is present but the reflex is not, principle lists ("be thorough") measurably underperform **trigger-conditioned procedures** ("when you receive X, before doing Y, do Z"). And the disciplines most prone to erosion are stated three ways — as a principle, as a named anti-pattern, and as a self-check — because redundancy on the highest-frequency failure modes is what moved the benchmark.

## 4. Iteration — developing the prompt

The prompt was developed as a measured loop, not a single draft:

1. **Baseline.** Score stock Opus (and Opus with a generic `CLAUDE.md`) on a battery of discriminating tasks — audit, autonomous completion, intent-reading, completion-verification, delegation-spec, multi-layer debugging.
2. **Hypotheses.** Form one hypothesis per observed gap, each tied back to its evidence from §2.
3. **Write & re-score.** Encode the hypotheses as trigger-conditioned procedures; re-score. Keep only changes that moved a discriminating task **without regressing the rest.**
4. **Guard against overfitting.** Re-check on held-out tasks the prompt was never tuned on (a from-scratch architecture design; a behavior-preserving refactor).
5. **Converge.** Stop when the battery saturates and the held-out tasks show no regression.

The disciplines targeted the three tasks where stock Opus actually diverged from Fable (multi-layer debugging, escalation calibration, intent-reading); the audit / completion-verification / design tasks were already at ceiling for stock Opus and were held there.

## 5. Scores — validation across four measurement systems

Tested against the stock general-purpose `CLAUDE.md` (and a no-prompt baseline), with objective graders wherever possible — hidden test suites, independent re-execution of the resulting repository state, byte-level output comparison, citation spot-checks.

| Measurement system | What it tests | Result |
|---|---|---|
| **Custom benchmark battery** | Audit, autonomous completion, intent-reading, completion-verification, delegation-spec, multi-layer debugging | **90 / 90** with this prompt vs **77 / 90** for the stock prompt; gains concentrated on the three discriminating tasks, no regression on the rest |
| **Real OSS tasks** (a validation library, an HTTP framework, a build tool) | Fixing real, merged issues; comprehending a large codebase | Graded by the actual merged-PR regression tests as a hidden grader. **Zero regression** vs the stock prompt; large-codebase architecture maps verified citation-by-citation with zero fabrication |
| **Flat benchmark in real Claude Code sessions** | The same tasks run in genuine main sessions, not simulated | Opus + this prompt reached **parity with Fable** on the discriminating tasks, and clearly exceeded stock Opus |
| **Long-session marathon** (8 sequential waves, ~1M tokens processed) | Whether the disciplines survive fatigue and a growing context | The multi-layer-debugging and escalation-calibration advantages **persisted under fatigue**: at a late wave, the stock prompt rationalized away a second bug while this prompt found and fixed both layers |

**Honest caveats** (in the spirit of the document's own reporting discipline): several cells are single-run; the grader was also the experiment designer (mitigated by making every score an objective, independently re-runnable check); and the most extreme real-world failures (dozens of permission-prompts over a multi-day session) require a multi-day, multi-compaction scale the fixed-length tests did not reach. The long-session countermeasures are evidence-based but their behavior at that extreme scale is left to real-world use.

---

## What the prompt actually adds

The core interventions, each tied to an observed Fable–Opus difference. (The § numbers point to sections in [`CLAUDE.md`](./CLAUDE.md) itself.)

- **Autonomous completion (§0.1, §5).** Reversible work that follows from the request proceeds without asking; a report never ends with a request for permission to continue.
- **Intent before action (§0.2).** Typos, slips, and instructions that seem aimed at a different context are read for intent, not executed literally; when a name and its behavior disagree, the *semantic* problem is fixed across every layer.
- **A three-lane escalation classifier (§2).** Every decision is *ask-first* (breaks a consumer-visible contract), *proceed-with-stated-assumption* (an execution detail derivable from principles), or *verify-before-claiming* (a completion claim) — designed to fix the inverted escalation axis directly.
- **A debugging descent protocol (§4.3).** Quantify the expected-correct state before touching anything; suspect every layer, not just code; finding one cause does not end the search.
- **A word-gate on "done" (§4.5).** The words *done / passing / 完了 / 完成* may be written only after you have run the verification axes yourself; asking the user to observe the result counts as skipping a step.
- **A delegation brief template and audit chain (§4.9).** Eight mandatory sections for any subagent brief, and a fixed audit sequence run on the result — because a subagent's "all green" self-report is an *input* to your audit, never its conclusion.
- **Long-session decay countermeasures (§4.10).** Wave-boundary re-anchoring, standing rules written to files rather than held in memory, and a precision-decay self-check for long mechanical stretches.

The rest of the document carries a general engineering philosophy — eight meta-principles, vendor-isolation, hard-cutover discipline, fail-closed defense, blueprint-vs-history documentation rules — that the interventions above sit on top of.

## Using it

Drop [`CLAUDE.md`](./CLAUDE.md) into the root of a repository you drive with Claude Code, with Opus as the main-session model. Add your project's structure, commands, and domain knowledge under the **Project-Specific** section at the end.

A few notes:

- **It assumes a subagent-capable harness.** The model strategy (§1) delegates ordinary implementation to subagents and keeps the main session on design, audit, and review. If your setup has no subagents, the design/audit disciplines still apply; the delegation sections become guidance for how *you* would structure and verify that work.
- **Model names are abstracted where possible.** Where a tier is named inside the document, it is easy to swap; the intent is a reasoning-tier abstraction (light / middle / heavy), not a specific product name.
- **Examples are trilingual.** Illustrative quoted phrases appear in English / 日本語 / 中文 interchangeably; the disciplines they illustrate are language-agnostic. Replace or extend them with your project's working language.
- **It is a blueprint, not a changelog.** Per its own §6, keep it present-tense; when a rule changes, rewrite the section as the new current state rather than appending history.

## Design notes

- The document is deliberately written as procedures with firing conditions, because the central finding is that Opus *can* do these things but does not reliably *fire* them unsolicited. Principle lists were measurably weaker than "when you receive X, do Y."
- The most-corrected behavior in the source material — ending a batch by asking permission — is addressed in three places (§0.1, §2, §5) on purpose. Redundancy on the highest-frequency failure mode was one of the techniques that moved the benchmark.
- The escalation classifier is three-way rather than a simple "when in doubt, ask," because the observed failure was *not* under-asking — it was asking about the wrong things. Both directions of the inversion had to be named.

---

## Author

[Yukiya Okuda](https://x.com/alumican_net)
