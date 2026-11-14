# macOS Screen Sharing and Remote Management

Symptom: Screen Sharing to `macos-claude` failed with a correct password, while
port 5900 was open over both the LAN and Tailscale.

## Cause

Remote Management (ARD) was on. When it is, it owns port 5900 and checks each
user's Remote Management privileges, rejecting a user without "Observe" and
"Control" as if the password were wrong.

The privileges live in the user record's `naprivs` attribute:

```bash
dscl . -read /Users/testbot naprivs
```

| Machine | `naprivs` | Meaning |
| --- | --- | --- |
| working testbot | `-1073741569` (`0xC00000FF`) | all privileges |
| `macos-claude` before | `-2147483395` (`0x800000FD`) | Control and Observe missing |

Comparing against a working runner over ssh found this in one command.

## Fix

```bash
ssh -t testbot@macos-claude 'sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -configure -access -on -users testbot,rfay -privs -all -restart -agent'
```

Then `naprivs` read `-1073741569`, and `vnc://macos-claude` worked.

## kickstart gotchas

* `-activate` failed over ssh with
  `Can't call method "print" on an undefined value at .../kickstart line 695`.
  That line writes `/Library/Application Support/Apple/Remote Desktop/RemoteManagement.launchd`,
  which root can't write from an ssh session without Full Disk Access
  (System Settings → General → Sharing → Remote Login → ⓘ → "Allow full disk
  access for remote users"). If that file already exists, Remote Management is
  already active and `-activate` isn't needed.
* A failed `-activate` aborts before `-configure`, so nothing changes. Recheck
  `naprivs` after every attempt rather than trusting the output.
* The System Settings route always works: General → Sharing → Remote Management
  → ⓘ, all options on for the user.

## Remote Management or plain Screen Sharing?

DDEV's docs ship `macos_screen_sharing.sh`, which turns Remote Management off
and uses plain Screen Sharing. The existing DDEV testbots use Remote Management
with full privileges instead. Either works; mixing them up is what produces the
"wrong password" failure.
