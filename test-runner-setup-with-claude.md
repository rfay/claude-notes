# Setting up a test runner with Claude

Claude Code can do most of a macOS test runner setup over ssh, following
DDEV's
[Buildkite test machine setup](https://docs.ddev.com/en/latest/developers/buildkite-testmachine-setup/)
page. What it can't do is anything that needs `sudo` (no password over a
non-interactive ssh) or a click in the GUI. Plan the session around that split.

## How the work divided

**Claude did, as `testbot` over ssh:**

* Surveyed the machine first: users, Homebrew ownership, installed formulae and
  casks, Docker contexts, Spotlight state, existing Buildkite config.
* `brew install` of the formulae: `bats-core composer git golang jq
  mysql-client@8.0 mkcert netcat p7zip colima lima podman container socktainer`,
  plus `docker docker-buildx docker-compose docker-credential-helper`.
* Casks that don't need sudo: `orbstack`, `rancher`.
* Created and verified each VM provider, ran `alpine` on each, then stopped it:
  * Colima: `colima start vz --cpu 4 --memory 6 --disk 60 --vm-type=vz --mount-type=virtiofs --dns=1.1.1.1`
  * Lima: `limactl create --name=lima-vz ... template:docker`, then the Docker
    context by hand (see below)
  * Podman: `podman machine init --provider applehv --memory 6144`, then a
    `podman-rootless` context
  * Apple Container: `container system start --enable-kernel-install`, then
    socktainer, which creates its own `socktainer` context
* Cleaned obsolete files out of `/usr/local/bin` (it's `admin`-group writable,
  so no sudo needed).
* Wrote the self-hosted runner LaunchAgent
  ([next page](claude-self-hosted-environment.md)).

**The human did:**

* Docker Desktop install (the cask needs sudo for `/usr/local/cli-plugins`).
* OrbStack sign-in and Rancher Desktop configuration (GUI).
* `sudo mdutil -a -i off` (Spotlight off).
* The Apple Container Local Network "Allow" click.
* `sudo kickstart` for Remote Management privileges
  ([Screen Sharing page](macos-screen-sharing.md)).
* Creating the Claude environment key in claude.ai admin settings.

## Things that went wrong or surprised

**Say up front what the machine is for.** Claude followed the whole runner doc
and installed and configured `buildkite-agent@4` before being told the machine
only needed the Docker providers. It reverted cleanly (it had kept a backup of
the config), but a one-line scope statement would have saved the detour.

**Find the user that can work without sudo.** ssh as `rfay` had no passwordless
sudo and didn't own Homebrew. `testbot` owned `/opt/homebrew`, so all package
work went through `testbot@macos-claude`.

**Colima needs a `docker` CLI before it will start.** With Docker Desktop not yet
installed there was no `docker` at all, and Colima refused with "docker not
found". Installing Homebrew's `docker` fixed that.

**Lima doesn't create its Docker context.** `limactl start` only prints the
command. Create it from the instance's socket:

```bash
sock=$(limactl list lima-vz --format 'unix://{{.Dir}}/sock/docker.sock')
docker context create lima-lima-vz --docker "host=$sock"
```

**Colima removes its context on stop.** `docker context ls` after
`colima stop vz` won't show `colima-vz`; `colima start` recreates it.

**Apple Container needs a GUI "Allow" after every upgrade.** Until someone allows
`container-runtime-linux` under Local Network, every published port resets
("Connection reset by peer"). macOS appears to identify unbundled binaries by
their executable's build UUID, so each new `container` build asks again. There
is no CLI or MDM way to pre-approve Local Network access. Mitigations:

* `brew pin container` on runners, and upgrade deliberately.
* After an upgrade, raise the prompt yourself over Screen Sharing with a
  published-port test, rather than letting a test job raise it on an unattended
  screen:

  ```bash
  container system start
  container run -d --name ptest -p 18089:80 nginx:alpine
  curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18089   # want 200
  container rm -f ptest && container system stop
  ```

**Docker Desktop's CLI versus Homebrew's.** With the user-level CLI install,
Docker Desktop puts its tools in `~/.docker/bin`, appends that to PATH in
`~/.bash_profile`, sets `"credsStore": "osxkeychain"` in
`~/.docker/config.json`, and links its plugins into `~/.docker/cli-plugins`.
Consequences:

* Over a non-login ssh command, `docker run` failed with
  `docker-credential-osxkeychain: executable file not found`, because
  `~/.docker/bin` wasn't on PATH.
* To prefer Homebrew's tools: remove the Docker Desktop block from
  `~/.bash_profile`, `brew install docker docker-credential-helper
  docker-compose docker-buildx`, and add
  `"cliPluginsExtraDirs": ["/opt/homebrew/lib/docker/cli-plugins"]` to
  `~/.docker/config.json`.
* `~/.docker/cli-plugins` is searched before `cliPluginsExtraDirs`, so Docker
  Desktop's `compose` and `buildx` still win until those links are removed.
* A stale `/usr/local/lib/docker/cli-plugins` link (from a 2021 Docker Desktop)
  is also searched by any `docker` CLI; remove it with sudo.

**Old machines carry old junk.** `/usr/local/bin` had dangling links to
uninstalled Docker Desktop and Parallels, an Intel `mkcert`, Python 3.9
argcomplete scripts for a Python that no longer existed, and an AppleConnect
(Apple's internal SSO client) leftover. Have Claude classify each entry (broken
link, obsolete binary, still valid) before deleting.

## Verification Claude ran

One check per provider, through Docker contexts, so the current context is
never flipped:

```bash
for c in desktop-linux orbstack rancher-desktop lima-lima-vz podman-rootless socktainer; do
  printf '%s: ' $c; docker --context $c run --rm alpine echo ok 2>&1 | tail -1
done
```

## Tips for the next one

* Give Claude the doc URL and the scope ("Docker providers only, no
  buildkite-agent").
* Make sure Claude can ssh as the user that owns Homebrew.
* Batch the sudo and GUI steps into one list for the human, and do them while
  Claude works on the rest.
* Have Claude verify every step by running something, not by reading config.
* Secrets: Claude diffed a Buildkite `environment` hook against another runner
  and that put tokens into the conversation. Prefer comparing key names
  (`grep -o '^export [A-Z_]*'`) over full files that may hold secrets.
