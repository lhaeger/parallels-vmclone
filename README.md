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
Win11  (golden, stopped, snapshot "base")
  │
  ├── vmclone add kundeA  ──►  Win11-kundeA   (running, own VPN session)
  ├── vmclone add kundeB  ──►  Win11-kundeB   (running, own VPN session)
  └── vmclone rm  kundeA  ──►  gone, deltas deleted
```

## How it works

A Parallels linked clone is derived from a *snapshot* of a parent VM. All files
present in the parent at the moment the snapshot was taken remain available to
the clone; everything the clone writes goes into its own delta files.

Unlike reverting snapshots on a single VM, **many linked clones of the same
snapshot can run simultaneously**, and each one gets unique hardware IDs and a
unique MAC address, so Windows and the guest network stack treat it as a
separate machine.

`vmclone` is a thin wrapper around three `prlctl` calls:

| Subcommand | What it runs |
|---|---|
| `add` / `new` | `prlctl clone $GOLDEN --name $PREFIX-<tag> --linked --id <snapshot-uuid>` then `prlctl start` |
| `rm` / `del` | `prlctl stop --kill` then `prlctl delete` |
| `ls` | `prlctl list -a -o name,status`, filtered to `$PREFIX*` |

The snapshot UUID is looked up by snapshot *name* (`$SNAP`) via
`prlctl snapshot-list -j`, so refreshing the golden image does not mean
editing UUIDs by hand.

## Prerequisites

* **macOS** with **Parallels Desktop**. `prlctl` ships with the app
  (`/usr/local/bin/prlctl`); command-line use is documented as a Pro/Business
  Edition feature, so verify it against your licence if you run Standard.
* **`jq`** — `brew install jq`.
* **bash** — the stock macOS `/bin/bash` 3.2 is fine.
* A **golden VM** that is *shut down* and has a *named snapshot* (see below).
  Linked clones cannot be made from Boot Camp VMs, encrypted VMs, or VMs with
  Safe Mode enabled.
* Enough **RAM** for the clones you want to run at once (4–6 GB per Windows 11
  guest), and enough disk for the golden image plus one delta per clone.

## Install

```sh
git clone https://github.com/lhaeger/parallels-vmclone.git
install -m 755 parallels-vmclone/vmclone /usr/local/bin/vmclone
```

## Configuration

Defaults live at the top of the script and can be overridden per invocation:

| Variable | Default | Meaning |
|---|---|---|
| `GOLDEN` | `Win11` | Name of the parent VM |
| `SNAP` | `base` | Name of the snapshot to clone from |
| `PREFIX` | `Win11` | Name prefix for clones (`$PREFIX-<tag>`) |

```sh
GOLDEN=Win11-LTSC SNAP=2026-09 vmclone add kundeA
```

## Usage

```sh
vmclone add kundeA      # clone from the snapshot and boot it   (~2 s + boot)
vmclone add kundeB      # second, independent clone
vmclone ls              # list golden VM and clones with status
vmclone rm  kundeA      # hard power-off and delete, deltas included
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
4. Shut the VM down cleanly. The snapshot must be taken while it is stopped.
5. Take the snapshot:

   ```sh
   prlctl snapshot Win11 --name base
   ```

Optionally `sysprep /generalize /oobe /shutdown` before step 5 — see the
caveats about machine SIDs.

### Keeping it current

```sh
prlctl start Win11          # boot the golden VM
# patch, update tools, shut down cleanly
prlctl snapshot Win11 --name 2026-10
SNAP=2026-10 vmclone add kundeA
```

Old snapshots must stay in place as long as clones made from them exist.

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
