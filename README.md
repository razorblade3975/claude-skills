# Claude Code Skills

Custom skills for [Claude Code](https://claude.com/claude-code) that extend its capabilities with structured workflows for code quality and git hygiene.

## Skills

### finalize-code

Three-phase finalization workflow for wrapping up implementation sessions:

1. **Simplify** — Runs code simplification on files changed during the session
2. **Review** — Sends all changes through an external code review (via `codex-reviewer`)
3. **Commit** — Groups changes into well-structured conventional commits (`feat`, `fix`, `refactor`, etc.)

### consolidate-commits

Cleans up branch commit history by analyzing and grouping related commits ahead of the remote base branch:

- Squashes sub-features, tests with their features, and plan docs with their implementations
- Keeps unrelated changes (refactors vs features, independent bugfixes) as separate commits
- Uses non-interactive `git rebase` with `GIT_SEQUENCE_EDITOR` — no manual editor interaction
- Creates backup branches and verifies end-state integrity after every rebase

### codex-reviewer

Gets an independent code review from OpenAI Codex CLI, then iterates until consensus:

- Sends session diffs with full context (goals, plan, architectural decisions)
- Evaluates feedback with technical rigor — accepts valid issues, pushes back on incorrect ones
- Iterates up to 3 rounds, then reports a structured summary of resolved issues, rejections, and deferrals

## Installation

Copy the skill directories into your Claude Code skills folder:

```bash
cp -r finalize-code consolidate-commits codex-reviewer ~/.claude/skills/
```

## Usage

Invoke any skill from Claude Code by name:

- `/finalize-code` — after finishing implementation work
- `/consolidate-commits` — before creating a PR or merging
- `/codex-reviewer` — when you want a second opinion on your changes
