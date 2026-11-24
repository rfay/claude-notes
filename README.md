# claude-notes

Notes from using Claude Code on real infrastructure work: what worked, what
broke, and what to do differently next time.

| Page | Covers |
| --- | --- |
| [Setting up a test runner with Claude](test-runner-setup-with-claude.md) | Having Claude provision a macOS DDEV test runner with every Docker provider, over ssh |
| [Claude self-hosted environment on macOS](claude-self-hosted-environment.md) | Running `claude self-hosted-runner` as a LaunchAgent that starts at boot |
| [Git and config for a runner that serves an org](self-hosted-runner-org-git.md) | Git credentials, host config and isolation when the runner serves an org; the git proxy and its alternatives |
| [Commit identity and signed commits](commit-identity-and-signing.md) | Commits that name the session's GitHub account, and why signing happens on your own computer |
| [Push approvals you can read](readable-push-approvals.md) | Getting Claude to push with one plain `git push` you can approve, and where that rule lives |
| [Tailscale complexities](tailscale-complexities.md) | Tailscale SSH policy denials, tags versus owned devices, screen sharing paths |
| [macOS Screen Sharing and Remote Management](macos-screen-sharing.md) | Why Screen Sharing rejected a correct password, and the `kickstart` fix |
| [Keeping a remote MacBook awake](macos-keep-awake.md) | Letting a runner keep working with its lid closed |

The machine in these notes is `macos-claude`, an Apple Silicon Mac on macOS 27,
reached over a Tailscale tailnet, with an auto-login `testbot` user.
