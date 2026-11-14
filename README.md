# claude-notes

Notes from using Claude Code on real infrastructure work: what worked, what
broke, and what to do differently next time.

| Page | Covers |
| --- | --- |
| [Setting up a test runner with Claude](test-runner-setup-with-claude.md) | Having Claude provision a macOS DDEV test runner with every Docker provider, over ssh |
| [Claude self-hosted environment on macOS](claude-self-hosted-environment.md) | Running `claude self-hosted-runner` as a LaunchAgent that starts at boot |
| [Tailscale complexities](tailscale-complexities.md) | Tailscale SSH policy denials, tags versus owned devices, screen sharing paths |
| [macOS Screen Sharing and Remote Management](macos-screen-sharing.md) | Why Screen Sharing rejected a correct password, and the `kickstart` fix |

The machine in these notes is `macos-claude`, an Apple Silicon Mac on macOS 27,
reached over a Tailscale tailnet, with an auto-login `testbot` user.
