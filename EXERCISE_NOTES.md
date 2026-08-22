# Exercise notes — Lab 03B

Reasoning, expected results, and the written answer to Exercise 5. Kept out of
`lab03_B_crewai_research_crew.ipynb` so the lab's own text stays unchanged; the notebook holds
only code, each cell headed by an `EXERCISE n` banner comment.

| Exercise | Where |
|---|---|
| 1. Fact-checker agent | notebook, 2 cells (build + check) |
| 2. `output_pydantic` | notebook, 3 cells (schema + crew + check) |
| 3. `Process.hierarchical` | notebook, 2 cells (arms + measurement) |
| 4. Prove memory works | notebook, 2 cells (arms + measurement) |
| 5. Compare to Lab 03A | **this file**, below |

One rule held throughout: no added cell reuses the original `research_task` / `analysis_task` /
`writing_task`. A `Task` stores its result on `task.output`, so passing an original to a new
`Crew` overwrites the baseline in place — and `outputs/before-memory/` is the control everything
is measured against. Hence the `make_research_task()` factory, whose prompt text is copied
verbatim from the original so the added agent is the only variable.

---

## Exercise 1 — the fact-checker

### The drift this is aimed at

The baseline run already contains the exact failure a fact-checker should catch. One claim decays
across all three hops:

| Hop | What it says |
|---|---|
| `research_brief.md` | "The industry standard … is **SWE-bench** … state-of-the-art agentic systems now resolve **over 20-30%** of these … problems autonomously" |
| `analysis.md` | "state-of-the-art agents can now resolve **20-30%** of complex software engineering issues" — *SWE-bench is gone* |
| `final_report.md` | "cutting-edge agents can now solve **nearly a third** of complex engineering issues" — *the figure is gone too* |

Three things went wrong, and only the first is the researcher's fault:

1. **"over 20-30%" is incoherent as written** — a floor and a range at once.
2. **The analyst dropped the attribution.** The number survived; the benchmark that gives it
   meaning did not.
3. **The writer rounded an unsourced range up** into "nearly a third" — the top of the range
   presented as the typical case.

No agent lied. The handoff let a specific sourced claim decay into a confident vague one, and
because each hop sees only text, never provenance, no agent in the baseline crew is positioned
to notice.

### Design choices

- **The writer does not receive the raw brief** (`context=[factcheck, analysis]`, where the
  baseline had `[research, analysis]`). The analysis already carries the brief's substance, the
  lab's own "context is not free" warning argues against a third full document in the last
  prompt, and withholding it closes the loophole that produced the drift — the writer can no
  longer reach past the audit to the unqualified original.
- **The auditor is told twice not to add facts.** An auditor that starts researching is just a
  second researcher, and its inventions would arrive downstream wearing the authority of a
  verification step.
- **Verdicts are a closed set** (`SUPPORTED` / `UNSUPPORTED` / `OVERSTATED` /
  `NEEDS-ATTRIBUTION`) and claims are quoted verbatim, so downstream agents — and the checker —
  can match them by string.

### Why the check is Python, not an LLM

An LLM auditing an LLM is how you get an agreeable rubber stamp. Everything objectively decidable
is decided in code, the same split as Lab 03A's deterministic checker.

`drift_report()` pulls precise figures from the upstream document and vague numeric stand-ins from
the downstream one. A figure that vanishes while a stand-in appears *is* the drift.

**The positive control is the important part.** The checker runs against the baseline first and
`assert`s. If it cannot find the drift we already know is in `outputs/before-memory/`, then its
clean verdict on Exercise 1 means nothing — so it must fail loudly instead. Verified locally: it
isolates hop 1 losing `SWE-bench` and hop 2 substituting `20-30%` → "nearly a third", and does
not false-fire on hop 1.

One deliberate narrowing: only fraction-style stand-ins ("a third", "the majority") count as
drift. An earlier version also counted bare "most"/"some", which would fire on any hop that
merely dropped a figure and happened to use the word — a false failure on most runs. Soft hedges
are reported but never decide the verdict.

### Reading the Exercise 1 result

A `FAIL` is a legitimate result, not a broken cell. Write down which check failed and why. A
verification agent that only *sometimes* catches a known-bad claim is exactly the finding
Exercise 5 should be arguing about. Equally, "no drift this run" on a single sample is weak
evidence either way — re-run to see whether it is stable.

---

## Exercise 2 — structured handoff

The analyst currently hands the writer prose, which is a contract nobody can enforce:
"Key Implications (3 bullets)" is a request, and a consumer wanting implication #1 has to go find
it with a regex.

What the schema buys:

1. **`analysis_task.output.pydantic` is a real object.** `.implications[0].claim` addresses a
   field. A missing field is an error at the boundary instead of a silent miscommunication three
   hops later.
2. **Ranking becomes checkable.** "Rank by impact" is unverifiable in prose; with `rank: int` it
   is an assertion.
