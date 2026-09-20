# parallels-vmclone

`vmclone` creates and destroys throwaway Windows 11 VMs on macOS with a single
command, using Parallels **linked clones** off one patched golden image.

## Use case

Consulting work often means being VPN'd into several customers **at the same
time**, each with its own VPN client, its own jump hosts and its own set of
tools that must not see the other customers' traffic or credentials. A single
shared Windows VM does not cut it, and maintaining one full VM per customer
means patching and re-tooling all of them.

This splits the problem in two:

* **One golden VM** — Windows 11, fully patched, all VPN clients and tools
  installed. It is maintained, but never used for actual work.
* **N disposable clones** — created on demand, one per customer session,
  thrown away afterwards. Each is an independent machine with its own
  hardware IDs and MAC address, so several can run and VPN concurrently.

Creating or destroying a clone must be one command and must not cost a
coffee break. Linked clones make that possible: a clone shares the golden
image's disk blocks read-only and writes only its own deltas, so creation
takes a couple of seconds and almost no disk space.

```
Win11  (golden, stopped, snapshotted automatically when it changes)
  │
  ├── vmclone add customerA  ──►  Win11-customerA   (running, own VPN session)
  ├── vmclone add customerB  ──►  Win11-customerB   (running, own VPN session)
  └── vmclone rm  customerA  ──►  gone, deltas deleted
```

## How it works

A Parallels linked clone is derived from a *snapshot* of a parent VM. All files
present in the parent at the moment the snapshot was taken remain available to
the clone; everything the clone writes goes into its own delta files.

Unlike reverting snapshots on a single VM, **many linked clones of the same
snapshot can run simultaneously**, and each one gets unique hardware IDs and a
unique MAC address, so Windows and the guest network stack treat it as a
separate machine.

`vmclone` is a thin wrapper around a handful of `prlctl` calls:

| Subcommand | What it runs |
|---|---|
| `add` / `new` | resolves a snapshot (see below), then `prlctl clone $GOLDEN --name $PREFIX-<tag> --linked --id <snapshot-uuid>`, `prlctl set $PREFIX-<tag> $VMSET <your options>` if any were given, and `prlctl start` |
| `rm` / `del` | `prlctl stop --kill` then `prlctl delete` |
| `ls` | `prlctl list -a -o name,status`, filtered to `$PREFIX*` |

### Snapshots are handled for you

You do not create or track snapshots by hand. On every `add`, `vmclone` checks
whether the golden VM's current snapshot still holds its present state:

* **It does** — the clone is made from that snapshot, whoever created it. No
  new snapshot, nothing to clean up later.
* **It does not** (you booted the golden to patch it) — `vmclone` takes a fresh
  snapshot named `vmclone-<timestamp>` and clones from that.

The check is exact rather than a guess about file timestamps. Parallels records
the VM's cumulative uptime in its config, and every snapshot stores a copy of
that config as it stood when the snapshot was taken. Equal uptime means the
golden VM has not run since — so the snapshot still matches the disk.

### Old snapshots are cleaned up

After every `add` and `rm`, `vmclone` deletes the snapshots it created once
nothing is linked to them any more. `prlctl snapshot-list -j` reports the
dependent clones of each snapshot, so this is read from Parallels rather than
inferred.

Two rules keep it safe:

* **Only snapshots named `vmclone-*` are ever deleted.** Anything you named
  yourself is left alone, even when no clone uses it.
* **The current snapshot is always kept**, because it is what the next `add`
  would reuse.

Parallels refuses to delete a snapshot while *any* linked clone of the golden
VM is running, even a clone of some other snapshot. When that happens,
`vmclone` says so and leaves the snapshots in place; the next `add` or `rm`
with everything shut down sweeps them up.

## Prerequisites

* **macOS** with **Parallels Desktop**. `prlctl` ships with the app
  (`/usr/local/bin/prlctl`); command-line use is documented as a Pro/Business
  Edition feature, so verify it against your licence if you run Standard.
* **`jq`** — `brew install jq`.
* **bash** — the stock macOS `/bin/bash` 3.2 is fine.
* A **golden VM** that is *shut down*. It does not need a snapshot — `vmclone`
  takes one when it needs one. Parallels cannot clone a running VM at all, so
  `add` refuses until the golden VM is off. Linked clones cannot be made from
  Boot Camp VMs, encrypted VMs, or VMs with Safe Mode enabled.
* Enough **RAM** for the clones you want to run at once (4–6 GB per Windows 11
  guest), and enough disk for the golden image plus one delta per clone.

## Install

`vmclone` is a single self-contained script with no build step. Get a copy and put
it somewhere on your `PATH`, executable. How you do that is up to you.

Copy it into a system-wide directory:

```sh
git clone https://github.com/lhaeger/parallels-vmclone.git
install -m 755 parallels-vmclone/vmclone /usr/local/bin/vmclone
```

