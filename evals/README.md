# Trigger evals

One case directory per prompt, grouped by skill: `<skill>/fires-1`, `fires-2` are prompts the
skill should fire on, `miss-1`, `miss-2` are near-misses it should stay out of (another skill
may fire there; only the named one is asserted). Each case grants the `Skill` tool alone and
stops after three turns, because the question is whether the description triggers, not what
the skill does once loaded. Most runs therefore end with "Reached maximum number of turns";
that is expected and does not affect the score, which comes from the grader alone.

Run the whole suite, or one skill, after editing a `description`:

```bash
claude plugin eval . --ablation none --tag trigger --no-publish
claude plugin eval . --ablation none --tag eod-summary --no-publish
```

`--ablation none` matters: under the default with-versus-without run, `tool_used: Skill`
graders are reported as plugin-fired indicators rather than scored, so the should-fire cases
would never fail. Results land in `evals/results/`, which is ignored. Add `-j 6` to run
cases in parallel; the whole suite is 44 cases.
