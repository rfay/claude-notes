# Read-only GitHub for sessions on the macOS runner

Sessions on `macos-claude` read and search all of public GitHub through
`gh-read`, a wrapper that runs the real `gh` with a read-only token. The full
`gh` login the runner used to have is gone.

## Why

A session's own `gh` is a shim that reaches only the repositories attached to
the session. A session wanting to read an upstream issue elsewhere called
`/opt/homebrew/bin/gh` by its full path instead, which was logged in as
`rfay` with `repo`, `read:org` and `gist` scopes. Every session on the runner,
whoever started it, could therefore act as rfay with write access to
everything rfay can write.

Now:

* `/opt/homebrew/bin/gh` is logged out (`gh auth status` says "not logged into
  any GitHub hosts").
* `gh-read` gives every session read access to public GitHub, with a token
  that can do nothing else.
* Managed Claude Code instructions tell sessions to use it, and a hook stops
  sessions printing credentials.

## The pieces

| What | Where |
| --- | --- |
| `gh-read` wrapper | `/usr/local/bin/gh-read` |
| Deny hook | `/usr/local/bin/claude-deny-token-commands` |
| Token | login Keychain item `gh-read-token` (account `testbot`) |
| Managed instructions | `/Library/Application Support/ClaudeCode/CLAUDE.md` |
| Managed settings | `/Library/Application Support/ClaudeCode/managed-settings.json` |

`/usr/local/bin` was chosen because it is on the runner's PATH (set in the
LaunchAgent plist: `/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:...`)
and on the default macOS PATH for Terminal and SSH. It is writable by the
`admin` group, so no `sudo` is needed. `~/.local/bin` exists but is not on the
runner's PATH.

### gh-read

