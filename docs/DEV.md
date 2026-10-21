# Development Workflow

This document describes how work is planned in Jira and how Git branches are structured in this repository.

> **Context:** This is currently a single-developer project. Strictly speaking, most of this process is unnecessary for one person — a single `main` branch and a to-do list would work. It is followed deliberately as team best practice: it keeps work organized, produces small reviewable units of change, and avoids the long-running-branch problems that appear as soon as more than one person is contributing. A Kanban board is used to keep the work visible and organized.

## 1. Jira hierarchy: Epic > Story > Subtask

Planned work is tracked in three levels:

```text
Epic
└── Story
    └── Subtask

Bug   (standalone — sits outside the hierarchy above, at the same level as a Story)
```

| Level       | What it is                                                                                                                                                                      | Example in this project                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Epic**    | A large outcome or milestone, made up of many stories. Too big to complete (or merge) as one unit of work.                                                                      | "Boot a minimal kernel in QEMU", "Implement memory management"                    |
| **Story**   | A complete, independently completable unit of work. Small enough to finish, test, and merge on its own.                                                                         | "Integrate Limine and boot to a C entry point", "Add a panic handler"             |
| **Subtask** | One technical step of its parent story. **Not** a complete unit of work on its own.                                                                                             | "Add linker script", "Write Limine configuration", "Add Makefile ISO target"      |
| **Bug**     | A defect found on `main`. Operates at the same level as a Story — a complete, independently fixable unit of work — but is **not** part of the Epic > Story > Subtask hierarchy. | "Kernel triple-faults on boot in QEMU", "Panic handler prints garbage characters" |

Rules of thumb:

- If a piece of work needs several distinct, separately testable steps, it is probably a **Story** (with Subtasks) or an **Epic** (with Stories).
- A Subtask must always have a parent Story. If something has no parent story, it is not a subtask — promote it to a Story.
- A Story should be completable and verifiable by itself. If finishing it leaves the project in a broken or meaningless state, it is probably a Subtask of a larger Story.
- Prefer breaking an Epic down into Stories up front, even if the details of later Stories stay rough until earlier ones are done.

### Bugs

Bugs are semi-independent from the hierarchy above. They exist to document errors discovered on `main` — that is, in code that has already been merged — rather than to plan new work.

- A Bug has **no parent Epic or Story** and does not need one to be started or completed. It may be _linked_ to the Epic/Story that introduced or relates to the defect for traceability, but linking is not nesting: the Bug is not a child of that issue.
- Because a Bug is at Story level, it is prioritized alongside Stories on the Kanban board (§2) and is branched from and merged back to `main` directly (§3–§4), like a Story.
- If a defect is found in work that is still on an unmerged Story/Subtask branch, fix it on that branch under the existing issue — do not raise a Bug. A Bug is only for errors that have reached `main`.
- A Bug report should record: what happened vs. what was expected, steps to reproduce, the environment (e.g. QEMU/boot configuration), and any relevant logs or debugger output. For kernel work, the exact failure mode matters — e.g. triple fault, hang, or garbled output are very different bugs.

## 2. Kanban board

Work is tracked on a Kanban board, moving left to right:

```text
Backlog  →  To Do / Ready  →  In Progress  →  In Review / Testing  →  Done
```

- **Backlog:** Ideas and roughly-scoped Epics/Stories, plus Bugs awaiting triage. Not yet ready to start.
- **To Do / Ready:** Stories, Bugs, or Subtasks of the current Story that are scoped well enough to pick up. Bugs sit alongside Stories here and are prioritized against them — a severe Bug (e.g. `main` no longer boots) may jump ahead of planned Story work.
- **In Progress:** Actively being worked on. Keep this column small — ideally **one Story or Bug at a time** (WIP limit: 1 Story/Bug, its current Subtask if any). Starting is not progress; finishing is.
- **In Review / Testing:** Code is complete on its branch and is being built/tested (e.g. booted in QEMU) before merging.
- **Done:** Merged to its target branch (see §4) and the Definition of Done (§5) is met. Merely "code written" is not Done.

Move the Jira issue when the work actually changes state — when you branch, when you open it for review/testing, and when you merge — not in a batch at the end of the day.

## 3. Branch naming

Every branch is tied to a Jira issue and named after its level in the hierarchy:

```text
epic/<id>/<short-description>
story/<id>/<short-description>
subtask/<id>/<short-description>
bug/<id>/<short-description>
```

- `<id>` is just the numeric part of the Jira issue key, without the project prefix — e.g. for Jira issue `OS-12`, `<id>` is `12`.
- `<short-description>` is a few lowercase, hyphen-separated words summarizing the issue.

Examples:

```text
epic/1/boot-minimal-kernel
story/8/integrate-limine-bootloader
subtask/9/add-linker-script
subtask/10/write-limine-config
bug/15/fix-qemu-triple-fault
```

Branch from the right base:

| Branch type           | Branched from                                                                                   |
| --------------------- | ----------------------------------------------------------------------------------------------- |
| `epic/<id>/<desc>`    | `main`                                                                                          |
| `story/<id>/<desc>`   | `main` (preferred — see §4), or its parent `epic/<id>/<desc>` branch if one exists              |
| `subtask/<id>/<desc>` | Its parent `story/<id>/<desc>` branch — **never** `main`                                        |
| `bug/<id>/<desc>`     | `main` — a Bug documents an error on `main`, so its fix is branched from and returned to `main` |

One branch per Jira issue. Do not reuse a merged branch for the next issue; cut a fresh one.

