# Commit identity and signed commits on shared runners

Commits from runner sessions should name the person who started the session,
and a person's signing key must never be on a runner that other people's
sessions use. These notes say how both runners, `coder.ddev.com` and
`macos-claude`, get there.

## What went wrong

* **Commits said Claude.** Before the runners, sessions committed with the
  machine's `~/.gitconfig`, which named Randy. With `--use-anthropic-git-proxy`
  the runner deletes `~/.gitconfig` before every session, so the identity has
  to come from somewhere else. The Coder runner module put
  `Claude <noreply@anthropic.com>` in `/etc/gitconfig`, and `macos-claude` used
  `--configure-git`, which sets the same identity.
* **Commits weren't signed in a way that helps.** Kiri
  (`libracoder/kiri`), for one, requires signed commits.

## Signing: re-sign on your own computer

Options considered:

| Option | Verdict |
| --- | --- |
| A personal signing key on the runner (a key file, the Coder workspace key, or a forwarded agent) | No. Every session on the runner, whoever started it, could sign as that person. A forwarded agent is no better: while it's connected any session can ask it to sign, and a 1Password prompt can't say which session asked. |
| `--configure-git`: Anthropic's signing service | Commits are verified as GitHub's `claude` account. That proves only that Claude's service made the commit. And once the committer is a person, GitHub marks the signature "Unverified", because the key belongs to `claude` and the email doesn't. |
| Squash-merge in the GitHub web UI | GitHub signs the squash commit, and the person stays the author. Fine for ddev repos, where we merge. Doesn't help with a PR to someone else's repo. |
| **Re-sign on your own computer before opening the PR** | **What we do.** The key never leaves your machine. |

Re-signing a branch a session pushed:

```bash
git fetch origin <branch>
git switch <branch>
git rebase --exec 'git commit --amend --no-edit -n -S' origin/main
git log --show-signature origin/main..HEAD   # every commit: Good "git" signature
git push --force-with-lease
```

`--amend` keeps the author and sets the committer to your own git identity,
which is what GitHub checks the signature against.

How GitHub decides "Verified", learned along the way:

* The signing key must be registered on the account as a **Signing Key**. An
  SSH key registered only for authentication doesn't count, though the same
  key can be added a second time as a signing key.
* The **committer** email must be a verified email on that account.

## Identity: the session's GitHub account

The identity that already acts for a session on GitHub is the GitHub account
connected to the claude.ai account that started it: HTTPS clones and pushes
and the GitHub MCP tools all use it. Commits should name the same account.

### Where it comes from

Three sources were considered:

* **The session token** (`CLAUDE_CODE_SESSION_ACCESS_TOKEN`, verified with
  `claude self-hosted-runner decode-token`) names the claude.ai account,
  `.act.email`. That's a login, not a git identity. It's often a work address
  that isn't the person's git email, and it can be a role account such as
  `accounts@ddev.com` that several people share. Rejected.
* **The machine's own `gh` login** is the runner owner's, whoever starts the
  session. Never used.
* **GitHub's `/user`, asked through the session's git proxy.** The proxy adds
  the session's GitHub credential to requests that carry the placeholder
  token `proxy-injected`, which is what the session's `gh` shim
  (`$CLAUDE_CONFIG_DIR/ccr/bin/gh`) does. In a session on `coder.ddev.com`
  this returned `rfay`, id 112444, `Randy Fay`, `randy@randyfay.com`. Chosen.

The proxy belongs to the session's own `claude` process, so nothing that runs
before Claude starts (an `--exec-path` wrapper, for example) can use it. The
lookup runs as a `SessionStart` hook.

### The hook

