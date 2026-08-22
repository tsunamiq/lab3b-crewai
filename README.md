# Lab 03B — A CrewAI Research Crew

UCSD *Agentic AI: Building Autonomous Intelligent Systems*, Week 3. All five "Your turn"
exercises are implemented.

The lab builds a three-agent crew — **researcher → analyst → writer** — running
`Process.sequential`, where the handoff between agents is declared with `context=` rather than
wired by hand. The topic throughout is *"the impact of AI agents on software development
workflows"*. The exercises then attack that crew: they add a verification agent, replace a prose
handoff with a typed one, price the manager in a hierarchical crew, and try to prove that
`memory=True` does something measurable.

## Files

| File | What it is |
|---|---|
| `lab03_B_crewai_research_crew.ipynb` | The notebook. 33 cells: the original 20 lab cells, byte-unchanged, then 13 added cells of exercise code. |
| `EXERCISE_NOTES.md` | Design rationale for all five exercises, plus the written answer to Exercise 5. All prose lives here so the notebook holds only code. |
| `outputs/` | Saved task outputs, one directory per run. `before-memory/` is the baseline control. |

## Notebook layout

| Cells | Contents |
|---|---|
| 0–19 | The original lab: setup, three agents, three tasks, the sequential run, the optional `memory=True` cell, and the exercise list. Untouched. |
| 20–21 | Exercise index, then shared helpers — `save_output()`, `make_research_task()`, `run_crew()` (503-tolerant), `task_text()`. |
| 22–23 | **Exercise 1** — fact-checker crew, then the deterministic drift check. |
| 24–26 | **Exercise 2** — Pydantic schema, the typed crew, then assertions on the typed output. |
| 27–28 | **Exercise 3** — sequential vs. hierarchical arms, then the cost/routing measurement. |
| 29–30 | **Exercise 4** — control and memory arms, then the entity-recall measurement. |
| 31–32 | Exercise 5 pointer (written answer is in the notes), and a repeat of the lab's exercise list. |

Run cells in order. Two dependencies are not obvious: the **Exercise 3 and Exercise 4 check cells
reuse `figures()`** defined in the Exercise 1 check cell (22–23 must have run), and **Exercise 2
builds on Exercise 1's four-agent shape**.

## The exercises

**1 — Add a fourth agent.** A Senior Fact-Checker sits between researcher and analyst, auditing
each claim against a closed verdict set (`SUPPORTED` / `UNSUPPORTED` / `OVERSTATED` /
`NEEDS-ATTRIBUTION`) and quoting claims verbatim so downstream agents can string-match them. The
writer receives `[factcheck, analysis]` — not the raw brief — which closes the loophole that
produced the drift in the baseline.

The target is a real failure already visible in `outputs/before-memory/`: *"over 20-30% [on
SWE-bench]"* → *"20-30% of complex software engineering issues"* (benchmark dropped) → *"nearly a
third"* (figure dropped, range rounded up). No agent lied; the handoff let a sourced claim decay
into a confident vague one.

The check is **Python, not an LLM** — an LLM auditing an LLM is how you get a rubber stamp — and it
carries a **positive control**: it runs against the baseline first and `assert`s that it finds the
drift we already know is there. A clean verdict from a checker that cannot detect known-bad input
would mean nothing.

**2 — Structured handoff.** `analysis_task` gets `output_pydantic=AnalysisOutput`, so the writer
receives typed data instead of prose. Ranked `Implication` objects with `Literal` impact and
confidence fields, `Recommendation` objects, and a `dropped_claims` field that carries Exercise 1's
audit trail through the schema. The check cell asserts things that are *impossible* to write against
the prose baseline: contiguous ranks with no ties or gaps, enum membership, confidence downgraded
when a claim rests on a flagged one. That impossibility is the argument for `output_pydantic`,
stated as code rather than as a claim.

**3 — Swap the process.** Both arms run **the same three context-free task descriptions** and
differ only in who wires them: `sequential` hard-wires `agent=` and lets CrewAI chain outputs;
`hierarchical` leaves `agent=` unset and a `manager_llm` agent routes. Keeping `context=[...]` here
would have been the trap — the manager would have nothing left to decide, and the run would be a
sequential run with a bigger bill.

