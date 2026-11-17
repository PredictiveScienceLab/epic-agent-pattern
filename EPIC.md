# EPIC: instructions for an agent

You are building, or acting inside, an **EPIC** system: *Expert Performance through Iterative
Critique*. This file tells you what the pattern is and how to build one, step by step. Follow it in
order. Where it says MUST, do not skip the step; where it says SHOULD, skip it only with a reason you
write down.

A worked example is in [`examples/ultrasound/`](examples/ultrasound/). Read it when a step below is
unclear: every step has a counterpart there.

---

## 1. The pattern in one paragraph

An expert does not produce a result in one shot. They act on an object with a small set of moves they
know, look at what came out, and correct it until nothing worth correcting is left, or they say they
cannot do better. EPIC gives an agent the same three things: a **language** in which the expert's
moves can be written down, an **executor** that carries a program in that language out and checks
it, and a **critic** that shows the result in the views an expert would use. The agent then loops:
observe, write a program, execute, critique, revise. The deliverable is the program and its result,
both of which a person can read, replay and correct.

```
            ┌──────────────────────────────────────────────────────┐
            │                                                      │
 evidence ─►│ OBSERVE ─► WRITE (language) ─► EXECUTE (executor) ─► │
            │     ▲                                   │            │
            │     └────── CRITIQUE (critic: views, checks) ◄──┘    │
            │                                                      │
            └──────► STOP: program + result + stated doubt         │
                     (or ABSTAIN: why it cannot be done)
```

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Object** | The thing the expert works on: a contour, a mesh, a schedule, a circuit, a document. |
| **Move** | One expert action on the object, at the granularity an expert thinks in ("move the apex 3 mm up", not "set pixel 4,512"). |
| **Language** | The set of moves, their parameters, and the rules for combining them into a **program**. |
| **Executor** | Code that turns a program into a concrete result, enforces the constraints, and refuses moves that would break them, with a reason. |
| **Critic** | Everything returned after execution so the result can be judged: renderings, measurements, constraint checks, comparisons with reference examples, and the written instruction for how to judge. |
| **Agent** | The model that observes, writes programs, reads the critic's output and decides the next edit or to stop. |
| **Benchmark** | A fixed set of cases with a scoring rule, used to measure the system. The agent being scored never sees the answers. |
| **Harness** | The code that prepares cases, runs agents, collects their outputs, audits what they did, and scores them. |

## 3. Invariants

These hold in every EPIC system. If a design breaks one, it is not EPIC; fix the design.

1. **No task-specific training.** The model's weights are not changed for the task. Knowledge enters
   through the language, the executor, the critic, written instructions and a few reference examples.
2. **Programs, not outputs.** The agent's result is a program in the language. The executor produces
   the output from it. Every result can be replayed from its program.
3. **The executor guards validity.** Constraints that must always hold (physical, anatomical, legal,
   type) are enforced by construction or checked before a result is returned. An invalid move is
   refused with an explanation the agent can act on; it is never silently repaired.
4. **The critic shows; it does not decide.** It renders and measures. The agent judges. A number the
   critic prints is evidence, not a verdict.
5. **The agent may stop or abstain.** It stops when it finds no specific, supported revision. It says
   what it still doubts. It may decline a case it cannot do.
6. **Answers stay hidden.** During a task the agent never sees the benchmark's answer for that case,
   scores, or other agents' work. The harness enforces this and audits it afterwards.
7. **Every claim is measured against noise.** A change counts as an improvement only if it beats the
   variation between two runs of the unchanged system on the same cases.

---

## 4. How to build an EPIC system

Four phases. Do them in order. Do not start phase B before phase A is finished, and do not touch the
held-out cases until phase D.

### Phase A. Pick a benchmark you can score

You MUST have a way to score results automatically before you build anything else.

1. **Choose cases with known good answers** made by experts, from a public dataset where possible.
   Record the source, licence and citation.
2. **Choose the score** and write down how it is computed. Prefer a score experts already use, so
   that published expert-to-expert agreement gives you a yardstick. Record those yardsticks with
   their sources.
