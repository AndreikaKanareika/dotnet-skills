---
name: configure-csharp-lsp-solution
license: MIT
disable-model-invocation: true
description: >
  Point the Roslyn C# language server at a solution that is not in the workspace root. Use when
  cross-project navigation is dead in a .NET repo - find-references returns only the declaration,
  workspace-symbol search comes back empty, hover shows nothing for types from other projects, C#
  IntelliSense sees only the current file - and the *.sln/*.slnx lives under src/ rather than at the
  root. Writes dotnet.defaultSolution into .vscode/settings.json. Not for repos whose solution is
  already at the workspace root, non-.NET repos, or build, restore, SDK, or targeting-pack failures.
---

# configure-csharp-lsp-solution

## Purpose

Restore cross-project C# intelligence in a repository whose solution does not sit at the workspace
root. The Roslyn language server searches for `*.sln`/`*.slnx` **only in the workspace folder
itself**, while project discovery recurses. In a repo with `src/App.sln`, the server finds every
`.csproj`, loads none of them, reports no error, and serves every file in miscellaneous-files mode.
Writing `dotnet.defaultSolution` names the solution explicitly and restores full navigation.
Despite the name, this is not VS Code-specific: the language server reads `.vscode/settings.json`
itself and logs `Using VS Code settings to auto load solution <path>`, so the fix applies to any
runtime that hosts the server.

## When not to use

- A `*.sln` or `*.slnx` already sits in the workspace root — the server finds it; the problem is elsewhere.
- The repository contains no `.csproj` — not a .NET workspace.
- The repository has `.csproj` files but no `*.sln`/`*.slnx` anywhere. This skill names an
  existing solution; it does not create one. Say so and stop.
- The symptom is a build, restore, SDK, or targeting-pack failure. Those produce real errors; this
  bug produces none.

## Inputs

| Input | Required | Notes |
|---|---|---|
| Workspace root | yes | The directory the session was launched from. Discover it; do not ask the user. **Not** necessarily the git root. |
| Candidate solutions | derived | Discovered by recursive search |

## Workflow

### Step 1 — Confirm this is a .NET workspace

Search recursively for `*.csproj`. If none, stop and say the repository is not a .NET workspace.

### Step 2 — Rule out the non-bug

List `*.sln` and `*.slnx` **in the workspace root only, non-recursively**. If one exists, stop:
the server can already find it, so this skill does not apply. Say so plainly rather than writing
a redundant setting.

### Step 3 — Check for an existing setting

If `.vscode/settings.json` exists and defines `dotnet.defaultSolution`, resolve the path.

- Path resolves → the setting is being honoured. Stop — unless the named solution plainly does not
  contain the projects being navigated between, in which case this is a candidate-selection problem:
  continue from Step 4.
- Path does not resolve → **this is still the bug.** The server logs
  `Default solution path {SolutionPath} does not exist, skipping.` and falls back silently.
  Repair it: run Step 4's search. If it returns exactly one candidate, write it without asking — the
  existing setting already establishes the intent. If it returns two or more, present and confirm as
  Step 4 requires. Then continue from Step 5.

### Step 4 — Find and rank candidates

Search recursively for `*.sln`/`*.slnx`, ignoring `bin/`, `obj/`, and `.git/`.

If the search returns **no** candidates, stop. There is no solution to name, so this skill does
not apply — report that the repository has projects but no solution file, and do not write a
setting.

Rank by:

1. Number of projects referenced by the solution (more is better)
2. Coverage of the repository's `.cs` files
3. Shallowest path (fewer directory levels from the root)

If exactly **one** candidate is found, write it (Step 5) and report the path you wrote. There is
nothing to disambiguate.

If **two or more** are found, present the ranked list with project counts, recommend the top entry,
and ask the user to confirm before writing. Do not silently pick one — in a monorepo the user knows
which solution matters.

### Step 5 — Write the setting

Write a path **relative to the workspace root**, forward slashes:

```json
{
  "dotnet.defaultSolution": "src/App.sln"
}
```

If `.vscode/settings.json` already exists, **merge** — read it, add or replace only the
`dotnet.defaultSolution` key, preserve every other key and the file's formatting. Never overwrite.

### Step 6 — Tell the user to restart

The server reads this setting exactly once, while handling the `initialized` notification. A running
server will not pick it up. State plainly that the session must be restarted before anything changes.

### Step 7 — Validate after restart

The restart ends this session, so hand the check to the user rather than claiming to run it:

> After restarting, run find-references on a type declared in one project and used in another.
> Many hits → fixed. Exactly one hit (just the declaration) → not fixed.

If you are the agent in the new session and the check returns one hit, say so plainly. Do not
report success you have not observed.

## Validation

| Check | Pass |
|---|---|
| `.vscode/settings.json` parses as JSON | yes |
| `dotnet.defaultSolution` resolves to an existing file | yes |
| Pre-existing keys still present | yes |
| Cross-project find-references after restart | more than one hit (user-verified after restart) |

## Common pitfalls

- **Claiming success before a restart.** The setting has no effect on the running server. This is
  the single most likely way to report a fix that has not happened.
- **Clobbering `.vscode/settings.json`.** It commonly holds unrelated editor settings. Merge.
- **Using the git root instead of the workspace root.** The server reads the folder the session was
  launched from. If they differ, the file must go in the workspace root.
- **Trusting an existing `dotnet.defaultSolution`.** A stale path is skipped silently, which looks
  exactly like the setting working.
- **Reaching for a `.slnf` solution filter.** Unsupported — the server globs `*.sln`/`*.slnx` only.