One run per arm suffices because the load-bearing numbers are **structural, not sampled**:
hierarchical mode inserts manager turns whatever the model says, so `total_tokens` and
`successful_requests` are properties of the topology and don't average away over reruns. Wall clock
and specifics-propagation *are* sampled — printed, but excluded from the verdict. The check reports
the token multiple, the manager's extra LLM calls, and routing fidelity from `task.output.agent`.
The expected 3/3 routing agreement is the finding, not a failure: the manager pays a multiple to
rediscover a wiring already known, so hierarchical mode earns its keep only when routing *isn't*
known up front. Note the sequential arm is not unwired — with no `context=`, `Process.sequential`
still passes each task its predecessor's output, so this measures manager-routing vs. built-in
chaining.

**4 — Prove memory works.** The lab's optional cell runs the same crew on the same topic with the
same prompts twice, then invites you to notice Run 2 resembles Run 1 — but identical prompts to a
temperature-0.3 model already produce similar output, so it cannot distinguish recall from
repetition. Two changes fix that: Run 2 asks for specifics only memory could supply (a *"What I
established last time"* section, with an explicit `NO PRIOR CONTEXT RECALLED` escape so an honest
blank is distinguishable from a guess), and there is a **`memory=False` control arm** — on a shared
topic most cross-run overlap is topic, not memory, so only the *difference between arms* is
evidence. The check measures entity overlap per arm and reports the delta.

**5 — Compare to Lab 03A.** Written answer in `EXERCISE_NOTES.md` — hand-rolled memory versus
framework memory, which was clearer, and which to reach for in production.

## Outputs

| Directory | Produced by |
|---|---|
| `before-memory/` | Baseline three-agent sequential run (cell 15). The control everything is measured against. |
| `after-factchecker/` | Exercise 1's four-agent crew — brief, audit, analysis, report. |
| `typed-handoff/` | Exercise 2 — `analysis.json` (typed) and `final_report.md`. |
| `exercise3-sequential/`, `exercise3-hierarchical/` | Exercise 3's two arms, for a side-by-side read of any task the manager routed differently. |
| `exercise4-control-no-memory/`, `exercise4-with-memory/` | Exercise 4's two arms. |

No added cell reuses the original `research_task` / `analysis_task` / `writing_task` objects. A
`Task` stores its result on `task.output`, so handing an original to a new `Crew` would overwrite
the baseline in place — hence the `make_research_task()` factory, whose prompt text is copied
verbatim so the added agent stays the only variable.

## Running it

Built for Google Colab. Add a `GEMINI_API_KEY` Colab secret (key icon, notebook access enabled)
and run top to bottom. Model is `gemini/gemini-3.5-flash` at `temperature=0.3` with
`num_retries=5` — the lab deliberately avoids `flash-lite`, which returns HTTP 503 too often for
requests of this size.

Three things worth knowing before you run:

- **Transient 503s are expected.** `run_crew()` catches them and reports which arm didn't finish
  rather than crashing the cell. Re-run in a few minutes.
- **Exercise 3's hierarchical arm is the expensive cell** — manager turns multiply the token bill
  for the same three tasks. That multiple is the measurement, but budget for it.
- **Exercise 4 needs a clean runtime for memory.** `CREWAI_STORAGE_DIR` is resolved when CrewAI
  builds its memory objects, so if the optional `memory_crew` cell (18) already ran this session,
  restart the runtime first. The cell wipes the store between arms with a handle-tolerant helper —
  a plain `shutil.rmtree` raises `OSError 39` when an open ChromaDB client recreates files
  mid-delete, so it retries and then falls back to renaming the directory aside.

A `FAIL` or a null result from any check cell is a legitimate outcome, not a broken cell. Exercise
1's checker only *sometimes* catching known-bad output, or Exercise 4 showing no memory effect, is
exactly the kind of finding Exercise 5 should be arguing about — record it rather than tuning the
prompt until it passes.
