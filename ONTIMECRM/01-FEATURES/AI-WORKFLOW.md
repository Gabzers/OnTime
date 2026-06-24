# AI Workflow — Test-First Loop

Hub: [[OnTimeCRM]] · Tasks: [[AI-TASK-PLAN]] · Setup: [[LOCAL-AI-SETUP]].

How the local AI executes **every** task. Test-first: describe the ideal behavior as a failing
test, then make it pass, then verify everything. A task is only "done" after the gate passes
AND the docs are updated.

## The loop (per task)

1. **RED — write the ideal test first.**
   - Backend: add/extend a flow test in `OnTimeCRM.Tests/Flows` (see [[TESTING]]).
   - Frontend: add a Vitest + React Testing Library test next to the component/api.
   - Run it. **Confirm it fails for the expected reason** (not a typo/compile error).

2. **GREEN — implement the minimum** to make that test pass. Follow [[CONVENTIONS]] and the
   data-layer rule ([[2026-05-30-data-layer]]): complex logic → stored function, simple → EF/service.

3. **VERIFY EVERYTHING — the gate below.** All of it, not just the new test.

4. **SYNC DOCS — update the vault** (table in [[HOW-TO-USE]]): tick [[AI-TASK-PLAN]] / [[KNOWN-ISSUES]],
   update [[DOMAIN]]/[[SCHEMA-AUTH]]/[[API-REFERENCE]] if anything changed, and write a session log
   in `03-SESSIONS/` (see [[SESSIONS]]).

5. **Commit** (optional) with a `feat:`/`fix:`/`test:` message naming the task.

## Who runs the gate

The **local model writes** the test + code (RED → GREEN). The **gate is executed by you,
Claude Code, or Continue in agent mode** — see [[LOCAL-AI-SETUP]]. The model is NOT done
when it says "the test should pass"; it's done when commands have actually been run and
all are green. Paste failures back into the chat to iterate.

## The verification gate ("verify tudo")

Run after **every** task. Nothing is done until all of this is green:

**Backend** (Docker running)
```bash
cd OnTimeCRM_Backend
dotnet build                 # 0 errors
dotnet test                  # ALL flow tests green — not only the new one
```
**Frontend**
```bash
cd OnTimeCRM_Frontend
npm run test                 # Vitest — all green
npm run build                # tsc typecheck + vite build, 0 errors
```
**Docs**
- [ ] matching vault note updated; [[AI-TASK-PLAN]] / [[KNOWN-ISSUES]] ticked
- [ ] session log written in `03-SESSIONS/`

> If a previously-passing test goes red, the task is NOT done — fix the regression before moving on.
> Never weaken a test to make it pass; change the test only when the *intended behavior* changed
> (and note why in the session log).

## Rules for the local model
- One task at a time. Don't start the next until the gate is green.
- Surgical context: `@file` the relevant note(s) + the test file, not the whole repo. See [[LOCAL-AI-SETUP]].
- If the task is bigger than one red→green cycle, split it into sub-tests and do them in order.

## Related
[[AI-TASK-PLAN]] · [[TESTING]] · [[CONVENTIONS]] · [[HOW-TO-USE]] · [[OnTimeCRM]]
