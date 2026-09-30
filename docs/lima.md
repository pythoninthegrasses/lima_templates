# `fedora-kde.yml` notes

## Lima version

Requires Lima >= 2.1.2. Earlier versions (incl. 2.1.1) SIGTRAP crash opening
the `vz` GUI window on macOS 26+: `VZVirtualMachineView` requires Cocoa/VZ
calls on the process's actual main OS thread, and Go's scheduler had already
migrated off it by the time the hostagent ran. Fixed in
[lima-vm/lima#5036](https://github.com/lima-vm/lima/pull/5036) ("pin main
goroutine to OS thread 0 for VZ GUI"), tracked in
[lima-vm/lima#4743](https://github.com/lima-vm/lima/issues/4743).
`brew upgrade lima` to get a fixed build.

## `limactl start` can time out even on success

`limactl start`'s CLI wrapper waits at most 10 minutes
(`DefaultWatchHostAgentEventsTimeout`) for a "running" event before it exits
nonzero — separate from any probe timeout in the template. A fresh instance's
first boot installs all of KDE via `dnf`, which alone can take ~9 minutes,
so the CLI can time out and exit 1 while the hostagent keeps running fine in
the background. `limactl ls` will still show the instance `Running`.

- Pass `--timeout 20m` on that first `limactl start`.
- If it times out anyway, don't assume it crashed: check
  `limactl ls` and `limactl shell fedora-kde -- systemctl is-active plasmalogin`
  before touching anything.
- `limactl delete` requires `Stopped` state first — `limactl stop` before
  deleting an instance the CLI reported as failed.

## Fedora 44's KDE spin dropped sddm

`@kde-desktop-environment` now pulls in `plasma-login-manager` (`plasmalogin`,
an sddm fork), not `sddm`. There is no `sddm.conf.d`-style drop-in directory;
the live config is `/etc/plasmalogin.conf`, shipped with an `[Autologin]`
section already present but commented out (`#User=`, `#Session=`). Enable
`plasmalogin.service`, not `sddm.service`. Session name is the desktop file
stem in `/usr/share/wayland-sessions/` (`plasma.desktop` → `Session=plasma`).

A provision script that assumes sddm will fail at `systemctl enable sddm`
("Unit sddm.service does not exist") under `set -e`, aborting the whole
script — including the `systemctl isolate graphical.target` call, so the
instance silently stays on `multi-user.target` with cloud-init reporting
`cloud-init-main.service` as a failed unit.

## `vz` display is fixed at 1920x1200, no clipboard

Lima's `vz` driver hardcodes a 1920x1200 virtio-gpu scanout
(`pkg/driver/vz/vm_darwin.go`); there is no YAML field to change it, and no
dynamic-resolution renegotiation as the macOS window resizes. Rendering is
software-only (llvmpipe) — there's no GPU acceleration path for a Linux
guest under `vz`. There's also no clipboard sharing between host and guest
(no SPICE agent attached).

The window itself comes from `Code-Hex/vz`, the library behind Lima's `vz`
driver. On macOS 26+ it draws its own 40pt header bar (close/minimize/
fullscreen buttons, plus a magnifying-glass "Zoom" toggle) on top of an
`NSScrollView`-wrapped VM view. That header bar has no code path that hides
it in true fullscreen — it's permanent on macOS 26/27, not a Lima setting.
The scrollbars some describe seeing are the Zoom toggle's doing
(`hasVerticalScroller`/`hasHorizontalScroller` are `NO` until that toggle is
on); clicking it off should remove them.

## No password on the guest user

Lima's cloud-config sets `lock_passwd: true` — the account has no password
hash (`shadow` shows `lance:!:...`), by design, since SSH-key auth is meant
to be the only way in. Autologin bypasses this at session start, but KDE's
screen locker (`kscreenlocker`) does normal PAM password auth on unlock, and
nothing will satisfy that against a `!` hash. If the lock screen ever
appears, set a password from the host first:

```
limactl shell fedora-kde -- sudo passwd lance
```