Or symlink it into a personal one, so `git pull` updates the installed command:

```sh
git clone https://github.com/lhaeger/parallels-vmclone.git ~/src/parallels-vmclone
mkdir -p ~/.local/bin
ln -s ~/src/parallels-vmclone/vmclone ~/.local/bin/vmclone
```

`~/.local/bin` is not on the default macOS `PATH`; add it in your shell profile if it
is not there already.

## Configuration

Defaults live at the top of the script and can be overridden per invocation:

| Variable | Default | Meaning |
|---|---|---|
| `GOLDEN` | `Win11` | Name of the parent VM |
| `PREFIX` | `Win11` | Name prefix for clones (`$PREFIX-<tag>`) |
| `SNAP` | *(unset)* | Pin to one existing snapshot by name. Unset means detect-or-create |
| `AUTO_PREFIX` | `vmclone-` | Name prefix for snapshots `vmclone` creates, and the only ones it deletes |
| `VMSET` | `--startup-view fullscreen --fullscreen-scale-view-mode keep-ratio` | `prlctl set` options applied to every new clone, before any given on the command line |

```sh
GOLDEN=Win11-LTSC vmclone add customerA      # snapshot handled automatically
SNAP=24H2 vmclone add customerA              # clone from that snapshot, whatever the golden looks like now
```

Setting `SNAP` turns the automation off for that call: the named snapshot is
used as-is, none is created, and — unless it happens to be named `vmclone-*` —
it is never garbage-collected.

### Configuring the clone

Anything after the tag is handed to `prlctl set` verbatim, in one call, while
the clone is stopped and before it boots. `vmclone` does not know or check what
the options mean: whatever `prlctl set` accepts on your Parallels version works,
including options added after this script was written.

```sh
vmclone add customerA --memsize 16384 --cpus 4
vmclone add customerB --startup-view fullscreen --fullscreen-scale-view-mode auto
vmclone add customerC -- --memsize 8192     # a lone -- is accepted and dropped
```

`prlctl set <category> --help` is the reference — note that the *category* name
goes where the VM name normally goes:

```sh
prlctl set --help               # lists the categories
prlctl set startup --help       # --startup-view, --autostart, --on-window-close, ...
prlctl set fullscreen --help    # --fullscreen-use-all-displays, ...
prlctl set memory --help        # --memsize
```

`VMSET` holds the options you always want, so they need not be retyped. Out of the
box it brings every clone up full screen at the guest's own resolution:

```sh
VMSET="--startup-view fullscreen --fullscreen-scale-view-mode keep-ratio"
```

Setting it replaces that default rather than adding to it, so repeat the parts you
want to keep:

```sh
export VMSET="--startup-view fullscreen --cpus 4"
vmclone add customerD --memsize 16384       # gets all three
vmclone add customerE --cpus 8              # --cpus 8 wins over VMSET's 4
VMSET= vmclone add customerF                # empty: no options at all
```

Command-line options are appended after `$VMSET`, and `prlctl` applies repeated
options last-to-win, so the command line overrides the default.

## Usage

```sh
vmclone add customerA      # snapshot if needed, clone, boot   (~2 s + boot)
vmclone add customerB      # second, independent clone
vmclone add customerC --memsize 16384   # any prlctl set option, applied before boot
vmclone ls                 # list golden VM and clones with status
vmclone rm  customerA      # hard power-off and delete, deltas included
```

`rm` uses `prlctl stop --kill`, i.e. a hard power-off — clones are disposable
by design, so nothing is flushed or shut down gracefully. Do not point it at a
VM holding work you want to keep.

## Preparing the golden image

1. Install Windows 11 in Parallels, then Parallels Tools.
2. Patch it fully, install the VPN clients, RDP/SSH tooling and whatever else
   the customer sessions need.
3. Consider disabling automatic Windows Update **for the clones**, so a fresh
   clone does not spend its first ten minutes patching a disk it is about to
   throw away. Patching happens in the golden image instead.
4. Shut the VM down cleanly. That is all — the first `vmclone add` takes the
   snapshot it needs.

Optionally `sysprep /generalize /oobe /shutdown` before the final shutdown —
see the caveats about machine SIDs.

### Keeping it current

```sh
prlctl start Win11             # boot the golden VM
# patch, update tools, shut down cleanly
vmclone add customerA          # notices the golden VM ran, snapshots it, clones
```

Snapshots made from an earlier golden state stay in place for as long as clones
still use them, and are deleted once those clones are gone.

## Caveats

* **The parent is load-bearing.** Deleting or moving the golden VM, or deleting
  the snapshot a clone was made from, breaks that clone. Parallels' *Unlink
  Clone* turns a clone into a standalone VM if you need to keep one.
* **Windows activation.** Each clone gets fresh SMBIOS identifiers, so a
  hardware-bound digital licence may show the clone as unactivated. A
  retail/MAK key in the golden image, or a customer-provided KMS, avoids the
  nagging.