3. **The audit gets a guaranteed lane.** This is where Exercise 2 collides with Exercise 1. The
   fact-checker's "do not repeat downstream" list is prose living inside the analyst's prose
   output. Constrain the analyst to a schema with no field for dropped claims and the audit trail
   is the first thing squeezed out — the schema would silently delete Exercise 1's safety
   feature. Hence `dropped_claims`.

Two things that break quietly if skipped:

- **`expected_output` must describe the JSON**, not "a ranked analysis". CrewAI puts both the
  schema and `expected_output` in the prompt; a prose-shaped one pulls against the schema and you
  get schema-shaped prose or a retry loop.
- **The writer now receives serialized JSON.** Its description must say so and name the fields,
  or it echoes field names into the report — reports containing "executive_summary:" are the
  classic symptom.

Every assertion in the check cell is impossible to write against
`outputs/before-memory/analysis.md`: no rank to compare, no enum to validate, no list of dropped
claims. That is the argument for `output_pydantic`, stated as code rather than as a claim.

---

## Exercise 3 — `Process.hierarchical`, priced against sequential

### The trap

A hierarchical crew needs `manager_llm` **or** `manager_agent` (never both), and the manager must
not appear in `agents=`. But the subtler trap is the tasks: if they keep their explicit
`context=[...]` wiring, the manager has nothing left to decide, and the run is a sequential run
with a much larger token bill.

So **both arms use the same three context-free task descriptions**, and differ only in who does
the wiring:

| | `sequential` | `hierarchical` |
|---|---|---|
| `agent=` | hard-wired by us | left unset — the manager assigns |
| handoff | CrewAI passes the previous task's output forward | the manager is the only carrier |

Note that the sequential arm is *not* unwired: with no `context=`, `Process.sequential` still
hands each task its predecessor's output automatically. The comparison is therefore
manager-routing vs. built-in chaining, not routing vs. nothing.

### Why one run per arm is enough

The load-bearing numbers — `total_tokens` and `successful_requests` from `crew.usage_metrics` —
are **structural, not sampled**. Hierarchical mode inserts manager turns whatever the model
happens to say, so the call count and token bill are properties of the topology; rerunning does
not average them away. This is the opposite of Exercise 4, where the measured quantity was
semantic recall and a single sample proved almost nothing.

Wall clock and specifics-propagation *are* sampled. Both are printed, neither drives the verdict.

### What the check cell measures

1. **Price of autonomy** — hierarchical ÷ sequential `total_tokens` for identical work.
2. **Manager overhead** — extra `successful_requests`, i.e. the manager's own turns.
3. **Routing fidelity** — `task.output.agent` per task vs. the assignment we would have
   hand-wired. The interesting outcome is 3/3: the manager spends multiples of the tokens to
   rediscover a wiring that was already known. That is the finding, not a failure — hierarchical
   mode earns its bill only when the routing is *not* known up front (task count unknown, agent
   chosen by intermediate results, work rejected and retried).
4. **Propagation** — specifics from the brief surviving into the final report, per arm. Uses the
   same named-system-or-figure measure as Exercise 4's `entities()`; the definition is repeated
   locally because Exercise 4's cell runs later. Its known limitation is inherited: a plain
   capitalised name with no internal capital, hyphen or digit (`Copilot`, `Devin`) is not counted,
   because admitting those would also admit every sentence-initial word.

Outputs land in `outputs/exercise3-sequential/` and `outputs/exercise3-hierarchical/` for a
side-by-side read of the disagreeing task.

---

## Exercise 4 — prove memory works

### Why the lab's optional cell does not prove it

`memory_crew` runs the same crew, on the same topic, with the **same task descriptions**, twice,
then invites you to notice that Run 2 resembles Run 1. But identical prompts to a
temperature-0.3 model already produce similar output. Every bit of that resemblance is explained
without memory, so the demo cannot distinguish "the crew recalled Run 1" from "the crew was asked
the same question twice." It also overwrites `writing_task.output`, destroying the baseline
objects.

### What was changed

1. **Run 2 asks for something only memory can supply** — a section listing the specific systems
   and figures established last time, with an explicit `NO PRIOR CONTEXT RECALLED` escape so an
   honest blank is distinguishable from a guess.
2. **A `memory=False` control.** On a shared topic, most of the overlap between two runs is the
   topic, not memory — any competent researcher names SWE-bench both times. The control measures
   that floor. **Only the difference between arms is evidence.**
3. **The control runs first**, while no store exists, so it cannot be contaminated even if the
   wipe fails.
4. **`CREWAI_STORAGE_DIR` is set and wiped.** By default CrewAI writes to a hidden platform
   app-data directory; pointing it somewhere visible makes the store inspectable *and* wipeable.
   The wipe matters because the lab's optional cell may already have seeded it with runs of this
   exact topic.
5. **Each arm mutates `crew.tasks` between runs** rather than building a second crew. A `Crew`
   builds its memory objects at construction, and in some CrewAI versions short-term memory is
   scoped to that instance — so reusing it gives memory its best possible shot. The control
   performs the identical mutation, so the arms differ in exactly one flag.

