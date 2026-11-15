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

This turns off all sleep, including lid-close sleep, and survives reboots.
Undo it with `sudo pmset -a disablesleep 0`.

Also keep the idle settings that already keep it awake with the lid open:

```bash
sudo pmset -c sleep 0 displaysleep 60 disksleep 10
```

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
