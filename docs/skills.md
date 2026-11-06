# snp skills

`snp` ships skills that take an idea from a design doc to issues an agent can build. In Claude Code, you run each one as a slash command, such as `/snp-think`. A skill reads your org's memory from the local copy `snp pull` keeps, and writes through `snp`.

## The skills

| Skill | What it does |
|---|---|
| `snp-think` | Writes a design doc for an idea, before any code. |
| `snp-plan` | Turns a design doc into a roadmap of milestones you can release one at a time. |
| `snp-spec` | Writes an executable spec for one roadmap milestone. `--fix` applies a review's accepted blockers to that spec, and `--reconcile-plan` moves the milestone from scoped to planned. |
| `snp-review` | Runs a panel of expert agents over a spec, and records its findings and decisions. |
| `snp-board` | Turns a planned milestone's spec into `snp` issues in one commit, after you approve the list. |
| `snp-build` | Builds one `snp` issue end to end: it claims the issue, plans and builds it with Superpowers on its own branch, opens a pull request, and marks the issue done once a person has merged it. It never merges. |
| `snp-milestone` | Runs a roadmap milestone from its spec to a built board. It starts background agents for the spec, the review and the board, checks what each one wrote, puts the plan on its own branch, and moves the milestone only when you approve. It never merges. |
| `snp-retro` | Grades a shipped milestone's predictions against what happened, and records the learnings. |

The `snp-*` agents are the experts and helpers these skills call. They are Claude-only.

## Where they are installed

`snp-install.sh` installs them for you. To do it yourself, run `snp skills install --user`:

| What | Where |
|---|---|
| Skills | `~/.claude/skills/` and `~/.agents/skills/`, where other agents read them |
| Agents | `~/.claude/agents/` only |

If `~/.agents` is in the way (a symlink, say), `--user` skips it with a warning and still installs `~/.claude`.

To install into one project instead, run `snp skills install --dir .` from its root. It writes the skills to `.claude/skills/` and the agents to `.claude/agents/`, and git must ignore both: add `.claude/skills/snp-*` and `.claude/agents/snp-*` to `.gitignore` first.

Pick one of the two. If both exist, Claude Code uses the user skill and the project agent. To catch a stale copy after an upgrade, run `snp skills install --user --check`, or `snp skills install --dir . --check` in a project.

`snp skills uninstall --user` removes what `--user` installed and leaves your other skills and agents alone.

[cli.md](cli.md) has the full `snp skills` pages.
