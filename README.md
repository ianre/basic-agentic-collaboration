# basic-agentic-collaboration protocol

A protocol for AI agents and humans to work on issues together, where the agent proposes and the human executes.

## Workflow

1. **Task setup** — Each project lives in `coworking/YYYY-MM-DD-<task-slug>/` with a `README.md`, `scripts/`, and `evidence/`. Use `coworking/new-task.sh` to create one. Before the first script, list candidate `Threads to consider` (cross-repo leads from `coworking/repo-map.md`).

2. **Proposal-and-review loop** — The agent writes a numbered, focused script (`scripts/NN-<slug>.sh`) declaring `# MUTATES: no` (or `yes` with justification). The human reviews and runs it; output lands in `evidence/NN-<slug>.txt`. The agent reads the evidence, updates the task README with findings, and proposes the next script. Repeat.

3. **Constraints** — Agent scripts are Read-only by default; mutations require written justification and human approval.

4. **Tracing** — Trace failures outward through the entirety of the execution chain rather than stopping at the visible symptom.

5. **Stop condition** — Before marking `resolved` or `blocked-on-human`, address every entry in `Threads to consider` (evidence pulled, or one-line out-of-scope note). Generic advice like "retry" or "escalate" is not a valid stopping point without a specific definition, pin, or parameter identified as the locus of failure.

6. **Evidence hygiene** — `coworking/*/evidence/` is gitignored by default. The agent writes sanitized summaries into the task README; the human decides what is safe to commit.

## Independence, provenance, and institutional disclaimer

This repository contains an independently authored scientific and technical artifact for describing agent-facing interaction protocols through `AGENTS.md`-style files.

I created this work independently while enrolled as a graduate student. I did not use employer equipment, employer repositories, internal employer documents, confidential employer information, or paid employer work time. This work was not created as part of my assigned employment duties.

This work was developed in connection with a graduate course. However, to the best of my knowledge, this work was not created under a sponsored research project, grant, assistantship, industry-sponsored collaboration, or university administrative assignment. I did not use restricted university resources beyond ordinary student resources available for coursework.

I have not completed a formal intellectual-property review by my employer, the university, or legal counsel. This notice is a good-faith provenance statement only. It is not legal advice, legal clearance, a warranty of title, or a representation made by my employer, university, instructor, or any other institution.

Any employer, university, or institutional affiliation is provided only for biographical context. This project is not endorsed by, sponsored by, or affiliated with my employer or university. The views, design choices, and claims in this repository are my own.

