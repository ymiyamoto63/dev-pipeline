---
name: dev-pipeline
description: Run a feature/bugfix through requirements -> design -> implementation -> test -> review -> PR, delegating each phase to a dedicated subagent.
argument-hint: <task description>
agent: dev-pipeline
---

Run the task described with this command through the full dev pipeline, following your agent instructions (requirements → design → implementation → test → review → PR). The text the user typed after the command is the task description.
