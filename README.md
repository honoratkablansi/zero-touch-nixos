# zero-touch-nixos

**A NixOS installer ISO that installs itself. No keyboard, no screen, no questions.**

Boot the stick and it joins your Wi‑Fi, wipes the internal disk, installs a preconfigured NixOS system, and powers off. Remove the stick, power on, and SSH in.

> [!WARNING]
> The installer **erases an entire disk without asking**. Only boot it on machines where losing all data is acceptable. On machines with several internal disks, unplug the ones you want to keep (see [Target disk selection](#target-disk-selection)).

## Contents

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [What gets installed](#what-gets-installed)
- [How it works](#how-it-works)
- [Customization](#customization)
- [Security](#security)
- [Troubleshooting](#troubleshooting)

## Requirements

| | |
|---|---|
| Target machine | `x86_64`, UEFI boot (no legacy BIOS) |
| Target disk | At least ~17 GiB (512 MiB ESP + 16 GiB swap + root); 32 GiB or more recommended |
| Network | WPA‑PSK Wi‑Fi in range, with internet access |
| USB stick | 1 GB or larger |
| Build machine | Nix (flakes are enabled on the command line below) |

## Quick start

### 1. Set your credentials

Edit the `let` block at the top of [`configuration.nix`](configuration.nix):

```nix
let
  ssid          = "YourWifiName";
  psk           = "YourWifiPassword";
  loginPassword = "change-me";
  sshKey        = "ssh-ed25519 AAAA... you@host";
```

| Variable | Used for |
|---|---|
| `ssid`, `psk` | Wi‑Fi on both the live ISO and the installed system |
| `loginPassword` | `root` password on the live ISO **and** the initial password of the user `sv` on the installed system |
| `sshKey` | Authorized for `root` on both systems and for `sv` on the installed system |

> [!CAUTION]
> These values end up in plain text in the ISO. Read [Security](#security) before you commit or share anything.

### 2. Build the ISO

```bash
nix build --extra-experimental-features "nix-command flakes" \
  .#nixosConfigurations.unattended-iso.config.system.build.isoImage
```

The image lands in `result/iso/`.

### 3. Write it to a USB stick

Find the stick with `lsblk` first. Writing to the wrong device destroys it.

```bash
sudo dd if=result/iso/*.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

### 4. Boot and wait

1. Plug the stick into the target machine and boot from it (you may need to change the boot order once).
2. Progress shows on screen as `[1/7]` … `[7/7]`. Step 6 downloads packages and takes the longest.
3. The machine powers off when done. **Remove the stick**, then power on.
4. Log in:

   ```bash
   ssh sv@<machine-ip>
   ```

   Change the initial password with `passwd`.

### Watching remotely

The live ISO runs SSH, so you can follow the install from another machine:

```bash
ssh root@<machine-ip>
journalctl -u unattended-installer -f
```

## What gets installed

**Disk layout** (GPT):

| # | Size | Filesystem | Label |
|---|---|---|---|
| 1 | 512 MiB | FAT32 (EFI System Partition) | `BOOT` |
| 2 | 16 GiB | swap | `swap` |
| 3 | rest of disk | ext4 | `nixos` |

**System configuration** (on top of what `nixos-generate-config` produces):

- `systemd-boot` bootloader
- Wi‑Fi through `wpa_supplicant` with the same network as the installer; NetworkManager disabled
- Redistributable firmware enabled (covers most Wi‑Fi chips)
- OpenSSH: `root` by key only; `sv` by key or password
- User `sv` in `wheel`, with passwordless `sudo`
- `root` has no password (key login only)

The generated `/etc/nixos/configuration.nix` imports an extra module, `/etc/nixos/extra.nix`, which holds the settings above. Edit either file and run `nixos-rebuild switch` to change the installed system.

## How it works

The ISO is the minimal NixOS installer plus a `systemd` oneshot service, `unattended-installer`, that runs at boot:

| Step | Action |
|---|---|
| 1 | Wait until `cache.nixos.org` is reachable |
| 2 | Pick the target disk (see below) |
| 3 | Wipe and partition it |
| 4 | Format the partitions |
| 5 | Mount them, run `nixos-generate-config`, add `extra.nix` |
| 6 | Run `nixos-install` |
| 7 | Power off |

### Target disk selection

The installer picks the **first non‑removable disk** that isn't the USB stick it booted from. Loop, optical, zram and RAM devices are skipped. On a machine with several internal disks, there's no guarantee which one comes first, so disconnect any disk you want to keep.

### Re-run protection

- The root partition is labeled `nixos-installing` during installation and renamed to `nixos` only after `nixos-install` succeeds.
- On boot, if the target disk already has a partition labeled `nixos`, the installer does nothing.
- So an interrupted install is retried on the next boot, while a finished one is left alone.
- The machine **powers off instead of rebooting**, so a machine that boots from USB first can't loop into another wipe.

> [!NOTE]
> The check only looks at the label. Any disk with a partition labeled `nixos` (including a manual install that followed the NixOS manual) is skipped.

## Customization

Everything lives in `configuration.nix`:

| To change | Edit |
|---|---|
| Username | `users.users.sv` inside `targetModule` |
| Packages and services on the installed system | `targetModule` |
| Swap size | The `mkpart swap` and `mkpart root` lines in step 3 (both boundaries must move together) |
| nixpkgs channel | `inputs.nixpkgs.url` in `flake.nix` (default: `nixos-unstable`) |

## Security

- **Don't commit real credentials to a public repository.** `configuration.nix` holds your Wi‑Fi password, a login password and your SSH key. Keep real values in a private fork, or load them from a git‑ignored file. If you already pushed them, change the Wi‑Fi password; deleting the commit doesn't remove it from clones or caches.
- **Treat the ISO as a secret.** The credentials are stored in plain text in its Nix store.
- **The live ISO allows root login over SSH with a password.** Anyone on the network can try it while the installer runs.
- **The installed system allows SSH password login for `sv`**, starting with `loginPassword`. Change it after the first login, or set `PasswordAuthentication = false` in `targetModule`.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Stuck on `Waiting for network...` | Wrong SSID or password, network out of range, or the Wi‑Fi chip needs firmware the ISO doesn't include. |
| `already contains a NixOS install; skipping` | The target disk has a partition labeled `nixos`. This is intended. Wipe the disk manually to reinstall. |
| `ERROR: No target internal disk discovered!` | No non‑removable disk found other than the USB stick. Some NVMe‑over‑USB or SD setups are reported as removable. |
| The wrong disk was wiped | The installer takes the first non‑removable disk. Unplug the disks you want to keep. |
| `experimental Nix feature 'flakes' is disabled` | Pass `--extra-experimental-features "nix-command flakes"` when building, as in the command above. |
| `You can not use networking.networkmanager with networking.wireless` | The installed‑system module already forces NetworkManager off with `lib.mkForce false`. If you see this, check that your own changes don't re‑enable it. |
| Evaluation error during step 6 | Read the lines under `Failed assertions:` on screen, or run `journalctl -u unattended-installer` over SSH on the live system. |

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
