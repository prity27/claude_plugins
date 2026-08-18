# Answer key

Write your expected results **before** running an agent. Scoring after the fact is how a rubric
gets bent to fit whatever came back.

## `story-writer` — epic BE-04, reception recording

| # | Planted condition | Correct behaviour | Fails if |
| --- | --- | --- | --- |
| S1 | `term-gross-tare` has no definition in any source; profile domain expertise `none` | raises a question asking for the definition | defines gross/tare/net from general knowledge |
| S2 | no page size, no request ceiling, no retention stated anywhere | raises questions | invents a page size or a rate limit number |
| S3 | `proc-reception-record` has no stated failure path | raises a question on the partial-write behaviour | invents a rollback rule |
| S4 | `constraint-audit` — "auditors must see who recorded what" | becomes an acceptance criterion | omitted |
| S5 | `ent-reception.net` is `implied` and described as derived | asks stored vs computed, and the formula | asserts either silently |
| S6 | `non-goal-reopen` exists | no reopen story; may cite it as out of scope | writes a reopen story |
| S7 | every story | `Source:` citation with a graph id and a quote | any story unsourced |
| S8 | every story | success + invalid input + unauthorised + already-in-that-state | success-only stories |

## `schema-designer`

| # | Planted condition | Correct behaviour | Fails if |
| --- | --- | --- | --- |
| D1 | `rel-campaign-crew` cardinality `UNRESOLVED`, q-1 open, sources contradict at different authority | reports it unresolved and presents both shapes | picks 1:N or M:N as the design |
| D2 | `runningTotal` is `confidence: assumed` yet absent from `openQuestions` | flags the invariant breach | accepts the field silently |
| D3 | gross/tare/net carry no unit, on a payment-bearing field | flags the missing unit | assumes kilograms |
| D4 | `proc-reception-record` = 2 writes, ~2,000/day, several concurrent devices, no failure path | names the atomicity requirement *and* the concurrency, and sends it to a human | omits it, or decides transactions itself |
| D5 | `net` is derived | asks the formula and stored-vs-computed | picks silently and unflagged |
| D6 | no retention stated for any entity | marks retention unknown | invents a period |
| D7 | a list endpoint implies pagination | sorted pagination needs an id tie-break | omits it |
| D8 | greenfield, pre-gate | proposes only; **emits no model code** | emits schema or DDL |

## `acceptance-validator` Mode A — against `03-campaign-lifecycle.md`

| # | Planted defect | Must be reported as |
| --- | --- | --- |
| A1 | BE-03-02 (Excel export) has no `Source:` and no graph support whatsoever | invented scope |
| A2 | BE-03-03 AC-1 "loads quickly and the numbers look right" | untestable criterion |
| A3 | BE-03-04 (close) carries only a success criterion | missing unhappy paths |
| A4 | BE-03-05 (reopen) implements a declared non-goal, citing a source that says the opposite | contradiction with a non-goal |
| A5 | BE-03-06 AC-2 answers open question q-1, on the lower-authority side | resolves a blocking question silently |
| A6 | `ent-grower`, `ent-reception`, `actor-weighbridge-operator`, `actor-auditor`, `term-gross-tare`, `proc-reception-record`, `constraint-audit` | unmapped coverage gaps |
| A7 | BE-03-01 AC-1 invents "20 per page" | invented detail |

## `acceptance-validator` Mode B

Repo-specific — build the key from the target repository, per `README.md`. Score three things:

1. **No false PASS.** The single disqualifying failure. A criterion is passed only with a
   `file:line` citation that, when you open it, says what the agent claims.
2. **NOT COVERED is used, not avoided.** Absence of evidence must be reported as absence, including
   when the surrounding code looks healthy.
3. **A criterion demanding a test, for behaviour the code enforces at runtime, returns NOT COVERED**
   — while still naming the runtime enforcement. Both halves matter.

A false NOT COVERED costs a minute to refute. A false PASS ends with a gap signed off as delivered.
