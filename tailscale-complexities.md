# Tailscale complexities

Context: DDEV test runners live in a Tailscale tailnet, tagged
`tag:ddev-runner`, with the access policy from DDEV's
[remote access docs](https://docs.ddev.com/en/latest/developers/buildkite-testmachine-setup/#remote-access-with-tailscale).

## "tailnet policy does not permit you to SSH to this node"

Symptom: after enabling Tailscale SSH on `macos-claude`, plain `ssh`, `tailscale
ssh`, and (apparently) everything else failed:

```text
tailscale: tailnet policy does not permit you to SSH to this node
Connection closed by 100.x.y.z port 22
```

Plain `ssh` had worked before.

Why:

* `tailscale set --ssh` (or `tailscale up --ssh`) makes tailscaled answer
  **every** port-22 connection that arrives over the tailnet. Your Mac's own
  OpenSSH no longer sees them. `ssh -v` shows the server as
  `remote software version Tailscale`.
* Tailscale then checks the policy's `"ssh"` section and denies before any key
  is tried. Your `~/.ssh` keys don't matter.
* The policy's `ssh` rule only had `"dst": ["tag:ddev-runner"]`. `macos-claude`
  was still a device **owned by a user**, untagged, so no rule matched.

The confusing part: `ssh testbot@tb-macos-arm64-4` worked, so it didn't look like
a policy problem. That runner also uses Tailscale SSH, but it is tagged
`tag:ddev-runner`, which the rule covers.

Fixes, any one:

* Tag the machine (`tag:ddev-runner`), which is what we did. Tagging changes the
  owner to "tagged-devices" and disables key expiry.
* Add a rule for your own devices:

  ```json
  {"action": "accept", "src": ["autogroup:member"], "dst": ["autogroup:self"], "users": ["autogroup:nonroot"]}
  ```

  `autogroup:self` matches only user-owned devices, never tagged ones.
* Turn Tailscale SSH off on that node: `tailscale set --ssh=false`. OpenSSH then
  answers over the tailnet again.

Fallback while locked out: ssh to the LAN address or `.local` name, which goes
to OpenSSH, not Tailscale SSH.

## Users in a Tailscale SSH rule

`"users": ["autogroup:nonroot"]` lets you log in as any non-root local user. If
a rule instead lists specific users (for example `testbot`), logging in as any
other user is denied by policy, which looks the same as the error above.

## Screen sharing: is it Tailscale?

Usually not. Check the port before blaming the tailnet:

```bash
nc -z -w 5 macos-claude 5900        # over Tailscale
nc -z -w 5 macos-claude.local 5900  # over the LAN
```

Both open meant the network path was fine and the failure was authentication,
which turned out to be Remote Management privileges
([Screen Sharing page](macos-screen-sharing.md)).

Notes:

* The policy's `grants` must list `tcp:5900`. The tailnet silently drops
  unlisted ports, so working ssh proves nothing about 5900.
* `macos-claude.local` (Bonjour, LAN) and `macos-claude` (MagicDNS, tailnet)
  reach the same server. Having both doesn't break anything, but saved entries
  can hold stale LAN addresses: our notes had `192.168.1.30` while the machine
  was on `192.168.50.134`.
* `vnc://macos-claude` from Finder or a browser worked when picking the machine
  inside the Screen Sharing app didn't. The app's own list may hold an old
  entry or try High Performance mode; use `vnc://` or a fresh ⌘N connection.

## Smaller things

* "client version ... != tailscaled server version" is a harmless warning when
  the CLI and daemon differ by a patch release.
* `tailscale status` shows `tagged-devices` as the owner of tagged nodes; that's
  how to confirm a tag took effect.
* Tag owners must be saved in the policy before a node advertises the tag, or
  the tag is silently refused.
