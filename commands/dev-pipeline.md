---
description: Run a feature/bugfix through requirements -> design -> implementation -> test -> review -> PR, delegating each phase to a dedicated subagent.
argument-hint: <task description>
---

Task: $ARGUMENTS

Run this task through the full dev pipeline, using the Agent tool to delegate each phase to its dedicated subagent. Each subagent starts with no memory of this conversation, so every call must be self-contained: include the task, the project root, and relevant file paths. Do not perform the phases yourself — delegate and synthesize.

Every phase's subagent writes its output document to `<project_root>/docs/` (creating the directory on first use) instead of only returning it as text — this is the durable record of the pipeline run. Tell each subagent the project root explicitly. When delegating to a later phase, point it at the doc path(s) from earlier phases (e.g. "read `docs/requirements.md` and `docs/design.md` first") rather than pasting their full contents into the prompt — the subagent has Read access and the file is authoritative.

Phases, in order:

1. **Requirements** — call subagent_type `requirements-analyst` with the raw task description and the project root. It writes `docs/requirements.md`. If it reports open questions that materially affect design, resolve them with the user via AskUserQuestion before continuing (don't guess on anything that changes scope) — update the file yourself if the resolution changes its content, or note the resolution when briefing the next phase.

2. **Design** — call subagent_type `software-architect`, pointing it at `docs/requirements.md`. It writes `docs/design.md`. Show the user a brief summary (files affected + approach, a few sentences) and confirm before moving on if the change is non-trivial (multiple files, new dependencies, architectural change). For small/obvious changes you may proceed without pausing.

3. **Implementation** — call subagent_type `implementer` once per implementation step from the design (or batch tightly-related small steps together), pointing it at `docs/requirements.md` and the relevant slice of `docs/design.md`, plus exact file paths. Each call appends to `docs/implementation-notes.md`.

4. **Testing** — call subagent_type `test-engineer`, pointing it at `docs/requirements.md` and `docs/implementation-notes.md`. It writes `docs/test-report.md`.
   - If it reports failures: route the failure details back to a new `implementer` call to fix, then re-run `test-engineer`. Repeat up to 3 times; if still failing, stop and report the blocker to the user instead of continuing to loop.

5. **Review** — call subagent_type `code-reviewer`, pointing it at `docs/requirements.md`, `docs/design.md`, and `docs/implementation-notes.md`. It writes `docs/review.md`.
   - If there are correctness-level (not stylistic) findings: route them back to `implementer` to fix, then re-run `test-engineer` and `code-reviewer` again. Repeat up to 2 times; if issues persist, stop and summarize the unresolved findings for the user instead of publishing.

6. **Publish** — before calling `pr-publisher`, explicitly ask the user for confirmation (this pushes to a remote and opens a PR — a visible, hard-to-reverse action). Once confirmed, call subagent_type `pr-publisher`, pointing it at `docs/requirements.md`, `docs/design.md`, and `docs/test-report.md` so it can write an accurate PR body. It also saves `docs/pr-description.md`.

Throughout: give the user a one-line status update between phases (what finished, what's next, which doc was written) rather than dumping each subagent's full raw output. At the end, report the PR URL (or wherever the pipeline stopped and why), and remind the user the full trail is in `docs/`.