From ddev/coder-ddev,
`modules/claude-self-hosted-runner/files/gh-read`
([PR 267](https://github.com/ddev/coder-ddev/pull/267)). It:

* accepts only read commands: `api` with GET/HEAD (or a GraphQL query, not a
  mutation), `search`, and `view`/`list`/`diff`/`checks`/`status` of issues,
  PRs, repos, runs, releases, workflows, gists and labels. Anything else exits
  2 before any token is read.
* runs the real `gh` by absolute path with an empty temporary `GH_CONFIG_DIR`,
  so the machine's own `gh` config, login and aliases play no part.
* takes its token, outside Coder, from the Keychain item `gh-read-token`, or
  else from `~/.config/gh-read/token` (mode 600).

The command filter is a guard against mistakes. The real limit is the token.

### The token

A fine-grained personal access token on the GitHub account **`ddltest`**:

* Repository access: **Public repositories (read-only)**. No other
  permissions, so it can't write anything or see anything private.
* Expiration: **none**. Rotate it if it may have leaked (it appears in a
  commit, PR, log or session transcript), if someone who could read it
  leaves, or when the runner is rebuilt. Revoke the old one in ddltest's
  [fine-grained tokens](https://github.com/settings/personal-access-tokens).
* Limits: 5000 core requests an hour and 30 search requests a minute (10 for
  code search). An earlier session here hit the search limit by looping
  searches; the managed `CLAUDE.md` tells sessions to space searches out or
  use list endpoints.

Every process running as `testbot` can read the token, whether it is in the
Keychain or in the file. That is acceptable only because it is public-read-only.

## The Keychain error, and the fix

Storing the token first failed:

```text
MacBookAir:~ testbot$ security add-generic-password -a testbot -s gh-read-token -w
security: SecKeychainItemCreateFromContent (<default>): Write permissions error.
```

Nothing was wrong with the keychain. `security default-keychain` and
`security list-keychains -d user` both showed
`~/Library/Keychains/login.keychain-db`, owned by `testbot`, and a test item
written from a runner session succeeded. The command had been run over **SSH**
(Tailscale). An SSH login gets its own security session, in which the login
keychain is not unlocked, and `security` reports that as a write permissions
error.

The fix: run the same command in Terminal **on the Mac's own screen** (or
through Screen Sharing). It prompts for the token without echoing it:

```bash
security add-generic-password -a testbot -s gh-read-token -w
```

Over SSH, unlock the keychain first (`security unlock-keychain
~/Library/Keychains/login.keychain-db`, which asks for testbot's password).

The runner is a LaunchAgent in testbot's GUI session, and testbot logs in
automatically at boot, so the login keychain is unlocked whenever the runner is
running. If auto-login is ever turned off, or the login and keychain passwords
diverge, `gh-read` will stop finding the Keychain item: switch to the file.

### Alternative: a token file

The Keychain gives no extra protection here (`security` is on the item's ACL,
so any `testbot` process reads it without a prompt) and doesn't work from SSH.
A file works everywhere. In your own Terminal, paste the token and press Enter
(no prompt is shown, nothing is echoed or kept in history):

```bash
mkdir -p -m 700 ~/.config/gh-read && (umask 077; read -rs T && printf '%s\n' "$T" > ~/.config/gh-read/token)
```

Use one or the other: `gh-read` tries the Keychain first, so delete the
Keychain item (`security delete-generic-password -s gh-read-token`) when
switching to the file.

## Replacing the token

Make the new token on `ddltest` (fine-grained, public repositories read-only),
then on the Mac's screen:

```bash
security delete-generic-password -s gh-read-token
```

```bash
security add-generic-password -a testbot -s gh-read-token -w
```

For the file, rerun the `mkdir ... read -rs ...` command above; it overwrites
the old token. Never paste the token into a Claude session.

## Installing or updating gh-read and the hook

Read the scripts before installing them. Fetch into a scratch directory, then
install:

```bash
B=https://raw.githubusercontent.com/ddev/coder-ddev/main/modules/claude-self-hosted-runner/files
mkdir -p ~/tmp/gh-read-dl && cd ~/tmp/gh-read-dl
curl -fsSLO "$B/gh-read" && curl -fsSLO "$B/claude-deny-token-commands"
less gh-read claude-deny-token-commands
install -m 755 gh-read claude-deny-token-commands /usr/local/bin/
```

Until PRs [266](https://github.com/ddev/coder-ddev/pull/266) and
[267](https://github.com/ddev/coder-ddev/pull/267) merge, use their branches
instead of `main`: `20261124_rfay_gh_read_wrapper` for `gh-read` and
`20261124_rfay_block_token_commands` for the hook.

The hook needs `jq` (`/opt/homebrew/bin/jq`). Without it the hook reads an
empty command and lets everything through.

Install the hook last when setting up from scratch: it blocks
`security find-generic-password` and anything naming `gh-read/token`, which
setup steps may need.

## Managed Claude Code files

Claude Code reads `/Library/Application Support/ClaudeCode/CLAUDE.md` and
`managed-settings.json` for every Claude Code process on the Mac, including
the runner's headless sessions. Nothing in user or project settings overrides
managed keys; `permissions.allow` lists merge with other scopes, and managed
hooks always run.

`CLAUDE.md` tells sessions to use `gh-read` for anything outside their attached
repositories, that its token reads public repositories only, to keep search
under 30 requests a minute, and never to print the token. The Coder version in
coder-ddev (`managed-CLAUDE.md`) was adapted, dropping the Coder login.

`managed-settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh-read:*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/claude-deny-token-commands"
          }
        ]
      }
    ]
  }
}
```

Both files are root-owned, mode 644. Install with:

```bash
sudo mkdir -p "/Library/Application Support/ClaudeCode"
```

```bash
sudo install -m 644 -o root -g wheel CLAUDE.md managed-settings.json "/Library/Application Support/ClaudeCode/"
```

If either file already exists, merge into it rather than overwriting: add the
`allow` entry and the `PreToolUse` entry to what is there.

## Checking it works

```bash
gh-read api user --jq .login                         # ddltest
gh-read issue list -R libracoder/kiri -L 3
gh-read pr view 542 -R socktainer/socktainer --comments
gh-read pr comment 1 -R x/y -b hi; echo $?           # "not allowed", 2
/opt/homebrew/bin/gh auth status                     # not logged in
```

The hook, without running anything it would block:

```bash
echo '{"tool_input":{"command":"gh auth token"}}' | /usr/local/bin/claude-deny-token-commands; echo $?   # 2
echo '{"tool_input":{"command":"ls -la"}}' | /usr/local/bin/claude-deny-token-commands; echo $?          # 0
```

In a new runner session, asking Claude to run `gh auth token` should be refused
with "Blocked: this command would print a credential".

On 2026-11-24 the `gh-read` reads, the write refusals (`pr comment`,
`issue create`, `api -X POST`, `api -f` without `-X GET`, a GraphQL mutation),
the logout and the hook's exit codes all passed. The hook blocked
`gh auth token`, `security find-generic-password ... -w`,
`cat ~/.config/gh-read/token`, `$(gh  auth token)` and `git credential fill`,
and let `gh-read`, `ls` and `git status` through.

## Still to do

* `gh auth logout` removes rfay's token from this Mac but doesn't revoke it.
  Revoke it in rfay's GitHub settings (Applications, Authorized OAuth Apps,
  GitHub CLI), bearing in mind that revokes `gh` logins on rfay's other
  machines too.
* Anyone who needs a full `gh` on this Mac should use a different macOS user,
  not `testbot`.
