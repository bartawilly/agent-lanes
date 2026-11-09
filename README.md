# agent-lanes

These are the Claude Code subagent definitions I use to build Erenna, my own product, with AI agents writing most of the code. A lead session that I drive writes a brief for each task and hands it to a builder agent that works in its own git worktree, at a tier chosen by how risky the task is. The builder writes tests first, runs the gates, checks its plan with a second model, and stops at "ready to push" with a short evidence report. A person does everything that reaches the shared repository. This repo holds the three agent definitions, an example brief, an example report, and notes on what is enforced and what is not. I wrote it up so readers of [iamahmed.com](https://www.iamahmed.com) can look at the real thing rather than a description of it.

## The flow

```
  lead                      builder lane                         person
  ----                      ------------                         ------
  writes brief  ------->    own git worktree      ------->       reads report
  (scope, files,            tier set by risk                     pushes
   pinned base,             tests first                          opens the PR
   tier)                    gates, exit codes kept               AI review in CI
                            advisor checks the plan              merges
                            HOLDS at READY-TO-PUSH
                            ~1 KB evidence report

  light tier: if the task needs judgement, hand it back to the lead
```

1. The lead writes a brief: what to build, the files and doc sections that matter, the base commit to build on, the tier, and any extra rules. A template is in [`examples/brief-template.md`](examples/brief-template.md).
2. A builder takes the task in its own git worktree, so parallel tasks never share a checkout.
3. It loads its role skills, writes the tests first, implements, and runs every gate the brief names, keeping the exit codes.
4. It consults a second, stronger model (the `advisor` tool in my Claude Code setup) after reading the brief, before its first edit, before every report, and whenever it is stuck.
5. It stops at READY-TO-PUSH and sends the report described below.
6. I read the report, push, and open the pull request. An AI review runs in CI. A person merges.

## Three tiers

All three run on Opus. The tier sets the reasoning effort and how much the agent is allowed to decide on its own.

| Tier | File | Effort | Gets |
| --- | --- | --- | --- |
| Light | [`agents/worker-light.md`](agents/worker-light.md) | `medium` | Mechanical, well-specified work with a pattern to copy: copy or string alignment, a single-file rename, porting tests to a new input shape, read-only sweeps that feed the lead's decision. |
| Standard | [`agents/builder-standard.md`](agents/builder-standard.md) | `high` | Normal implementation in a known module with a clear spec and an existing pattern. |
| Deep | [`agents/builder-deep.md`](agents/builder-deep.md) | `xhigh` | Anything risky: security, tenant isolation, payments and other money paths, data migrations and seeds, cross-module event flows, investigations with real trade-offs, and UI that has to pass the design gate. |

The deep list is a rule, not a guideline: work in those areas always goes to the deep tier, however small it looks.

The light tier is not allowed to stretch. If a light task turns out to need design judgement, carries risk, or has an ambiguous spec, the worker stops and reports that the task is mis-tiered. I re-dispatch it at the right tier. The rule exists because an improvising worker's report looks as confident as a correct one.

## The report contract

Standard and deep builders end every task with a report of about a kilobyte, in a fixed order:

1. First line: the final SHA from `git rev-parse HEAD` and the exit code of `git merge-base --is-ancestor <pinned-base> HEAD`, so I know the work sits on the base I asked for.
2. Attestation: which skills were loaded and which were not, and which documents were read.
3. A gate table with the exit code of every gate the brief names.
4. The evidence for the behaviour the brief asked for.
5. Disclosed gaps, ranked by how much they matter.
6. The docs the builder updated.
7. An `advisor:` line recording the second-model check.
8. A `LESSON:` block, or the word `none`.

A light worker reports more simply: what it changed, what it ran with exit codes, and anything it noticed but did not touch.

[`examples/ready-report.md`](examples/ready-report.md) shows the shape. It is made up, and says so.

## Standing rules

Every agent definition carries these. A brief can add rules but never relax them.

- Never push, open a pull request, merge, or post to an outside service. The lead does all of that.
- Never write a mention that would summon the CI reviewer.
- No commit trailers, and no work-item or PR numbers in code comments.
- Never read, write or print `.env` contents.
- Never touch staging or production data or any shared database. Migrations run only against the lane's own throwaway database, built from empty.
- Work only inside your own worktree, with absolute paths on every shell call. Never touch another lane.
- Commit only named files. After a READY report, add new commits rather than amending.

**How these are enforced today:** worktree isolation is structural; each lane has its own checkout, so one lane's edits cannot land in another's files. Everything else in the list above is a standing rule written into the agent prompt. Claude Code does not mechanically stop an agent that breaks one of them: whether a stray `git push` prompts me or just runs depends on the permission mode the session is in, and nothing in the setup denies it outright. A worktree also shares the repository's remotes, so a lane could technically push if it ignored its rules and had credentials. The protection today is the prompt, the advisor checkpoints, and me reading every report and diff before anything is pushed. The next section is how I would close that gap.

## Lean harness

The checks have to be proportional to the task, or the reports stop being read. So:

- Reports stay around a kilobyte. A report I skim is worse than no report.
- Builders read the sections of large documents that the brief names, never the whole document.
- Mutation-style proofs are kept for the few core invariants (at most five), such as tenant isolation or money totals. A mutation-style proof means deliberately breaking the invariant in the code and showing that the test then fails, which proves the test can catch the problem. Doing that for every test would cost more than it finds.

## Cloud parity

I also run these agents in Claude Code cloud sessions, which do not get my local `~/.claude/agents` or skills. The same agent definitions, and the skills they use, are packaged in a private kit. The cloud environment's setup script clones the kit and runs its install script, which only installs when `CLAUDE_CODE_REMOTE=true`. A cloud session therefore runs the same definitions as my machine. The kit stays private because it bundles third-party skills under their own licences; this repo publishes only the definitions I wrote.

The definitions here have been made generic: the product name, internal paths and project-specific details are taken out. Step 1 of each builder refers to a project "grounding skill". That skill is not included. In my setup it chooses the role skills for the task (for example, test-first development or UI design) and names the doc sections to read. If you copy these files, replace that step with your own. The builders also call an `advisor` tool that my setup provides; if yours has no such tool, use a review subagent at the same checkpoints or drop that step.

To try them, copy the files in `agents/` into `.claude/agents/` in a project or `~/.claude/agents/` for your user.

## Hardening

This is a recommended next step, not what runs today. The aim is to turn the most important standing rules into settings that Claude Code enforces, whatever the prompt says.

A project `.claude/settings.json`, committed so that every worktree checks it out:

```json
{
  "permissions": {
    "deny": [
      "Bash(git push *)",
      "Bash(gh pr create *)",
      "Bash(gh pr merge *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Edit(**/.env)",
      "Edit(**/.env.*)"
    ]
  }
}
```

Notes, from the [Claude Code permissions docs](https://code.claude.com/docs/en/permissions):

- Deny rules are checked before ask and allow rules, from every settings scope, and a settings-level deny also applies to subagents.
- `Bash(git push *)` also matches a bare `git push`. `Bash(git push:*)` is an equivalent spelling.
- `Read(./.env)` only matches the `.env` in the current directory. `**/.env` matches nested ones too, which matters in a monorepo with an env file per app.
- Read and Edit deny rules cover Claude's file tools and shell commands Claude Code recognises, such as `cat`, `head`, `sed` and redirections. They do not cover a script that opens the file itself.
- Bash rules match the command text, so they are not a security boundary around the program. The docs give `git -C . push` as a push that `Bash(git push *)` does not match. For OS-level limits on files and network, turn on Claude Code's [sandboxing](https://code.claude.com/docs/en/sandboxing); for custom checks on the full command, use a `PreToolUse` hook.
- Do not put `Bash(git push *)` in a subagent's `disallowedTools` frontmatter. With a pattern there, Claude Code removes the whole Bash tool from that subagent. The settings deny list is the right place.

The real boundary for "never push" is on the remote: branch protection on the main branch, and agent sessions running with a credential that cannot push or merge. The deny list catches the ordinary case early; the remote settings stop the rest.

## Licence

MIT, © 2026 Ahmed Nasser. See [LICENSE](LICENSE).

More about how I work: [https://www.iamahmed.com](https://www.iamahmed.com)
