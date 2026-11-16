# Keeping a remote MacBook awake with the lid closed

A runner that sleeps drops off Tailscale, stops taking self-hosted Claude
sessions, and can make `claude self-hosted-runner` exit when it loses contact.
On a MacBook, `pmset sleep 0` (System Settings → Energy → "Prevent automatic
sleeping") is not enough: **closing the lid still sleeps the machine**, unless
it's in clamshell mode with an external display.

`macos-claude` is a MacBook on AC power, and had:

```text
sleep                0 (sleep prevented by powerd, screensharingd)
```

with no `SleepDisabled` line, so it would sleep when the lid closed.

## Allow lid-closed operation

```bash
sudo pmset -a disablesleep 1
pmset -g | grep SleepDisabled     # SleepDisabled 1
```

This turns off all sleep, including lid-close sleep. It takes effect at once,
with no reboot, and stays set across reboots.
Undo it with `sudo pmset -a disablesleep 0`.

Also keep the idle settings that already keep it awake with the lid open:

```bash
sudo pmset -c sleep 0 displaysleep 60 disksleep 10
```

## Testing it

On `macos-claude` (2026-11-16), with `disablesleep 1` and AC power:

1. Disconnect Screen Sharing first. While a viewer is connected,
   `screensharingd` holds a `PreventSystemSleep` assertion, so the test proves
   nothing. Check with `pmset -g assertions | grep PreventSystemSleep`.
2. Probe from another machine without logging in, since an ssh login can keep
   the Mac awake through `ttyskeepawake`:

   ```bash
   while true; do nc -z -G 5 -w 5 macos-claude 22 && echo "$(date +%T) ok" || echo "$(date +%T) FAIL"; sleep 10; done
   ```

3. Close the lid. The system log records it as `clamshelled: YES`:

   ```bash
   log show --last 1h --predicate 'eventMessage CONTAINS[c] "clamshelled: "' --style compact
   ```

Result: lid closed for 26 minutes, 149 of 149 probes answered, no Sleep entries
in `pmset -g log`, and the Claude runner kept polling every 3–4 minutes.

An earlier report that "Claude stopped responding when the lid closed" turned
out to be unrelated: the runner had been reloaded with new flags, which
interrupted the session.

## Cautions

* `disablesleep` applies on battery too. A closed MacBook that never sleeps,
  in a bag or on a soft surface, gets hot and drains the battery. Use it for a
  machine that stays on a desk on AC power.
* Closing the lid still turns the internal display off. Screen Sharing keeps
  working and shows a virtual display.
* Clamshell mode (AC power, external display, keyboard or mouse attached) also
  keeps a closed MacBook awake, without `disablesleep`, but stops when the
  display is unplugged.
* Check after macOS upgrades: `pmset -g | grep -E 'SleepDisabled| sleep '`.
