---
name: simplify
description: Review and clean up changed code for reuse, quality, and efficiency. Use when the user invokes "/simplify" or asks to simplify recent code changes, review changed files for cleanup opportunities, remove duplication, improve abstractions, or fix efficiency issues before finalizing code.
---

# Simplify

## Workflow

Use this skill to review current code changes, identify cleanup opportunities, and directly fix worthwhile issues.

### 1. Identify Changes

Inspect the current git changes:

- Run `git diff --stat` and `git diff`.
- If there are staged changes, also run `git diff --cached` or `git diff HEAD` so the review covers staged and unstaged work.
- If there are no git changes, review the most recently modified files the user mentioned or files you edited earlier in the conversation.

Preserve unrelated user changes. Do not revert or reformat code outside the review scope.

### 2. Launch Three Review Agents

Launch three subagents concurrently in one tool message when subagents are available and the user has explicitly invoked this skill. Give each agent the full diff and a clear, bounded task. Tell each agent to report findings only, not edit files.

When using Claude Code's `Agent` tool, launch all three agents in parallel rather than waiting for each one sequentially.

If subagents are unavailable, perform the three reviews yourself using the same checklists.

Use these agent prompts:

```text
You are reviewing a code diff for reuse opportunities. Search the codebase for existing utilities, helpers, adjacent patterns, shared modules, and similar implementations that could replace newly written code. Flag duplicated functions and inline hand-rolled logic such as string manipulation, path handling, environment checks, and ad-hoc type guards. Return concise findings with file/line references and suggested existing APIs. Do not edit files.
```

```text
You are reviewing a code diff for quality issues. Look for redundant state, cached derivable values, observers/effects that should be direct calls, parameter sprawl, copy-paste with slight variation, leaky abstractions, and stringly-typed code where constants/enums/string unions/branded types already exist. Return concise findings with file/line references and suggested fixes. Do not edit files.
```

```text
You are reviewing a code diff for efficiency issues. Look for redundant computations, repeated file reads, duplicate network/API calls, N+1 patterns, missed concurrency, startup/per-request/per-render hot-path bloat, TOCTOU existence checks, unbounded data structures, missing cleanup, event listener leaks, and overly broad operations. Return concise findings with file/line references and suggested fixes. Do not edit files.
```

### 3. Fix Issues

Wait for all review agents to complete before editing.

Aggregate findings, then fix each issue directly when it is valid and worth addressing. If a finding is a false positive or not worth changing, note it briefly and skip it. Do not spend time arguing with skipped findings.

Follow the repository's existing patterns and helpers. Keep edits narrow, avoid unrelated refactors, and update focused tests when the cleanup changes behavior or shared code paths.

### 4. Verify and Report

Run the most relevant available checks for the changed files. If checks cannot be run, state why.

Finish with a brief summary:

- What was fixed, or that the code was already clean.
- Any findings intentionally skipped.
- Verification performed.
