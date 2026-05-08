# AGENTS.md

## What This Document Is

This file establishes the protocol by which AI agents collaborate with human
operators in this repository. It is read by every agent that begins work here,
before any other action.

The protocol exists because the work performed here — investigating
pipeline failures, tracing configuration across repositories, and coordinating
remediation — is interpretive and context-dependent in ways that demand human
judgment, while also benefiting from the systematic reasoning and bookkeeping
that AI agents provide. The collaboration loop described below is designed to
combine those capabilities without granting agents any independent access to
production systems.

## The Credential Model

Agents working through this protocol have **no credentials of any kind**. They
do not authenticate to cloud providers, version control hosts, internal
services, or any other resource. They do not invoke MCP servers that would act
on the operator's behalf. Every action that touches a real system is executed
by the human operator, under the operator's own identity, on the operator's
machine.

This is not a workaround. It is the design. The audit trail in every system
the operator touches correctly attributes every command to the human, because
the human is the actor. Agents propose, reason, and document; humans execute.
The boundary between these roles is the folder structure described below.

## The Coworking Directory

All collaborative work lives under `coworking/`. Each investigation is a dated
subdirectory of the form `coworking/YYYY-MM-DD-<task-slug>/`, created from
`_template/` via the helper `coworking/new-task.sh`. Inside each task
directory:

- `README.md` is the task's narrative — its goal, current status, hypotheses,
  threads under consideration, threads followed, findings, and the next
  proposed step.
- `scripts/` holds numbered shell scripts proposed by the agent
  (e.g. `01-fetch-pipeline-logs.sh`).
- `evidence/` holds the corresponding outputs
  (e.g. `01-fetch-pipeline-logs.txt`). This directory is gitignored by default.

The numbering matters. It establishes a chronological record of what was
asked, what was learned, and how the investigation progressed. Out-of-order
or unnumbered scripts break the audit trail.

## The Investigation Loop

The protocol is a turn-based cycle:

1. The agent reads the task README and any prior evidence, then proposes the
   next script. The script is written to `scripts/NN-<slug>.sh` and explained
   in the README: what question it answers, what evidence would change the
   agent's hypothesis, and why this is the cheapest informative step
   available.
2. The human reviews the script for safety and intent, then runs it. The
   script writes its output to `evidence/NN-<slug>.txt`.
3. The agent reads the evidence, updates the README with findings, and either
   proposes the next script or proposes a stopping condition.

This loop holds until the task reaches a terminal state — `resolved`,
`blocked-on-human`, or `blocked-on-evidence`. Skipping a step (running a
script the agent wrote without review, or having the agent assume what
evidence would have shown) breaks the protocol.

## Cheapest Evidence First

Before proposing a script that calls any external system, the agent should
ask whether the question can be answered from artifacts already in the
repository or already on disk. Pipeline definitions, version pins, included
templates, and configuration files often contain the answer to questions
about what *should* have happened. External commands answer questions about
what *did* happen.

Agents that reach for the CLI when the YAML would have answered the question
waste the operator's review time and obscure the reasoning trail. State
explicitly in the README when a script is proposed because the in-repo
evidence has been examined and found insufficient.

## Tracing Failures Across Repositories

A pipeline failure is almost never self-contained in the repository where it
surfaces. The execution chain runs from the trigger configuration to the
pipeline definition, to called templates and shared scripts, to the
components those templates invoke, to the modules those components depend
on. This chain routinely crosses repository boundaries. The visible error is
the end of the chain; the cause is somewhere upstream.

When a failure log identifies a component as the proximate failure, the
agent's next question is not "what does this component do" but **"where is
this component invoked, from what definition, with what version pin, and
does that definition live here or in another repository?"** Trace from
configuration outward, not from the symptom inward.

In practice this means working from two ends at once.

From the **configuration end**, examine the checked-in pipeline definitions,
the templates they include, the default values in shared functions, and the
version pins on every external reference.

From the **execution end**, examine the logs to see what values were
actually used, which scripts ran, and which step produced the error.

The cause is found where these two ends meet — typically a mismatch between
what the configuration declared and what the execution resolved it to. When
the trace crosses into another repository, that repository becomes an
investigation thread; consult `coworking/repo-map.md` to locate its AGENTS.md
and understand its role in the pipeline.

## The Repository Map and Threads

`coworking/repo-map.md` indexes the repositories that may bear on
investigations originating here: their local paths or URLs, their AGENTS.md
files, their role in the pipeline, and the failure modes for which they are
likely sources of evidence.