* **Duplicate hostname and machine SID.** All clones inherit the golden image's
  computer name and SID. Fine for "VPN in, RDP onward". Not fine if two clones
  join the same AD domain, or if a customer's NAC objects to duplicate names.
  The script contains a commented-out `Rename-Computer` line (costs one
  reboot) for the hostname; genuine SID uniqueness requires a sysprepped
  golden image, at the price of an OOBE pass on first boot.
* **`ls` also lists the golden VM**, because the default `PREFIX` is a prefix
  of `GOLDEN`. Set `PREFIX=work` if you would rather see only clones.
* **Disk growth.** Deltas grow with everything the clone writes, Windows
  Update included. Disposing of clones regularly is what keeps this cheap.
* **Cleanup needs everything shut down.** Parallels will not delete a snapshot
  while any linked clone of the golden VM is running. If you always keep a
  clone up, unused snapshots accumulate until you shut them all down.
* **Deleting a snapshot merges its delta** into the next one in the chain. On a
  large golden image that is minutes of disk I/O, and it happens inside the
  `add` or `rm` that triggers the cleanup.
* **Change detection keys on uptime.** Booting the golden VM marks it changed,
  which is the point. Editing its config without booting — RAM, devices — does
  not, so an existing snapshot is reused. Take a snapshot yourself, or set
  `SNAP`, if such a change must reach the clones.
* **Do not name your own snapshots `vmclone-*`.** That prefix marks a snapshot
  as disposable, and cleanup will delete it once no clone depends on it.
* **Clone options are applied while the clone is stopped**, between the clone
  and the boot. That is what lets `--memsize` and friends take effect without a
  restart, but options that need a *running* VM — `--device-connect`,
  `--device-disconnect` — will fail there. Run those with `prlctl` yourself once
  the clone is up.
* **A failed `prlctl set` deletes the clone.** Apart from `rm`, this is the only
  place `vmclone` deletes a VM: if any forwarded option is rejected, the
  just-created clone is removed and `add` exits non-zero, so you never keep a
  half-configured machine. The snapshot the `add` may have taken does stay
  behind — it is the current one, and the next `add` reuses it. A failed
  `prlctl start` is different: that clone is configured and kept, so you can
  retry the start.
* **`VMSET` is split on whitespace, with no quoting.** Values containing spaces
  cannot be expressed there; pass those on the command line, where your shell
  quotes them. `~` is not expanded either, so write
  `export VMSET="--shf-host-add p --path $HOME/docs"` in double quotes if you
  need a home-relative path.
* **Forwarded options are not validated.** A typo is caught by `prlctl`, not by
  `vmclone`, and only after the clone exists — costing a clone-and-delete round
  trip of a few seconds rather than an instant rejection. `--name` is the one
  exception: it is refused up front, because renaming a clone would hide it from
  `rm`, `ls` and snapshot cleanup while it still pins its snapshot.
* **Screen resolution is not a `prlctl` setting.** No `prlctl set` category has
  an option for the guest's display resolution — the mode is negotiated at
  runtime between Parallels Tools and the Parallels window, and never stored in
  the VM config, so nothing `vmclone` forwards can pin it. (`--videosize` is
  video *memory*; `--high-resolution` and friends are HiDPI scaling switches.)
  What you can set is how the clone presents itself, which is what the default
  `VMSET` does: `--startup-view fullscreen` brings it up full screen, and
  `--fullscreen-scale-view-mode keep-ratio` leaves the guest's own resolution
  alone and scales the picture to the display, preserving the aspect ratio. Use
  `auto` instead if you would rather the guest resize itself to match the Mac
  you are sitting at — that needs Parallels Tools, which the setup above already
  requires. An exact pixel size has to be set inside the guest, e.g. through
  `prlctl exec`.

## Alternatives

* **VMware Fusion** (free for all users since 2025) does the same thing:
  `vmrun clone src.vmx dst.vmx linked -snapshot=base -cloneName=x`, plus
  `vmrun start/stop` and `deleteVM`.
* **Vagrant** with the `vagrant-parallels` provider and
  `parallels.linked_clone = true` gives `vagrant up` / `vagrant destroy`
  semantics and per-customer config in files, at the cost of packaging
  Windows 11 as a box and provisioning over WinRM.
* **UTM/QEMU** with qcow2 backing files is cheapest and fiddliest; on Apple
  Silicon you are on Windows 11 ARM either way.

## References

* [prlctl clone reference](https://docs.parallels.com/parallels-desktop-developers-guide/command-line-interface-utility/manage-virtual-machines-from-cli/general-virtual-machine-management/clone-a-virtual-machine)
* [Parallels KB 122669 — how to work with Linked Clones](https://kb.parallels.com/en/122669)

## License

MIT — see [LICENSE](LICENSE).
