# Baseline run — before memory was enabled

Output of the three-agent sequential crew from `lab03_B_crewai_research_crew.ipynb`
(notebook cell 15), captured **before** `memory=True` was turned on. This is the
control to diff later runs against.

- Crew: researcher -> analyst -> writer, `Process.sequential`
- Handoff: task-local `context=` only; no cross-run memory
- Topic: "the impact of AI agents on software development workflows"
- Agents: 3 (no fact-checker yet — that's Exercise 1)

| File | Task | Context received |
|---|---|---|
| `research_brief.md` | `research_task` (researcher) | none (first step) |
| `analysis.md` | `analysis_task` (analyst) | `[research_task]` |
| `final_report.md` | `writing_task` (writer) | `[research_task, analysis_task]` |
