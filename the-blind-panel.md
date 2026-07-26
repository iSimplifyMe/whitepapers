> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-blind-panel](https://isimplifyme.com/whitepapers/the-blind-panel)** — it revises there first; this mirror follows.
> Joe Elstner, Founder, iSimplifyMe · Published 2026-07-26 · License: [CC BY 4.0](LICENSE)

# The Blind Panel: Position Bias and Agreement as Preconditions for a Win Rate

A measurement architecture for evaluation panels — the two statistics that decide whether a win rate is a result or a number, and why a panel that reports only the win rate has published nothing.

---

## Abstract

> **Answer:** A win rate from an evaluation panel is uninterpretable on its own. Two cheap statistics determine whether it carries information: position bias, which reveals judges responding to layout rather than content, and inter-rater agreement, which reveals whether the panel detected any shared signal at all. Either failure voids the win rate entirely, and neither is expensive to compute. A panel that reports a win rate without them has produced a number, not a result.

This is the evaluation-loop companion to [*The Trust Ladder: Supervised Autonomy for AI Code Review*](https://isimplifyme.com/whitepapers/the-trust-ladder). The Trust Ladder covers how a non-deterministic gate earns authority on measured precision. This paper covers the measurement itself — specifically the moment an organization asks a panel of judges, human or model, which of two candidates is better, and treats the answer as evidence.

The intended reader is anyone running an LLM-as-judge evaluation, an A/B panel over model outputs, a design review scored by a rubric, or a human preference study. The argument is short: **the win rate is the number everyone quotes and the number that means least**, and the two statistics that would tell you whether to believe it are almost never reported.

The reference implementation is [`blind-panel`](https://github.com/iSimplifyMe/blind-panel) — zero dependencies, MIT.

---

## 1. The Win Rate Is Not the Result

> **Answer:** A win rate answers "how often did judges prefer candidate A" and nothing else. It cannot distinguish a real quality difference from judges systematically picking the left-hand option, from judges answering at random, or from four samples of noise. Those three failure modes produce win rates that look exactly like a finding, and each is detectable with arithmetic a spreadsheet could do.

The standard evaluation report is a table of win rates. Candidate A beat the reference 72% of the time; candidate B, 48%. The conclusion writes itself, and it is frequently wrong — not because the judging was careless, but because the report omits everything that would let a reader check it.

Consider three panels, each reporting a 72% win rate for candidate A.

In the first, judges were shown A on the left in every pair. A's identity and its screen position are perfectly confounded, and a well-documented left-side preference in pairwise choice tasks is sufficient to produce 72% with no quality difference whatsoever.

In the second, the judges disagreed with each other at close to chance. Individually each produced a verdict; collectively they detected nothing. Averaging noise produces a number with a decimal point, and the decimal point is doing rhetorical work the data cannot support.

In the third, there were eleven comparisons. Eight of eleven is 72%. The 95% confidence interval on eight of eleven spans roughly 43% to 90% — which is to say, the result is consistent with candidate A being worse.

All three published the same headline. Only one of them might mean something, and the report as written cannot tell you which.

---

## 2. The Six-Zero Run

> **Answer:** The first smoke test of our own blinding tool produced a run in which the candidate was placed on the left in every single pair — the exact confound the tool exists to detect, occurring inside the tool. The cause was independent random side assignment, which with six items produces an all-one-side run about 3% of the time. Randomness is not balance, and treating it as balance is a bug that hides inside correct-looking code.

We publish this because the failure is instructive and because the code that produced it was, in the ordinary sense, correct.

The blinding step assigned each pair a side with an independent seeded coin flip. That is the obvious implementation, it is genuinely random, it is reproducible from the seed, and it passed its unit tests. The first end-to-end run printed:

```
"sideBalance": { "candidateLeft": 6, "candidateRight": 0 }
```

Six items, six left placements. With independent flips the probability of an all-one-side run at n=6 is 1 in 32 — roughly 3%, which is to say it will happen, and it will happen quietly. Nothing errored. The manifest was valid, the key was valid, the tally arithmetic was correct. A panel run against that manifest would have produced a perfectly reasonable-looking win rate in which candidate identity and screen position were indistinguishable.

Three properties of that failure generalize past our implementation:

- **Correct randomness is not the same as a correct design.** Every individual assignment was unbiased. The *set* was degenerate. Statistical properties of a procedure are not inherited by any particular run of it.
- **The failure was silent and shaped like success.** No exception, no warning, no anomalous value — just a valid artifact that could not support the inference someone would draw from it. This is the same shape as the Trust Ladder's expired-credential reviewer, where absence of signal looked identical to approval.
- **It was visible only because the tool reported something nobody asks for.** Side balance is not a standard output of an evaluation harness. It was printed because it seemed cheap to print. That is the entire reason the bug was caught before it reached a result.

The fix is in §3, and the general rule is the one that follows from it: **prefer guarantees by construction over properties that hold in expectation.**

---

## 3. Balance by Construction

> **Answer:** Rather than flipping a coin per pair, assign each candidate an equal number of left and right placements and shuffle that list with a seeded Fisher-Yates. Balance is then a property of every run rather than an average over many runs, while the assignment of any individual pair remains unpredictable to the judge. This is the same move as bounding a system's behaviour structurally instead of monitoring for violations.

The distinction matters because the two approaches are indistinguishable in testing and diverge exactly when it counts.

Independent flips give balance *in expectation*. Over a thousand runs the mean skew approaches zero, and every statistical property you would write down about the procedure is satisfied. But an evaluation is not a thousand runs; it is one run, and a degenerate one is both possible and undetectable from inside the result.

Constructed balance gives it *per run*. For each candidate, build a list of placements that is half left and half right — odd counts take the extra placement from the seeded stream, so the surplus does not always favour the same side across candidates — then shuffle. Every candidate now receives an identical number of left and right placements. The judge still cannot predict any individual pair, because the shuffle is unpredictable; what has been removed is only the possibility of a pathological set.

The architectural principle generalizes well beyond blinding, and it is the same one behind [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain): when a property is load-bearing, make it structurally impossible to violate rather than probable to satisfy. Monitoring catches violations after they occur. Construction prevents the class.

---

## 4. The Two Preconditions

> **Answer:** Position bias is the share of "left" answers across the panel; far from 50% means judges responded to layout rather than content. Inter-rater agreement is the mean pairwise rate at which judges reached the same unblinded verdict; near chance means the panel detected no shared signal. Both should gate the report — a run failing either has not measured anything and should exit non-zero rather than publish a win rate.

### Position bias

Compute the fraction of all verdicts that chose the left-hand option. Under a correctly blinded design with balanced placement, this should sit near 0.5 regardless of which candidate is better, because "better" and "left" have been decorrelated by construction.

A large deviation has exactly one innocent explanation — sampling noise at small n — and several damaging ones: judges skimming and defaulting, a rendering artifact that makes one side more legible, an interface that presents the left option first in reading order, or a prompt that primes an ordering. None of those are recoverable after the fact. The run is spent.

Our threshold flags deviations beyond 25 percentage points at n ≥ 8, which is deliberately permissive: it is a screen for pathology, not a significance test.

### Inter-rater agreement

For each pair of judges, compute the rate at which they reached the same verdict on the pairs **both** answered, then take the mean across judge pairs. Restricting to the overlap matters; scoring a judge against gaps in another's coverage moves the statistic for reasons that have nothing to do with the judges.

Agreement near chance means the panel is not detecting a shared signal. Whatever each judge responded to, it was not something the others also saw. A win rate computed from such a panel is an average of unrelated preferences, and adding judges will tighten its confidence interval without making it mean more — which is the trap, because a tighter interval reads as a stronger result.

### Both gate the report

In the reference implementation either condition exits non-zero. This is a deliberate ergonomic choice rather than a statistical one: a check that merely prints a warning beside a headline number will be read past. A run that failed its preconditions has not produced a result, and the tooling should behave that way.

---

## 5. What Agreement Cannot Tell You

> **Answer:** High inter-rater agreement is consistent with two very different situations — the candidates genuinely differ, or the judges share a bias. The statistic cannot separate them, and reporting high agreement as though it establishes validity is the most common way blind panels are oversold. Agreement is a necessary condition for a meaningful result, never a sufficient one.

This is the limit worth stating plainly, because the incentive runs the other way.

A panel with 95% agreement produces a satisfying report. The judges concurred; the finding looks robust. But identical reasoning would apply to a panel of five judges who share a training distribution, a rubric, a cultural prior, or a prompt — and who therefore agree strongly about something other than quality. Model-based judges are especially exposed here: a panel of instances of the same model is not five independent observers, and their agreement measures self-consistency rather than truth.

Agreement tells you the panel responded to *something* in common. It is silent on whether that something is the property you meant to measure. Establishing that requires means outside the statistic: judges drawn from genuinely different populations, deliberately varied evaluation lenses, calibration against items with known ground truth, or an adversarial judge briefed to argue the opposite.

The honest formulation, and the one the reference implementation prints: high agreement means **either** a real difference **or** a shared bias, and this number cannot distinguish them.

---

## 6. Confidence, and Why n Is Usually Too Small

> **Answer:** Report a confidence interval beside every win rate, because eight-of-eleven and eighty-of-one-hundred-and-ten are not the same claim despite sharing a percentage. A normal-approximation interval is adequate and takes one line. At n=6 the 95% half-width is roughly ±0.40 — which is the arithmetic stating plainly that six comparisons cannot support a conclusion.

Evaluation panels are usually small, because judging is the expensive step. That is a legitimate constraint, and it becomes a problem only when the report presents a small-sample percentage with the same confidence as a large-sample one.

A normal-approximation half-width at 95% is `1.96 × sqrt(0.25 / n)`. It is not the tightest interval available and it is not the right choice at extreme proportions, but it is one line of code and it is dramatically better than omitting uncertainty:

| n | 95% half-width |
|---|---|
| 6 | ±0.40 |
| 11 | ±0.30 |
| 40 | ±0.15 |
| 100 | ±0.10 |
| 400 | ±0.05 |

The table is the argument. To separate a 55% win rate from a coin flip you need several hundred comparisons; most published panels run a few dozen and report to two decimal places. Printing the interval next to the estimate makes the mismatch visible to the reader without requiring them to do the arithmetic.

---

## 7. Separation: The Harness Does Not Judge

> **Answer:** Blinding, randomisation, unblinding, and statistics belong in tooling; deciding which candidate is better belongs to the judges. Keeping them separate is what makes a published result auditable — anyone holding the seed can re-derive the assignment and recompute the arithmetic instead of trusting a summary. A harness that also judges cannot be checked by anyone who does not rerun the judging.

The reference implementation deliberately does not call a model, score anything, or express an opinion about quality. It emits a manifest for judges and a sealed key withheld until verdicts return, then resolves verdicts against the key and computes the statistics above.

This buys three properties that matter more than convenience.

**Auditability.** The blinding is seeded and sorted, so it depends on content rather than directory listing or object key order. Publish the seed and a reader reconstructs the exact assignment and checks every number. Published evaluation results are ordinarily unfalsifiable in practice; this makes them checkable in principle and cheap to check in fact.

**Judge independence.** Because the harness knows nothing about how judging happens, the same run can mix human and model judges, or judges given deliberately different lenses. Diversity of judgment is the main defence against the shared-bias failure in §5, and a harness coupled to one judging method quietly forecloses it.

**Longevity.** Model APIs, prompt formats, and rubric conventions turn over quickly. Blinding and inter-rater statistics do not. Separating them means the durable part is not rewritten every time the disposable part changes.

---

## 8. Applying It

> **Answer:** Run a panel in four steps — prepare a blinded manifest with a recorded seed, distribute the manifest while withholding the key, collect verdicts as (pair, judge, choice) triples, then tally and refuse to report if either precondition fails. Publish the seed alongside the result so the run can be re-derived.

```bash
npx blind-panel prepare --candidates=a,b --items=q1,q2,q3 --out=./run --seed=run-1
# judges receive ./run/manifest.json — never ./run/key.json
npx blind-panel tally --dir=./run --verdicts=./verdicts.json
```

Four practices carry most of the value, and none of them require the tool:

1. **Record and publish the seed.** It converts an assertion into something a reader can reconstruct.
2. **Withhold the key until verdicts are in.** Blinding that could have been undone mid-run is not blinding.
3. **Report side balance, position bias, agreement, and a confidence interval alongside every win rate.** Four numbers. The §2 failure was caught by one of them and by nothing else.
4. **Vary your judges deliberately.** Different populations or different assigned lenses. Five instances of one model are one observer with a large error bar.

The broader claim is not that evaluation panels are unreliable. It is that a panel becomes evidence only once the conditions for interpreting it have been checked and published — and that those checks cost almost nothing relative to the judging they qualify. An organization willing to spend on a panel and unwilling to spend four numbers on validating it has bought a headline rather than a measurement.

---

## References and further reading

- [`iSimplifyMe/blind-panel`](https://github.com/iSimplifyMe/blind-panel) — reference implementation, zero dependencies, MIT
- [*The Trust Ladder: Supervised Autonomy for AI Code Review*](https://isimplifyme.com/whitepapers/the-trust-ladder) — how a non-deterministic gate earns authority on measured precision
- [*The Finite Chain: Bounding Multi-Agent Autonomy by Construction*](https://isimplifyme.com/whitepapers/the-finite-chain) — the same construction-over-monitoring argument, applied to agent autonomy
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) — measurement-first governance posture

---

Cite as:

> Elstner, Joe. "*The Blind Panel: Position Bias and Agreement as Preconditions for a Win Rate*." iSimplifyMe, 2026. https://isimplifyme.com/whitepapers/the-blind-panel
