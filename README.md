# disable-acs

Keep the kernel `pci=disable_acs_redir` parameter aligned with the NVIDIA GPUs that
are directly attached to PCIe root ports, for **GRUB** and **systemd-boot** systems.

On multi-GPU Linux boxes, leaving ACS redirection enabled on a GPU's root port puts
that GPU in its own IOMMU group, which breaks peer-to-peer traffic (NVLink over PCIe,
GPUDirect, and some multi-GPU workloads). The fix is to disable ACS redirection on
exactly the root ports the GPUs sit behind. Doing that by hand means finding those
BDFs, escaping the list correctly for the bootloader, and redoing it whenever the
topology changes. This script automates the whole loop.

## Bootloader support

| Bootloader | What the script reads and writes |
| --- | --- |
| GRUB | `/etc/default/grub`, then regenerates `/boot/grub/grub.cfg` with `grub-mkconfig` |
| systemd-boot | `options` lines of `/boot/loader/entries/*.conf`, plus `/etc/kernel/cmdline` |

> **Unified Kernel Images are not supported.** With a UKI the command line is baked
> into the `.efi` image at build time, so there is nothing to patch. The script
> detects this (`layout=uki` in `/etc/kernel/install.conf`, a boot entry whose `linux`
> key points at a `.efi`, or any `/boot/EFI/Linux/*.efi`) and stops with instructions
> instead of writing something that would never take effect. `--verify` still works on
> such systems, since it inspects the running kernel.

Anything else — rEFInd, syslinux, a bootloader generating its own menu, or a
systemd-boot layout outside the standard locations — is out of scope, and detection
failure is a hard error rather than a guess.

## Requirements

- **GRUB**: `/etc/default/grub`, `/boot/grub/grub.cfg`, and `grub-mkconfig`; or
- **systemd-boot**: `/boot/loader/entries/*.conf`, optionally with `/etc/kernel/cmdline`
- Root: reading complete PCIe ACS capabilities needs raw config space access
- Bash, `lspci` (`pciutils`), coreutils, `sed`
- NVIDIA display/3D GPUs attached directly to distinct PCIe root ports.
  Switches and other bridge topologies are rejected rather than handled silently.

The bootloader is auto-detected from those files: GRUB requires both
`/etc/default/grub` and `/boot/grub/grub.cfg` to exist, systemd-boot requires the
entries directory or `/etc/kernel/cmdline`. If both are present the script refuses to
guess and asks for `--bootloader`. Detection runs in preview and `--apply` modes;
`--verify` needs neither configuration.

Exit status is `0` on success (including "already up to date") and `1` on any error,
including a failed `--verify` and any failed write that was rolled back.

## Usage

```sh
sudo ./disable-acs                              # show detected GPUs, live ACS state, proposed change
sudo ./disable-acs --apply                      # back up the boot configuration and update it (no reboot)
sudo ./disable-acs --verify                     # check the running kernel and live ACS registers
sudo ./disable-acs --bootloader systemd-boot    # skip auto-detection
sudo ./disable-acs --expect-gpus 2              # require at least N GPUs (default: 4)
```

Run with no flags first: it is a dry run that prints what it would change. Nothing is
written until `--apply` is passed. After rebooting, `--verify` confirms that both the
active kernel command line and the ACS registers on every detected root port are
correct, exiting non-zero if any port still redirects. `--verify` (also accepted as
`--status`) is bootloader independent: it reads `/proc/cmdline` and PCI config space
only.

## How it works

1. Enumerate NVIDIA display/3D devices (`0x10de`, class `0x03*`) in sysfs and resolve
   each one's direct parent PCI device from the topology symlinks.
2. Reject anything that is not a direct PCIe Root Port, and reject ports that lack
   ACS request/completion redirect capabilities (`lspci -vv`).
3. Refuse to continue if two GPUs share a root port, or if fewer GPUs are found than
   expected — a half-populated topology must not be declared healthy.
4. Read the live `ACSCtl:` bits for each port and report whether redirection is still
   enabled.
5. Build `pci=disable_acs_redir=<root-port>;<root-port>;...` with stable ordering,
   replacing any previous instance of the parameter while keeping all other boot
   arguments. The BDFs are the root ports, which is what the kernel parameter expects.

The parameter is escaped per bootloader, because they do not agree: GRUB needs the
semicolons escaped (`0000:00:01.0\;0000:00:03.0`), while systemd-boot entries and
`/etc/kernel/cmdline` take them verbatim (`0000:00:01.0;0000:00:03.0`).

**GRUB** (`--apply`):

- backs up `/etc/default/grub` and `/boot/grub/grub.cfg` with a timestamped suffix;
- writes the new defaults through a temp file and an atomic `mv`, preserving metadata;
- validates a full `grub-mkconfig` preview before touching the real `grub.cfg`;
- restores both backups and exits non-zero if either the preview or the real run fails.

It deliberately refuses to edit nonstandard configurations: more than one
`GRUB_CMDLINE_LINUX_DEFAULT` assignment, a non-single-line double-quoted value, or an
ACS parameter in `GRUB_CMDLINE_LINUX` are all hard errors so you can resolve the
conflict manually.

**systemd-boot** (`--apply`):

- updates the `options` line of every entry that has a `linux` line, adding the line
  if the entry has none, and reporting entries it skipped;
- updates `/etc/kernel/cmdline` as well, so `kernel-install` does not reintroduce the
  old command line the next time a kernel is installed. Both must agree: entries alone
  would be overwritten on the next kernel update, and the cmdline file alone would not
  affect the running configuration;
- needs no regeneration step — systemd-boot reads these files directly;
- backs up every file it changes, writes it atomically, re-reads it to confirm the
  parameter landed (and that no GRUB-style escaping leaked in), and restores all
  backups otherwise;
- refuses a multiline `/etc/kernel/cmdline`, and warns if the entries directory is on
  the root filesystem — writing there while your ESP is unmounted changes files the
  firmware never reads.

## Environment overrides

Useful for testing without a real boot configuration or live topology:

| Variable | Default | Purpose |
| --- | --- | --- |
| `ACS_SYSFS` | `/sys/bus/pci/devices` | sysfs PCI device directory |
| `ACS_CMDLINE_FILE` | `/proc/cmdline` | running kernel command line |
| `ACS_GRUB_DEFAULT` | `/etc/default/grub` | GRUB defaults file |
| `ACS_GRUB_CFG` | `/boot/grub/grub.cfg` | generated GRUB configuration |
| `ACS_LOADER_DIR` | `/boot/loader` | systemd-boot loader directory |
| `ACS_LOADER_ENTRIES_DIR` | `/boot/loader/entries` | boot entry directory |
| `ACS_KERNEL_CMDLINE` | `/etc/kernel/cmdline` | kernel-install command line |
| `ACS_KERNEL_INSTALL_CONF` | `/etc/kernel/install.conf` | used to detect `layout=uki` |
| `ACS_UKI_GLOB` | `/boot/EFI/Linux/*.efi` | Unified Kernel Images to detect |

## Warning

Disabling ACS redirection weakens PCIe/IOMMU device isolation between the affected
root ports. Traffic from a device behind such a port can reach other devices that
would otherwise be isolated from it. Only use this on a machine where you control
what is plugged in, and prefer the narrowest possible set of ports — which is what
this script does by deriving them from the GPU topology instead of guessing.
