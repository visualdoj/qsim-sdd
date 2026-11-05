# qsim workspace
 
Repositories are checked out side by side (paths are relative to this file):
 
| Path | Contents | Access |
|---|---|---|
| ../qsim | source code, tests, documentation | code changes go here |
| ../qsim-sdd | specs, plans, tasks, skills | SDD artifacts go here |
| ../agent-intructions | shared agent instructions | READ-ONLY: do not edit, commit, or push |
 
@../agent-intructions/AGENTS.md
 
## Rules
- Do not create agent-related files in qsim (CLAUDE.md, .claude/, specs/), and do not reference specs or task IDs in code, comments, or commit messages.
- Every feature starts with a spec in qsim-sdd/specs/NNN-name/; code in qsim comes after.
- Use the same branch name NNN-name in both repositories; the spec and the code go in separate PRs.

## qsim-sdd layout
- constitution.md — project principles: stack, style, constraints
- context/qsim.md — how to build qsim and run its tests
- specs/NNN-name/{spec,plan,tasks}.md
- .claude/skills/sdd-*/ — SDD workflow commands (the sdd- prefix avoids name collisions)