[`claude-session-identity`](https://github.com/ddev/coder-ddev/blob/main/modules/claude-self-hosted-runner/files/claude-session-identity):

* reads the proxy from the session's git config
  (`http.https://github.com/.proxy`) and the CA from
  `$CLAUDE_CONFIG_DIR/ccr/ca-bundle.crt`, and asks for `/user`;
* uses the profile name, else the login, and the public email, else
  `<id>+<login>@users.noreply.github.com`, so commits are credited to that
  account either way;
* writes `GIT_AUTHOR_*` and `GIT_COMMITTER_*` to `$CLAUDE_ENV_FILE`, which
  Claude Code applies to the session's Bash commands, and tells Claude the
  identity in its output;
* changes nothing when any step fails, so commits fall back to the system
  git config's `Claude <noreply@anthropic.com>`.

Not yet verified: that `CLAUDE_ENV_FILE` exports reach Bash commands in a
runner session. Claude Code documents it, and the hook does get the variable,
but a nested `claude -p` couldn't authenticate to prove the rest. Check it in
the first session after deploying:

```bash
env | grep -E '^GIT_(AUTHOR|COMMITTER)_'
```

It depends on two internals of the git proxy: the `proxy-injected` token and
the `ccr/ca-bundle.crt` path. If Anthropic changes them, the hook fails safe
to the Claude identity. A supported way to learn the session's GitHub login
would be worth asking for.

## coder.ddev.com

The `claude-self-hosted-runner` module in `ddev/coder-ddev` installs the hook
as `~/.local/bin/claude-session-identity` and adds it to the `SessionStart`
hooks in `/etc/claude-code/managed-settings.d/coder-runner.json`.
`/etc/gitconfig` keeps `Claude <noreply@anthropic.com>` only as the fallback.
After the template is pushed, restart the workspace.

## macos-claude

As `testbot`:

1. Install the hook. It needs `curl` and `jq`; macOS ships `/usr/bin/jq` since
   macOS 15.

   ```bash
   mkdir -p ~/.local/bin
   curl -fsSLo ~/.local/bin/claude-session-identity https://raw.githubusercontent.com/ddev/coder-ddev/main/modules/claude-self-hosted-runner/files/claude-session-identity
   chmod +x ~/.local/bin/claude-session-identity
   ```

   Before the `ddev/coder-ddev` change merges, `scp` the file from a checkout
   of its branch.

2. Add it to the host config directory's
   `~/.config/claude-runner/host-config/settings.json`, merging with what's
   there:

   ```json
   {
     "hooks": {
       "SessionStart": [
         {
           "matcher": "startup|resume",
           "hooks": [
             { "type": "command", "command": "/Users/testbot/.local/bin/claude-session-identity" }
           ]
         }
       ]
     }
   }
   ```

3. Remove `--configure-git` from the LaunchAgent. Its signing would mark every
   commit with a person as committer "Unverified".

4. Without `--configure-git`, nothing provides a fallback identity, so set one
   in the system config of the `git` sessions use. Homebrew's git keeps it in
   `/opt/homebrew/etc/gitconfig`, not `/etc/gitconfig`:

   ```bash
   sudo /opt/homebrew/bin/git config --system user.name Claude
   sudo /opt/homebrew/bin/git config --system user.email noreply@anthropic.com
   ```

5. Restart the runner between sessions. It reads the host config directory
   only at startup.

   ```bash
   plutil -lint ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
   launchctl bootout gui/$(id -u)/com.anthropic.claude-self-hosted-runner
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
   ```

6. In a new session, check:

   ```bash
   env | grep -E '^GIT_(AUTHOR|COMMITTER)_'                 # the session's GitHub account
   git config --show-origin --get-regexp '^(gpg|commit)\.'   # nothing left from --configure-git
   ```

   If `commit.gpgsign` is still set, the git proxy kept the old signing
   settings in the rewritten `~/.gitconfig`; record what it wrote.

## Status

* `coder.ddev.com`: hook tested in a session against the real proxy (found
  `@rfay`, wrote the env file, failed safe without an env file or a proxy);
  template tests pass. In
  [ddev/coder-ddev#268](https://github.com/ddev/coder-ddev/pull/268), not
  yet deployed.
* `macos-claude`: not yet changed.