3. **Split the cases** before looking at results:
   - *examples*: a handful shown to the agent as references (in the critic's own views);
   - *development*: used to build and improve the system;
   - *held-out*: never opened until phase D; fix their order now, with a seed, so that you can use
     them a few at a time.
4. **Write a scorer** that reads the agent's final program, executes it with the same executor, and
   compares with the answer. The scorer and the answers live where agents cannot reach them.
5. **Write reference points**: the score of doing nothing (the starting object untouched), of the
   simplest automatic method, and, if possible, of a perfect program fitted to the answer with the
   language (this tells you whether the language can express the answer at all).

Deliverable: `benchmark/` with the case lists, the scorer, the reference points, and a short
`BENCHMARK.md` saying all of the above.

### Phase B. Build a baseline EPIC system

Build the simplest version of each part that works end to end. Breadth before polish.

**B1. The object.** Define one data structure for it and one starting point (for example a single
reference case). Every result is a transformation of the starting point.

**B2. The language.**
- List the moves an expert names when they describe their work. Start with five to ten.
- Give each move a name, parameters with units, and a one-line meaning.
- Prefer moves that are local and relative ("push this stretch 3 mm out") to absolute coordinates,
  so that an earlier edit does not invalidate a later one.
- Fix how moves combine (a sequence applied in order is usually enough).
- Require a comment on each statement saying what was seen. It is how a person (and the agent itself)
  reads the program later.
- Write `LANGUAGE.md`: the grammar, every move, a complete short example.

**B3. The executor.**
- Parse the program, apply the moves, and produce the result.
- Enforce every invariant constraint. Prefer constructions that cannot violate a constraint over
  checks that catch violations afterwards.
- On a refused move, return the line, the reason and a suggestion ("would cross X: move X first or
  ask for less"). Stop at the first refused line; the result is the state before it.
- Make execution deterministic and fast enough to run tens of times per case.

**B4. The critic.**
- Render the result the way an expert looks at it, with coordinates or scales so that a misfit can
  be read off as a number to type into the next move.
- Show the same views for the reference examples, with their expert answers, so the agent learns
  the conventions from pictures, not only from words.
- Print a short report: what each statement did, key measurements, and the same measurements for
  the examples. Flag habits (things the experts always do) when the result departs from them; do
  not flag sizes (things that genuinely vary between cases).
- Add tools the agent can call to look closer (zoom, a profile, a probe), and say in the guide which
  of them are advice only.
- Write the judging instruction: what to check, in what order, how small a misfit is noise, and the
  stopping rule.

**B5. The agent guide and prompt.**
- One guide file the agent reads first: the rules of the task (what it may read and do), what the
  evidence shows, the experts' conventions, the language, the views, the procedure, the stopping
  rule, and what to hand in (the program and a notes file with confidence, what was visible, the
  error it expects, and remaining doubts).
- One prompt template per case: where the work directory is, what the case is, the budget, the
  hand-in.

**B6. The harness.**
- Prepare one blind work directory per case and agent, under an opaque id.
- Run one agent per case. Record every tool call.
- Audit afterwards with an allow-list: every call must be recognisably one the agent was allowed to
  make. Read anything flagged.
- Score with the phase A scorer.

**B7. Baseline measurement.**
- Run the baseline on the development cases. Score it.
- Run it a **second time** on the same cases and measure how much the scores change. This is the
  noise floor. Keep it; phase C needs it.
- Freeze the baseline: record a hash of every file an agent can read or run.

Deliverables: the language, executor, critic, guide, harness; `results/baseline.md` with scores,
reference points and the noise floor.

### Phase C. Improve it with autoresearch

Run an autoresearch loop in which the **mutable surface is the EPIC system itself**: the language,
the executor and the critic (including the guide's judging instructions). The benchmark, the scorer
and the answers are read-only.

**C1. Set up the loop.** Write `program.md` for the improving agent. It MUST state:
- the goal (for example: raise the mean development score);
- what may be edited: `language/`, `executor/`, `critic/`, `guide/` (name the actual paths);
- what must not be edited: `benchmark/`, the scorer, the case lists, the audit, the invariants of
  section 3;
- the budget per experiment (cases, agents, tokens or money, wall time);
- the decision rule for keeping a change (below);
- the log format.

**C2. One experiment = one hypothesis.**
1. Read the last results and the failures: which cases, which part of the object, what the segmenting
   agents wrote in their notes and tool feedback.
2. Name the failure and the part of the system it points to:
   - **language**: the agent sees the problem but cannot express the fix compactly, or needs many
     statements for one expert move (add or reshape a move);
   - **executor**: valid intentions are refused, results are invalid or crumpled, or execution is
     slow (fix the construction, the guards or the error messages);
   - **critic**: the agent does not see the error, misreads a view, or stops too early or too late
     (add or change a view, a measurement, a reference, or the judging instruction).
3. Write the hypothesis and the decision rule **before** running: what will change, on which cases,
   and what result counts as better.
4. Make one change to one part. Run the test suite of the executor.
5. Run the changed system on the development cases. Score it.
6. Keep the change only if it beats the unchanged system by more than the noise floor (for example
   more than twice the standard error of the paired difference). Otherwise revert it.
7. Append one line to `results.tsv`: id, date, part changed, hypothesis, score before, score after,
   noise floor, kept or reverted, cost.

**C3. Cheap probes before expensive runs.**
- To test the **language**: fit a program to the expert answer by optimisation. If even a fitted
  program cannot reach the answer, the language is the limit; if it can, the limit is elsewhere.
- To test the **critic**: plant known errors in an expert answer and ask agents to find them. If
  agents cannot see a 4 mm error, no instruction about 2 mm errors will help.
- To test the **executor**: run every refused or invalid case from the logs through the new version.

**C4. Rules of the loop.**
- Never edit the benchmark, the scorer or the answers to raise the score.
- Never show the agent an answer, a score or another agent's result during a task.
- Keep one change per experiment, so that a result can be attributed.
- Repeat runs, or use more cases, when the expected gain is close to the noise floor. One agent per
  case per arm is rarely enough to see a gain smaller than the run-to-run variation.
- Stop the loop when several experiments in a row are reverted, or when the budget is spent.
  Report what was tried and failed as carefully as what worked.

### Phase D. Test on held-out cases, once

1. Write the protocol before opening any held-out case: which cases, how many agents, which numbers
   decide, and the thresholds (use the published expert yardsticks).
2. Freeze the system; check the freeze at scoring time.
3. Run once. An agent that dies for an infrastructure reason is reset and rerun; an agent that hands
   in is never rerun.
4. Audit every tool call. Score. Report against the protocol, including what was not met.
5. Later changes to the system are amendments with their own written plan, tested on new held-out
   cases, never on cases already used.

---

## 5. When you are the agent inside an EPIC system

If your prompt puts you inside an EPIC task (you have a work directory, a guide, a language and
tools), then:

1. Read the guide completely before anything else. Study the reference examples in the critic's views.
2. Observe the evidence before writing anything.
3. Write the smallest program that captures the large structure first; refine later.
4. After every execution, read every view the guide names. Check that each edit did what you meant,
   in direction and size.
5. Edit only what you can justify from what you see. Comment every statement with what you saw.
6. If an edit made things worse, delete it; do not pile a correction on a mistake. If you find
   yourself reversing an earlier edit, stop editing there: the evidence is not clear enough.
7. Stop when a full pass over the views shows nothing you can correct with confidence, or when the
   budget is spent. Hand in the program and the notes the guide asks for, including the error you
   expect and what you doubt.
8. Never look for the answer, scores, other agents' work or the harness's code.

## 6. Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Moves at the level of pixels or raw parameters | The agent cannot think in them; programs become long and fragile | Moves an expert would name |
| An executor that silently repairs invalid programs | The agent never learns what it did wrong; results drift | Refuse with a reason |
| A critic that prints a verdict ("score 0.87") | The agent optimises the number, not the work; answers leak | Show views and measurements, never the answer |
| Claiming a gain from one run | Two runs of the same system differ; most "gains" are noise | Measure the noise floor and beat it |
| Tuning on the held-out cases | The final number stops meaning anything | Development cases only; held-out once |
| Several changes in one experiment | A gain or loss cannot be attributed | One change per experiment |
| Restating the experts' rules in your own words | Restatements drift from the source | Quote the written rules, with a pointer to where they are stated |

## 7. What to hand in when you build one

```
benchmark/        cases, splits, scorer, reference points, BENCHMARK.md
language/         LANGUAGE.md and the parser
executor/         the executor, its guards, its tests
critic/           renderings, report, tools, reference examples
guide/            the agent guide and the prompt template
harness/          prepare, run, audit, score, freeze
program.md        the autoresearch program
results.tsv       one line per autoresearch experiment
results/          baseline, development rounds, held-out report
```
