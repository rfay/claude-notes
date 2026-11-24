# Push approvals you can read

Claude may push to forks, but only after someone approves the command. That
approval means nothing if the prompt is a script. We kept seeing prompts like
this, and they couldn't be approved responsibly:

```bash
bash -c "cd /path/to/ddev && (git remote get-url rfay >/dev/null 2>&1 || git remote add rfay git@github.com:rfay/ddev) && for i in 1 2 3 4; do git push -u rfay HEAD && break; sleep \$((2**i)); done"
```

A person pushing by hand has a tracking branch and types `git push rfay
<branch>` or `git push --force-with-lease rfay <branch>`. Claude should
produce that same command.

## Why Claude writes scripts

* **Setup in the push command.** The checkout has no fork remote, either
  because nobody added one or because session prep removed it (on
  `macos-claude` the runner resets checkouts). Claude then guards the push
  with `git remote get-url … || git remote add …`.
* **A `bash -c` preference.** A personal `CLAUDE.md` rule wrapped every
  command that didn't start with an executable, so `cd … && git push` came
  out as a quoted script.
* **Built-in retry guidance.** The remote session prompt says to retry
  network errors with backoff, and Claude writes that as a loop.
* **Wrong URL scheme.** An SSH remote where only the HTTPS proxy works, or
  the other way round, fails, and Claude adds fallbacks.
* **Identity checks.** The coder-ddev skill asks Claude to compare
  `ssh -T git@github.com` with `get_me` before an SSH push. That's fine as
  its own step, but it often ended up in the same command as the push.

## The rule

Give Claude the same rule wherever it runs:

1. Setup is a separate command: `git remote add <owner> <url>`, alone.
2. The push is one plain command naming remote and branch:
   `git push -u <remote> <branch>`, `git push <remote> <branch>`, or
   `git push --force-with-lease <remote> <branch>`. No `&&`, `;`, pipes,
   loops, `sleep`, variables, `cd` or `bash -c`.
3. Before the push, Claude names the branch, the remote URL, the commits (from
   a separate `git log --oneline`), and whether it's a force push and why.
4. A network error gets reported and the same command is run again as a new
   call, never in a loop.

Naming the remote and branch, rather than relying on bare `git push`, makes
the destination visible in the prompt. In `ddev/ddev` it also keeps the
command inside the `Bash(git push *)` `ask` rule, which doesn't see
`git -C dir push` or `bash -c`.

## Where it lives

| Place | Covers | Notes |
| --- | --- | --- |
| `ddev/ddev` `AGENTS.md`, Git workflow | Everyone working in `ddev/ddev`, any agent, any machine | The `ask` rule and `pre-push` hook stay as the enforcement |
| coder-ddev `modules/claude-self-hosted-runner/files/SKILL.md` | Every coder.ddev.com runner session, any repo | Fork remotes use SSH there, through the workspace agent relay |
| `rfay/dotfiles` `claude/CLAUDE.md` | Randy's own machines and coder workspaces | Personal: remote name `rfay`, SSH or HTTPS by environment |
| `macos-claude` host config | Every session on that runner, any repo | Org-neutral, so no personal remote names; see below |

## Setting it up on macos-claude

The runner reads `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR`
(`/Users/testbot/.config/claude-runner/host-config`) only at startup, and
everything there applies to every org member's session. Add the org-neutral
version to its `CLAUDE.md`:

```markdown
## Pushing to forks

Git reaches GitHub here only through the session's HTTPS proxy, and only for
repositories attached to the session. To push to a fork, attach it first,
then add it as a remote with its HTTPS URL. Session prep removes remotes
added by hand, so check `git remote -v` in every session.

People approve a push by reading its command, so make it look like what they
would type:

- Add a missing remote alone: `git remote add <owner> https://github.com/<owner>/<repo>`.
- Push with one plain command naming remote and branch: `git push -u <remote> <branch>`,
  `git push <remote> <branch>`, or `git push --force-with-lease <remote> <branch>`.
  No `&&`, `;`, pipes, loops, `sleep`, variables, `cd` or `bash -c`.
- Right before the push, say which branch goes to which remote URL, list the
  commits from a separate `git log --oneline`, and say whether it rewrites
  the branch and why.
- On a network error, report it and run the same command again as a new
  call. Never write a retry loop.
```

Then restart the runner as in
[Git and config for a runner that serves an org](self-hosted-runner-org-git.md):

```bash
launchctl bootout gui/$(id -u)/com.anthropic.claude-self-hosted-runner
```

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
```

Wait until no one has a session running, since the restart releases it.
