# Open Source Contribution Log

Tracking issues tackled, implementation plans, and pull requests across multiple open-source projects — CodePath AI301.

## Contribution Index

| Project | Issue | Status | Link |
|---|---|---|---|
| Frank!Framework | [#4739 — Larva wait-for-condition mechanism](https://github.com/frankframework/frankframework/issues/4739) | Assigned / In Progress | [View Issue](https://github.com/frankframework/frankframework/issues/4739) |


---

## Contribution 1: Frank!Framework

**Repository:** [frankframework/frankframework](https://github.com/frankframework/frankframework)

**Issue:** [#4739 — In Larva tests, there should be a mechanism to wait for a condition to become true](https://github.com/frankframework/frankframework/issues/4739)

### Problem Summary

Frank!Framework's Larva integration-test tool executes scenario steps (read/write against senders, listeners, databases, filesystems, etc.) as single, synchronous actions and compares the result against an expected file. There is currently no way to express "wait until this condition becomes true, polling with a timeout" — for example, waiting for an asynchronously-written row to appear in an error/message store after an adapter times out.

Today, the only workaround is a fixed-duration wait (`sleep`) inserted before the read step. This is either:
- **too short**, causing intermittent, flaky test failures when the async write hasn't completed yet, or
- **too long**, unnecessarily slowing down the whole test suite.

The issue requests a proper polling/wait-for-condition mechanism as a first-class Larva feature.

### My Understanding of the Fix

The most promising approach is to extend Larva's existing per-step action model (`org.frankframework.larva.actions`, and the step-execution logic in `ScenarioRunner`) with optional `waitfor.timeout` / `waitfor.interval` properties on a step. When set, the step retries its read-and-compare cycle on the given interval until either the expected result is produced or the timeout elapses — falling back to today's single-shot behavior when the properties are absent, so no existing scenario is affected.

### Status

- [x] Repository forked on GitHub
- [x] Feature branch created: `feature/4739-larva-waitfor-condition`
- [x] Comment posted on the issue expressing interest in being assigned
- [ ] Implementation (Phase 2)

---

*Part of the CodePath AI301 open-source contribution assignment.*
