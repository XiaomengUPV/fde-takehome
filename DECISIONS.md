# Decisions

## What I deliberately did not do, and why

`execute_step` (provided) calls `chat`/`chat_with_tools` with no try/except,
unlike `write_plan` and my `revise_plan`, which both swallow model errors into
a degraded fallback. In replay mode this never surfaces, but in `live` mode a
timeout or rate-limit inside a step would raise up through `run_planning_agent`
and crash the whole run, even though the docstring promises "this function
does not raise for ordinary failures." I left it alone because it's in the
file explicitly marked provided and no visible test exercises it, but it's the
first thing I'd fix before pointing this at a real endpoint — wrapping the
`execute_step` call in `run_planning_agent` with a try/except that degrades to
a `status="error"` `StepResult` would close the gap without touching the
provided function itself. I also didn't add any normalization for the
observation prefix (`surprise:`/`ok:`/`thin:`) beyond `str.startswith` — the
observer's system prompt pins the format and every fixture honors it, so
defending against a differently-cased or -punctuated reply felt like
speculative robustness rather than a real risk worth the complexity.

## How I would know this works

In replay mode, `make test` is the gate: 24 deterministic contract tests, run
on every change, zero cost — a red run means something is actually broken,
not "the model felt different today." That's necessary but not sufficient,
since replay can't tell me the *prompts* still produce sane plans against a
real model. For that I'd run a fixed eval set of ~25 representative goals
(budget-constrained, multi-day, ambiguous, non-visual) against `LLM_MODE=live`
at low temperature, 3 trials each, and score structurally: plan length within
3-5 steps, zero uncaught exceptions, every search-backed final answer carrying
at least one `[url]` citation, and the revision rate per run. The revision
rate is the one I'd watch hardest — near 0% across all 75 trials suggests the
observer never flags real surprises, and above roughly 40% suggests it's
crying "surprise" on ordinary results and burning the replan budget for
nothing; either extreme is the "quietly broken" signal, not a crash.

---

**AI assistance:** Built with Claude Code (Sonnet 5) end to end — it read the
brief and existing code, proposed the plan for A1-A3 which I reviewed and
approved before any edit, wrote the implementation, ran the offline test
suite after each piece, and drove the Streamlit UI with a throwaway Playwright
script to screenshot the approve gate, the step trace, and a mid-run revision
card before calling A3 done. I read every diff against the docstrings myself
rather than trusting green tests alone, since the visible suite is stated to
be a floor, not the full grading contract.

**Time spent:** Roughly one focused session: reading the brief/code and
planning, implementing A1/A2/A3, running the contract suite plus one
end-to-end terminal and browser check, and writing this file.