### Reading the result

**A null delta is a legitimate finding, not a failure.** Plausible causes:

- CrewAI short-term memory is scoped per-`kickoff` in some versions, so Run 2's crew never
  queried Run 1's store despite sharing the instance.
- Long-term memory stores task-level quality evaluations, not research content — so there may be
  nothing there for a researcher to recall.
- Retrieval is embedding similarity with a relevance threshold; the Run 2 prompt may simply not
  retrieve anything above it.

**Do not tune the Run 2 prompt until it looks like recall.** That measures the prompt, not the
memory.

Watch the confabulation row too. The control is *told* it has a history; if it declines the
offered blank and names specifics absent from its own Run 1, it manufactured them. Anything
relying on a model's self-report of what it remembers is measuring compliance with the prompt,
not recall.

Runtime: four full three-agent runs, ~12 LLM calls plus embedding calls.

---

## Exercise 5 — framework memory vs. hand-rolled memory

I built memory by hand in Lab 03A and got it from the framework here. They are not the same
mechanism wearing different clothes.

### What each one actually is

**Lab 03A** — memory is a Markdown file, `/content/reflexion_memory.md`, wrapped by
`ReflexionMemory` with exactly two doors: `remember(lesson)` appends a line, `recall(last_n)`
reads lines back. What gets stored is a *deliberate distillation* — the reflector turns a critique
into one sentence starting "Next attempt:", and that sentence *is* the memory. Retrieval is
last-N: deterministic, ordered, complete.

**Lab 03B** — memory is `memory=True`. CrewAI stands up three stores (short-term, long-term,
entity), embeds content through Gemini, and retrieves by vector similarity. What gets stored, I
did not choose. What comes back, I cannot predict.

### The comparison

| | Lab 03A (by hand) | Lab 03B (CrewAI) |
|---|---|---|
| Lines of code | ~40 for the class + reflector | one kwarg |
| What is stored | one distilled sentence per iteration, by my design | the framework's choice across 3 stores |
| Where it lives | a Markdown file I can `cat` | SQLite + Chroma under `CREWAI_STORAGE_DIR` |
| Retrieval | last-N: deterministic, ordered | embedding similarity above a threshold |
| Inspectable | open the file | dump a vector DB and infer |
| Testable | `recall()` is a pure function — assert on it | assert on *whether* recall happened, indirectly |
| Clearing stale memory | `memory.reset()`, a documented method | find and delete a storage directory |
| Failure mode | wrong lesson stored — visible in the file | nothing retrieved — silent, looks like normal output |
| Cost | zero extra calls | embedding calls per store and per retrieval |
| Portability | a Markdown file | CrewAI's schema |

### The thing I did not expect

The framework's memory is **harder to debug than the hand-rolled one, in the specific way that
matters most**: when it silently does nothing, the output still looks fine. Exercise 4 needed a
control arm, an entity-overlap metric, and four crew runs to answer "did it recall anything?" —
and even then the answer is statistical. In 03A that question is `print(memory.recall())`.

That asymmetry generalizes past memory. The same invisibility produced the drift Exercise 1 fixed:
`context=[research_task]` is a beautifully compact way to express a handoff, and it is precisely
*because* the handoff is one word that nobody looks at what crosses it. A number lost its source
between researcher and analyst, then lost its precision between analyst and writer, and the crew
reported success. Hand-rolling that chain, I would have been holding the intermediate string in a
variable, and would likely have read it.

So the framework did not save me the work. It moved the work from *writing the mechanism* to
*proving the mechanism ran*. On this lab, the second job was bigger.

### Which I would reach for in production

**CrewAI for the orchestration; something explicit for the memory.**

The crew abstraction earns its place. Sequential handoff, per-task context declaration,
`output_pydantic` at the boundary, four agents wired in a dozen lines — I would not hand-roll
that, and Exercise 2 shows the payoff: a typed boundary turns "rank by impact" from a hope into an
assertion.

`memory=True` I would not ship as-is, for one reason: **I cannot answer "why did the agent say
that?"** In production that question arrives attached to an incident. A store I did not design,
holding content I did not choose, retrieved by a similarity I cannot replay, is an input to my
system that I cannot reconstruct after the fact. The lab's own warning — memory is "powerful and
sticky" and can "carry stale or wrong facts forward" — describes a risk you can only manage if you
can *see* what is in there.

What I would build is 03A's shape at 03B's scale: an explicit store whose write path I control,
whose contents are readable, and whose retrieval is deterministic enough to reproduce. Not a
Markdown file — a real datastore with retrieval I chose. But the same property, the one 03A had
and `memory=True` gives up: **I can read the memory and say what the agent knew.**

The honest summary: 03A was clearer because I built it, and I would keep that clarity for state
that outlives a run. 03B was faster because I did not, and that trade is fine for orchestration,
where a mistake shows up in the output. For memory, a mistake shows up as confidence.
