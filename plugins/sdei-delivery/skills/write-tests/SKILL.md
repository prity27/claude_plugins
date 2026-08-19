---
name: write-tests
description: Write backend unit and integration tests and frontend component tests, each named for the acceptance criterion it proves, standing up the test runner first when the project has none. Use after /build-module, or to add coverage to existing code that has approved stories.
argument-hint: "[epic id | path] [backend | frontend] — default: the epic just built"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash
---

# Write tests

Tests that prove the acceptance criteria. Each test is **named for the AC id it proves**, which is
what makes `/validate-delivery` a mechanical check rather than an opinion.

A test suite that exercises code without mapping to criteria tells you the code does what the code
does. Mapping to AC ids tells you whether what was promised was delivered.

## 0. Preconditions and the runner

Read `docs/delivery/PROFILE.md`, the approved epic, and its acceptance criteria.

Then establish whether a runner actually exists — this is step zero, not an assumption:

Take the test command from the profile's **test runner** and **gate commands** rows, then verify
it against the repository. Look for three things in whatever form the stack expresses them: a
declared test command, a runner config file, and test files that actually exist.

```bash
# the declared command — from the profile, e.g.
npm test | pytest | go test ./... | mvn test | dotnet test | bundle exec rspec

# the runner config, by stack family
ls jest.config.* vitest.config.* playwright.config.* 2>/dev/null          # Node
ls pytest.ini tox.ini setup.cfg 2>/dev/null; grep -n "\[tool.pytest" pyproject.toml 2>/dev/null  # Python
ls *_test.go 2>/dev/null; ls src/test 2>/dev/null                         # Go / Java

# do test files exist at all
find . \( -path ./node_modules -o -path ./.venv -o -path ./target \) -prune -o \
  \( -name '*.test.*' -o -name '*.spec.*' -o -name 'test_*.py' -o -name '*_test.go' \) -print | head
```

Three cases:

1. **A working runner and existing tests** — read two of them and match their shape exactly.
2. **A runner declared in the manifest with no config, and the test command failing or matching
   nothing** — common, and it means there is no harness. Standing one up is a decision with real
   consequences (which runner, how the database is handled, whether CI runs it), so **put it to the
   user before writing config**, and propose the smallest thing that runs one real test end to end.
3. **Nothing at all** — same conversation, from a blank slate. Recommend the runner that matches
   the stack the profile records, not the one you like.

Do not scatter tests using a framework the project has not adopted. An orphan test file that nobody
can run is worse than none.

## 1. Backend tests

**Name every test for its AC.** `BE-03-02 AC-2 — closing a campaign with an open reception returns
409 and leaves it active`. The id must be greppable, because the validation matrix greps for it.

Per story, cover in this order:

- **the success path**, asserting the actual response shape and status, not merely that it did not
  throw;
- **validation failures** — each rule the validator declares, with the real error code;
- **authorization** — the unauthenticated caller, the wrong-role caller, and **the right-role
  caller acting on someone else's record**. That last one is the object-level check, it is the most
  common real defect, and it is almost never tested;
- **state and idempotency** — the already-in-that-state case, the double submit;
- **boundaries** — empty result, page limits, maximum lengths, date edges and timezone boundaries,
  zero and negative quantities;
- **failure paths** — what happens when a dependency is unavailable or a multi-step write fails
  halfway. If the code has no answer, the test is what makes that visible.

Assert on **observable behaviour** — the response, the persisted state, the audit entry written —
not on internal calls. A test asserting that a mock was called proves the code called a mock.

Test data is built per test, not shared and mutated across a file. Shared fixtures produce order
dependence, and order dependence produces the flake that gets the suite disabled.

## 2. Frontend tests

Also named for their AC. Test what a user experiences: rendered output, interaction, state after
interaction, and the request that was issued — not implementation detail.

Cover per component or screen: the loading state, the empty state, the error state, the populated
state, form validation mirroring the server's rules, the disabled and submitting states, and the
keyboard path. **Loading, empty and error are states of the feature**, and a story that lists them
as criteria needs each one asserted.

Query by role and label rather than by class or test id where the framework allows it — it tests
what a user and a screen reader can find.

Mock at the network boundary, not at the module boundary, so the test exercises the real client
code that the acceptance criterion actually depends on.

## 3. Run them, and report honestly

Run the suite. Report the real output.

Then map every AC in the epic to its test, and list every AC you did **not** cover with the reason —
untestable as written, needs infrastructure that does not exist, or blocked on a decision. That list
is an input to the validation gate, so never quietly omit it.

A test written to pass rather than to prove is worse than no test. If an AC cannot be proven with a
test, say so rather than writing something adjacent and calling the criterion covered.

## 4. Wire it into the gates

If the project's CI does not run tests, say so and propose the change. A suite nobody runs decays
within weeks.

## Standing rules

- **Never change source code to make a test pass** without saying so explicitly and separately. A
  failing test is information; suppressing it destroys the information.
- Never assert on mocks where you can assert on state.
- Never skip a test to get a green run. A skipped test is reported as not covered.
- Never write tests against an unapproved epic — the criteria can still change.