As a rule, a branch is merged back into the branch it was cut from: a Subtask into its Story, a Story into `main` or into its parent Epic (matching where it was branched from), a Bug into `main`, and an Epic into `main`. Merge into one target only — not both.

## 4. Merging rules

```text
subtask/*/*  ──►  story/*/*  ──┬──►  main
                               └──►  epic/*/*  ──►  main   (allowed, but not preferred — see below)
bug/*/*      ─────────────────────►  main
```

- **Subtask branches are only merged into their parent Story branch.** Never merge a `subtask/*/*` branch directly into `main` (or into an Epic branch). A subtask is not a complete unit of work, so landing it on `main` alone would leave `main` in a partial state.
- **Story branches can be merged into `main` or into their parent Epic branch.** Merging to `main` is the normal, preferred path: a Story is complete and testable on its own, so it is safe to land independently. If the Story was branched from an `epic/*/*` branch that is in use, it may instead be merged back into that Epic branch, and it will reach `main` when the Epic branch is merged. Choose one target, matching where the Story was branched from.
- **Bug branches are merged into `main`.** Like a Story, a Bug fix is a complete unit of work, and since the Bug exists on `main`, the fix returns directly to `main` — it is not merged into any Epic/Story branch, even if it is linked to one.
- **Epic branches can be merged into `main`, but this is discouraged.** See below.

### Prefer completing Stories over tackling an Epic directly

Although merging an `epic/*/*` branch to `main` is allowed, the recommended approach is:

1. Break the Epic into Stories in Jira.
2. Branch and complete each Story individually (`story/*/*` off `main`).
3. Merge each finished Story to `main` as it is completed.

Merging a Story into its parent Epic branch instead is permitted — that is how an `epic/*/*` branch is assembled when one is in use — but note the trade-off: the Story does not reach `main` until the whole Epic merges, so it inherits the divergence risk described below. Only use the Epic branch as the merge target when there is a reason the Stories cannot land on `main` independently.

Working directly on a long-lived `epic/*/*` branch until the whole epic is done leads to a branch that:

- **Diverges from `main`** as other improvements land there, making the eventual merge large, conflict-prone, and hard to review.
- **Delays integration** — problems between the epic's work and the rest of the project are discovered all at once at merge time instead of incrementally.
- **Blocks partial progress** — nothing is usable or demonstrable on `main` until everything is finished.

Short-lived Story branches merged frequently keep diffs small, conflicts trivial, and `main` always in a working state. Reserve an `epic/*/*` branch for cases where stories genuinely cannot be landed independently — and if one is used, regularly bring `main` into it (and merge completed Stories into it) to limit divergence, then merge it to `main` as soon as the epic is complete.

> For a single developer this distinction barely matters — there is nobody else landing changes on `main` to diverge from. The Story-first habit is kept anyway so the workflow scales unchanged if/when collaborators join.

## 5. Definition of Done

A Story or Bug (or Subtask, relative to its Story branch) is Done when:

- [ ] The code builds cleanly with the project's toolchain.
- [ ] It has been tested as far as practical — for kernel work, that normally means booting it in QEMU.
- [ ] For a Bug: the reported error no longer reproduces with the fix in place, using the reproduction steps recorded on the Bug.
- [ ] New/changed C code is formatted (`clang-format`, per `.clang-format`).
- [ ] Documentation is updated if behavior, setup, or workflow changed.
- [ ] The branch is merged into its correct target (Subtask → Story, Story → `main` or its parent Epic, Bug → `main`) and the remote/local branch is deleted.
- [ ] The Jira issue is moved to Done, with the Jira key referenced in the branch name, commits, and pull request (e.g. `OS-8: Integrate Limine bootloader`) so the work stays traceable.

## 6. End-to-end example

For the Epic _"Boot a minimal kernel in QEMU"_ (`OS-1`):

1. **Plan:** In Jira, create the Epic `OS-1`, with Stories such as `OS-8` "Integrate Limine and boot to a C entry point". Break `OS-8` into Subtasks `OS-9` "Add linker script" and `OS-10` "Write Limine configuration". Move `OS-8` to To Do on the Kanban board.
2. **Branch the Story:** `git checkout main && git checkout -b story/8/integrate-limine-bootloader`. Move `OS-8` to In Progress.
3. **Work the Subtasks:** Branch `subtask/9/add-linker-script` from the Story branch, implement, test, and merge it **back into `story/8/integrate-limine-bootloader`**. Repeat for `OS-10`.
4. **Review/test:** With all Subtasks merged into the Story branch, build and boot the result in QEMU. Move `OS-8` to In Review / Testing.
5. **Merge to `main`:** Merge `story/8/integrate-limine-bootloader` into `main`, delete the Story branch, and move `OS-8` to Done.
6. **Repeat** with the next Story under `OS-1` until the Epic itself can be closed.

Note that `OS-9` and `OS-10` never touched `main` directly — only the complete Story `OS-8` did.

If an `epic/1/boot-minimal-kernel` branch were in use instead, `OS-8` would have been branched from it and merged back into it at step 5, reaching `main` only when the Epic branch is merged at the end.

### Bug example

If the merged kernel later triple-faults on boot, that is raised as Bug `OS-15` "Kernel triple-faults on boot in QEMU" — standalone, with no parent Epic or Story (optionally linked to `OS-8` for traceability). It is triaged on the Kanban board alongside Stories, fixed on `bug/15/fix-qemu-triple-fault` branched from `main`, verified against the recorded reproduction steps, and merged straight back to `main`.
