# Claude self-hosted environment on macOS

A [self-hosted environment](https://code.claude.com/docs/en/self-hosted-environments)
lets Claude Code cloud sessions run on a machine you own. The host runs
`claude self-hosted-runner`, which registers with Anthropic using an environment
key and picks up sessions dispatched to that environment.

Docs:

* [Overview](https://code.claude.com/docs/en/self-hosted-environments)
* [Quickstart](https://code.claude.com/docs/en/self-hosted-environments-quickstart.md)
* [Deploy to production](https://code.claude.com/docs/en/self-hosted-environments-deploy.md)
* [Reference](https://code.claude.com/docs/en/self-hosted-environments-reference.md)

## Prerequisites

* Team or Enterprise plan; an Owner turns on "Allow self-hosted environments" at
  claude.ai → Admin settings → Cloud environments, and creates the environment.
* Claude Code 2.1.224 or later on the host (`claude self-hosted-runner --help`
  works). Here: 2.1.325 from the `claude-code@latest` cask.
* Git 2.24 or later. macOS or Linux; Windows can't host a runner.
* The environment key is shown once when created. Copy it then.

## Setup used here

Runner user `testbot`, which auto-logs in at boot, so a LaunchAgent starts at
boot without anyone signing in. Check with:

```bash
defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser   # testbot
pmset -g | grep ' sleep'                                                  # 0
```

### 1. Directories

```bash
umask 077
mkdir -p ~/.config/claude-runner ~/claude-runner ~/Library/Logs
chmod 700 ~/.config/claude-runner ~/claude-runner
```

`~/claude-runner` is the checkout base. Keep it separate from your own clones.

### 2. The key

Paste it yourself, so it never goes through a Claude conversation:

```bash
ssh -t testbot@macos-claude '(umask 077 && cat > ~/.config/claude-runner/environment-secret)'
# paste the key, Enter, Ctrl-D
```

### 3. LaunchAgent

`~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.anthropic.claude-self-hosted-runner</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/claude</string>
    <string>self-hosted-runner</string>
    <string>--environment-secret-file</string><string>/Users/testbot/.config/claude-runner/environment-secret</string>
    <string>--base-dir</string><string>/Users/testbot/claude-runner</string>
    <string>--client-label</string><string>macos-claude</string>
    <string>--health-port</string><string>0</string>
    <string>--log-file</string><string>/Users/testbot/Library/Logs/claude-self-hosted-runner.log</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key><string>/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    <key>HOME</key><string>/Users/testbot</string>
  </dict>
  <key>WorkingDirectory</key><string>/Users/testbot/claude-runner</string>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>30</integer>
  <key>ProcessType</key><string>Background</string>
  <key>StandardOutPath</key><string>/Users/testbot/Library/Logs/claude-self-hosted-runner.out.log</string>
  <key>StandardErrorPath</key><string>/Users/testbot/Library/Logs/claude-self-hosted-runner.err.log</string>
</dict>
</plist>
```

Choices:

* `/opt/homebrew/bin/claude` is the cask's stable link, so upgrades don't break
  the path.
* `KeepAlive`: with the default `--drain-grace-sec 0` and `--capacity 1`, the
  runner exits after each session. launchd restarts it after the throttle
  interval.
* `--health-port 0`: the default health port is 8080, which DDEV uses as its
  router HTTP port under rootless Podman and socktainer.
* launchd gives agents a minimal PATH; set it explicitly so sessions find
  Homebrew's `git`, `docker`, and so on.

### 4. Start and check

```bash
plutil -lint ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.anthropic.claude-self-hosted-runner.plist
tail ~/Library/Logs/claude-self-hosted-runner.log      # want: Registered: runner_id=...
launchctl print gui/$(id -u)/com.anthropic.claude-self-hosted-runner | grep -E 'state|pid'
```

Stop or reload with `launchctl bootout gui/$(id -u)/com.anthropic.claude-self-hosted-runner`.

## Security notes

* Anyone in the org who can dispatch to the environment runs code as `testbot`,
  with `testbot`'s files, Docker access, and credentials. Keep only what every
  member may use.
* Anyone with the key can register runners and take sessions queued on the
  environment. Rotate it from the environment's Configuration tab if it leaks.
  It expires 365 days after creation.
* Git push from sessions needs credentials that work without a prompt; set those
  up separately. `--use-anthropic-git-proxy` is the managed alternative, and it
  rewrites the user's global git config.

## Getting Claude to do this

Claude Code didn't know the runner's commands from memory, and
`claude --help` only shows `--environment <id>` (start a session on an
environment), not the runner. A `claude-code-guide` subagent found the docs;
`claude self-hosted-runner --help` on the host gave the real flags. The docs
have no launchd recipe, so the LaunchAgent above is ours, not Anthropic's.