At the start of every task, after stating the goal, the agent reads the repo
map and records candidate threads in the task README under
`Threads to consider`. Each thread is one line: which repo, why it might bear
on the task. As the investigation progresses, threads move to
`Threads followed` with a one-line note on what was learned or why the thread
closed.

## Stop Conditions

Before a task is marked `resolved` or `blocked-on-human`, the agent must:

- Confirm that the trace described above has reached a specific definition,
  version pin, or configuration parameter that can be identified as the
  locus of failure — **or** record specifically what evidence would be needed
  to complete the trace and why it cannot be obtained.
- Review `Threads to consider` and account for each one, either by recording
  evidence pulled from the thread or by noting in one sentence why the
  thread was out of scope.

A finding that consists only of generic recommendations — retry the
pipeline, escalate to the component owner, check service status — is not a
valid stopping point. These actions may be the right response, but only
*after* the chain has been traced and the locus identified. Recommendations
without a trace mean the investigation stopped early.

## The Mutation Policy

Scripts are read-only by default. Every script must begin with one of:

- `# MUTATES: no`
- `# MUTATES: yes — <one-line justification>`

A script is mutating if it calls any command that creates, modifies, deletes,
or rotates a real resource — including triggering pipelines, restarting
components, modifying configurations, or writing to anything outside its own
evidence file. When in doubt, declare mutation.

Before proposing a mutating script, the agent must record in the task README
the justification for the mutation, the expected effect, the rollback path,
and any alternative read-only approaches that were considered and rejected.
The human makes the final decision on whether to run it.

## Script Constraints

Scripts must be:

- POSIX-compliant shell (not bash-specific features), under 40 lines,
  beginning with `set -euo pipefail`.
- Self-contained: one focused question per script. If the question requires
  more, split it across multiple numbered scripts.
- Limited to the CLI tools and standard utilities already available in the
  repository's environment. No `curl | sh`, no inline base64, no
  obfuscation.
- Restricted to writing only their corresponding `evidence/NN-<slug>.txt`
  file.

These constraints exist so the human's review is cheap. A forty-line POSIX
script doing one thing can be read in under a minute. A two-hundred-line bash
script with conditional logic cannot, and the load-bearing element of this
protocol is the operator's review. If review becomes expensive, the operator
will start skimming, and the protocol fails silently.

## Evidence Handling

Pipeline outputs frequently contain account identifiers, internal URLs,
CRNs, usernames, and other sensitive data. The default `.gitignore` excludes
`coworking/*/evidence/` so raw evidence is never committed by accident. The
human decides on a per-task basis whether any portion of an evidence file is
safe to extract into the README.

Agents writing summaries into the README should sanitize as they go: prefer
patterns and counts over identifiers, and prefer descriptive names ("the
build component") over CRNs and account IDs unless the specific identifier
is genuinely necessary to the finding. The human confirms each summary
before commit.

## A Worked Example

The pipeline `pipeline-frontend` fails on run `r-2026-04-...`. The
operator opens a task and provides the IDs.

**Goal:** Identify why the run failed and propose a remediation.

**Threads to consider:**
- `pipeline-templates` — pipeline definitions are sourced here.
- `build-component` — the visible failure mentions a build step.

**Script 01** (`scripts/01-fetch-execution-logs.sh`):

```sh
#!/bin/sh
# MUTATES: no
set -euo pipefail
ibmcloud dev pipeline-runs-get "$PIPELINE_ID" --run-id "$RUN_ID" \
  > evidence/01-fetch-execution-logs.txt
```

Evidence shows a timeout in a step named `apply-infrastructure`. The agent
updates the README: timeout in `apply-infrastructure`. Two open questions
follow — which template defines this step, and what version of the
underlying component is pinned. The next script does *not* re-fetch the same
logs at higher verbosity; it reads the pipeline definition in this
repository to find where `apply-infrastructure` is defined. That definition
turns out to come from `pipeline-templates@v3.4.1`. The `pipeline-templates` thread is
now active, and the next script reads its AGENTS.md to learn how the
template's history is tracked.

The cycle continues. Each script answers one question. Each evidence file is
read before the next script is written. The README accumulates a trail that
another operator — or another agent — could follow without asking questions.

## Getting Started

When an agent begins work here:

1. Check `coworking/` for an active task. If one exists, read its README in
   full before doing anything else. If not, create a new task with
   `coworking/new-task.sh <task-slug>`.
2. State the goal in the task README in one or two sentences.
3. Read `coworking/repo-map.md` and populate `Threads to consider`.
4. Examine the in-repo configuration relevant to the goal before proposing
   any external command.
5. Propose the first script with a clear statement of what question it
   answers and what evidence would change the hypothesis.

Then wait. The operator runs the script. The loop begins.
