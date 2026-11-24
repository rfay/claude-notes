# Git and config for a runner that serves an org

A self-hosted runner serves the whole org, not the person who set it up. Our first
setup on `macos-claude` treated it like a personal machine, and git, config and
credentials all went wrong. This page covers what the docs intend and what we
changed.

Docs:

* [Deploy to production](https://code.claude.com/docs/en/self-hosted-environments-deploy.md):
  git credentials, isolation, network
* [Customize sessions](https://code.claude.com/docs/en/self-hosted-environments-configuration.md):
  wrapper, hooks, host config
* [Reference](https://code.claude.com/docs/en/self-hosted-environments-reference.md):
  every flag and env var
* `claude self-hosted-runner --help` on the host, which is more detailed than
  the web pages

## What went wrong

The runner started with only `--environment-secret-file`, `--base-dir`,
`--client-label`, `--health-port` and `--log-file`.

* **No git credentials.** `git push` failed over SSH (`Permission denied
  (publickey)`) and over HTTPS (`could not read Username`).
  `check_repo_access` still reported push OK, because the session's GitHub
  credential reached only the GitHub MCP tools, never git. We worked around it
  by pushing with MCP `create_branch` and `push_files`, which made 9 API
  commits instead of one normal commit.
* **One person's config in every session.** The runner copies its host config
  directory into every session, and the default is the runner account's
  `~/.claude`. Ours held Randy's personal `CLAUDE.md` (push to the `rfay` fork
  over SSH, `YYYYMMDD_rfay_` branch names) and skills symlinked from his
  dotfiles. Every org member's session got them.
* **One person's git and SSH config.** `~/.gitconfig` had Randy's name and
  1Password commit signing (`op-ssh-sign`, missing on the host, so commits
  failed until signing was turned off). `~/.ssh/config` set
  `IdentityAgent ~/.1password/agent.sock` for every host, which overrides a
  forwarded agent.

## How runners are meant to work

* **One owner at a time.** "A runner serves one owner at a time." The first
  session locks the runner to its creator. With the default
  `--drain-grace-sec 0`, the runner exits when that owner's sessions end, and
  a supervisor (launchd `KeepAlive` here) starts a fresh one. One runner
  therefore serves one person at a time; more people at once need more
  runners, or `claude self-hosted-runner orchestrator`.
* **A dedicated account.** "Run the runner as a dedicated user or in a
  container, never as your own user." Every session runs as that user and can
  read whatever that user can, credentials included.
* **Host config is org config.** "The runner snapshots the host's `~/.claude/`
  at startup… applies to every session." Set
  `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR` to an org-neutral directory. Users'
  claude.ai account skills still reach the sessions they start.
* **Idle sessions hold the runner.** Anthropic sends each session an idle
  limit (about 30 minutes in our experience on coder.ddev.com), but the runner
  applies it only on Linux. On macOS a session keeps the runner until it ends
  or is archived, and every other session waits in the queue. Set
  `--release-idle-session-min` to release idle sessions on any OS. A released
  session isn't lost: the conversation is stored server-side, and the user's
  next message resumes it on whichever runner is free.
* **Leftover state is readable.** Without `--remove-session-state`, each
  session's transcript copy, debug log and session directory stay under
  `<base-dir>/_sessions/`, readable by later sessions. Removing them doesn't
  stop a session from resuming, because the transcript lives server-side.
  What goes is anything the session wrote in its own directory
  (`<base-dir>/_sessions/<id>/`), such as a repo it cloned there by hand.
  Canonical clones under `<base-dir>/<owner>/<repo>` stay.

## Prescription

For a GitHub-hosted org on a single macOS runner:

1. Use a dedicated runner account with no personal dotfiles: no personal
   `~/.gitconfig`, `~/.ssh/config` includes or `~/.claude`. Put operator-wide
   git settings in `/etc/gitconfig`, since the git proxy rewrites
   `~/.gitconfig`.
2. Add these flags to the runner:
   * `--use-anthropic-git-proxy`: git authenticates as the session's creator,
     with no credentials on the host.
   * Not `--configure-git`, which makes every commit Claude's. A
     `SessionStart` hook sets the commit identity to the session's GitHub
     account instead; see
     [Commit identity and signed commits](commit-identity-and-signing.md).
   * `--remove-session-state`: deletes each session's state when it ends.
   * `--release-idle-session-min 30`: releases a session after 30 minutes
     with no user message, so the next person's session can start.
   * `--push-outcome-on-release`: pushes the session's `claude/...` branch
     before a release deletes it locally, and fetches it back on resume.
     Without it, unpushed commits on that branch are lost. Uncommitted
     changes may still be lost, so commit and push before stepping away.
3. Set `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR` to an org-neutral directory. It
   starts empty; add an org `CLAUDE.md`, skills or hooks there, and restart
   the runner after changes, since it reads the directory only at startup.
4. Put personal preferences elsewhere: claude.ai account skills, your own
   laptop's `~/.claude`, or the repo's `AGENTS.md` when they're project rules.
5. Keep `--capacity 1`, which the git proxy requires.

The LaunchAgent from
[Claude self-hosted environment on macOS](claude-self-hosted-environment.md)
changes like this:

```xml
    <string>--client-label</string><string>macos-claude</string>
    <string>--use-anthropic-git-proxy</string>
    <string>--remove-session-state</string>
    <string>--release-idle-session-min</string><string>30</string>
    <string>--push-outcome-on-release</string>
    ...
  <key>EnvironmentVariables</key>
  <dict>
    ...
    <key>SELF_HOSTED_RUNNER_HOST_CONFIG_DIR</key><string>/Users/testbot/.config/claude-runner/host-config</string>
  </dict>
```

Switching over:

```bash
cp -p ~/.gitconfig ~/.gitconfig.personal-backup    # the proxy deletes ~/.gitconfig with no backup
mkdir -p ~/.config/claude-runner/host-config
plutil -lint ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
launchctl bootout gui/$(id -u)/com.anthropic.claude-self-hosted-runner
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
cat ~/.gitconfig    # now the runner's: Claude identity, code-sign, proxy credential helper
```

The restart doesn't lose a running conversation. The session is stored
server-side and resumes on the next message, on the new runner.

## Git credential options

| Option | How | Fits when | Watch out |
| --- | --- | --- | --- |
| Anthropic git proxy | `--use-anthropic-git-proxy` | Repos on github.com, each member pushing as themselves | github.com only; no branch deletion, no tag pushes, no private submodules or other private repos; `--capacity 1`; rewrites the runner account's `~/.gitconfig` and `~/.config/git` before every session |
| Per-session minted token | `--exec-path` wrapper that reads the creator from the `act` claim of `CLAUDE_CODE_SESSION_ACCESS_TOKEN` (`claude self-hosted-runner decode-token`) and mints a short-lived token for them | GHES, other git hosts, or rules the proxy can't express | You build and run the token service |
| Host token or deploy key | A narrowly scoped token in a `credential.helper`, or a deploy key with `url.<base>.insteadOf` | A small set of repos, everyone pushing as one identity | Every session can read it, so every member can use it. The docs warn against long-lived or broad push credentials on a shared runner. |
| SSH-only hosts | `--git-ssh-rewrite <host>` plus a key on the host | A git host that only speaks SSH | Same exposure as a host token |

The Deploy page also covers GHES and split-horizon DNS
(`--git-host-rewrite`).

## Behavior after switching to the proxy

Observed on `macos-claude`, Claude Code 2.1.325:

* `git push` over HTTPS to `github.com` works. The session's env points git at
  Anthropic's proxy through `GIT_CONFIG_*` and
  `CLAUDE_CODE_AGENT_PROXY_GIT_HOSTS=github.com`.
* Commits carry an SSH signature from the runner's signing key. `git commit
  --amend` keeps the original author and sets the committer to
  `Claude <noreply@anthropic.com>`.
* Only repos attached to the session can be reached. To push to a fork, attach
  it (`add_repo`, for example `rfay/ddev`); it gets its own clone with an HTTPS
  `origin`.
* The runner resets checkouts when it prepares a session. After the restart,
  the `ddev/ddev` checkout was on the session's branch, and a remote added by
  hand earlier (`rfay`) was gone. Local branches and commits stayed.
* Twice in one session, git lost its connection: `Failed to connect to
  127.0.0.1 port 62329`. The runner had rewritten the session's git config
  (`<base-dir>/_sessions/<id>.gitconfig`, `http.https://github.com/.proxy`)
  with a port nothing listened on, once after the session's folders changed and
  once after `add_repo`. The session's own `claude` process still listened on
  its original port, found with
  `lsof -nP -iTCP -sTCP:LISTEN | grep claude`, and
  `git -c http.https://github.com/.proxy=http://127.0.0.1:<that port> ...`
  worked. Probably a runner bug worth reporting.
* `gh` is a shim that supports only `gh api`, so open PRs with
  `gh api repos/{owner}/{repo}/pulls` or the GitHub MCP tools.
* The runner doesn't install the commit-and-push reminder that Anthropic-hosted
  sessions have. The Customize page has a Stop hook example to add to the host
  config directory.

## Pushing by hand on the runner host

The proxy credentials exist only inside sessions, so a person who SSHes in as
`testbot` has no git credentials. Use your own key through agent forwarding:

```bash
ssh -A testbot@macos-claude
export GIT_SSH_COMMAND='ssh -o IdentityAgent=SSH_AUTH_SOCK'   # only if ~/.ssh/config sets IdentityAgent
git remote add rfay git@github.com:rfay/ddev                   # remotes added by hand don't survive session prep
git push --force-with-lease=BRANCH:EXPECTED_SHA rfay BRANCH
```

Don't commit from that shell. `~/.gitconfig` now signs through the runner's
service and names Claude as committer.

## Other tools' credentials

The same rule applies to CLIs like Buildkite's `bk`: anything the runner
account holds, every member's session can use. `bk auth login` over SSH also
fails with `failed to store token in credential store: exit status 195`,
most likely because it stores tokens in the login keychain, which an SSH
login can't write.

What worked: run `bk auth login --org=ddev --token=...` from a Screen Sharing
session as `testbot`. The token lands in the login keychain, and sessions,
which run under the same launchd GUI domain, can read it. Give the token only
the scopes sessions need; ours has `read_builds` and `read_build_logs`, enough
for `bk build list` and job logs.

Without a GUI session, `bk` also reads `BUILDKITE_API_TOKEN` and
`BUILDKITE_ORGANIZATION_SLUG` from the environment, for example from the
runner plist's `EnvironmentVariables`. A per-owner token through an
`--exec-path` wrapper is the stricter option.

## Open questions

* Do plugins in the host config directory reach sessions? The docs don't say.
  Sessions do get the claude.ai account's and organization's plugins and
  skills: they land in the session's config under `plugins/synced/` and
  `skills/synced/`. Adding `ddev/agent-skills` to the claude.ai plugin catalog
  is probably the way to get it into sessions, but nobody has tried it yet.
* Is one runner enough, or do we need several runners or the orchestrator so
  more than one person can use the environment at once? Idle release shortens
  the wait but still serves one session at a time. Archiving a session in the
  Claude app also frees the runner at once.
* Does `--release-idle-session-min` skip sessions with uncommitted changes,
  as Anthropic's own idle limit does? The help doesn't say.
* GitHub's view of the commit signatures: does GitHub show the runner-signed
  commits as Verified?
