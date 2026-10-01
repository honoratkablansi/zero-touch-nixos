
# zero-touch-nixos

**A NixOS installer that installs itself. No keyboard, no screen, no questions.**

The ISO connects to Wi‑Fi, wipes the internal disk, installs a fully configured NixOS system, and shuts down when done. Unplug the stick, power on, and you're ready to SSH in.

> ⚠️ **WARNING:** This installer erases an entire disk without confirmation. Only use it on machines where total data loss is acceptable.

---

## Overview

When the ISO boots, a `systemd` service called `unattended-installer` executes seven steps. Progress displays on the physical console and in the journal:

| Step | Action |
|------|--------|
| 1 | Waits for Wi‑Fi and internet access (until `cache.nixos.org` is reachable) |
| 2 | Detects the target disk: first non‑removable disk that isn't the USB stick |
| 3 | Partitions the disk (GPT, UEFI) |
| 4 | Formats the partitions |
| 5 | Mounts them and runs `nixos-generate-config` |
| 6 | Runs `nixos-install` (downloads packages — this takes the longest) |
| 7 | Powers the machine off |

The machine **shuts down instead of rebooting** intentionally. A machine that boots USB-first would otherwise loop back into another wipe cycle.

### Disk Layout After Installation

| Partition | Size | Purpose |
|-----------|------|---------|
| 1 | 512 MiB | EFI System Partition (FAT32, label `BOOT`) |
| 2 | 16 GiB | Swap (label `swap`) |
| 3 | Remaining space | Root filesystem (ext4, label `nixos`) |

### Installed System Configuration

- Bootloader: `systemd-boot` (UEFI mode)
- Networking: `wpa_supplicant` (inherits the installer's Wi‑Fi network)
- SSH: OpenSSH enabled with your public key authorized for both `root` and the regular user
- User account: Standard user in the `wheel` group with passwordless `sudo`
- Firmware: Redistributable firmware enabled (covers most Wi‑Fi chips)

### Safety Mechanism

The root partition is labeled `nixos-installing` during installation and renamed to `nixos` only after successful completion. If the installer detects an existing NixOS installation, it skips the entire process and exits. A failed or interrupted install won't be mistaken for a completed one.

---

## Requirements

| Component | Specification |
|-----------|---------------|
| Architecture | `x86_64` |
| Boot Mode | UEFI only (no legacy BIOS) |
| Network | WPA‑PSK Wi‑Fi within range |
| Internet | Working connection on that network |
| Media | USB stick (1 GB minimum recommended) |
| Builder Machine | Nix installed with flakes enabled |

---

## Quick Start

### 1. Configure Your Credentials

Edit the `let` block at the top of `configuration.nix`:

```nix
let
  ssid          = "YourWifiName";
  psk           = "YourWifiPassword";
  loginPassword = "temporary-root-password-for-live-ISO";
  sshKey        = "ssh-ed25519 AAAA... your@email.com";
  ...
```

Also update the `targetModule` section to set your desired username and initial password for the installed system.

> 🔒 **Security Note:** This file contains sensitive credentials. Keep it in a private repository or exclude it via `.gitignore`.

### 2. Build the ISO

```bash
nix build --extra-experimental-features "nix-command flakes" \
  .#nixosConfigurations.unattended-iso.config.system.build.isoImage
```

Output location: `result/iso/`

### 3. Write to USB Stick

**⚠️ Double-check the device path!** Writing to the wrong disk destroys its contents:

```bash
sudo dd if=result/iso/*.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Verify with `lsblk` before running this command.

### 4. Install and Power Off

1. Insert the USB stick into the target machine
2. Boot from USB (may require adjusting boot order once)
3. Watch the progress counter `[1/7]` through `[7/7]`
4. When the machine shuts down, **remove the USB stick**, then power on

### Remote Monitoring

The live ISO runs SSH, allowing you to monitor the installation from another machine:

```bash
ssh root@<machine-ip-address>
journalctl -u unattended-installer -f
```

---

## Security notes

- `configuration.nix` contains your Wi-Fi password, a login password and your SSH key. **Don't push it to a public repository with real values.** Keep real credentials in a private repo, or move them to a file that is listed in `.gitignore`.
- The Wi-Fi password and the passwords are stored in the Nix store of the ISO, so treat the ISO image itself as sensitive.
- The live ISO allows root login over SSH with a password. Anyone on your network can try to connect while it's running.
- The installed system uses `initialPassword` for the normal user. Change it after the first login with `passwd`.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `experimental Nix feature 'flakes' is disabled` | Flakes must be enabled in the live ISO. `nix.settings.experimental-features` is already set in `configuration.nix`. |
| `You can not use networking.networkmanager with networking.wireless.networks` | The generated config enables NetworkManager. The installed-system module forces it off with `lib.mkForce false`. |
| Installer prints `already contains a NixOS install; skipping` | The disk already has a finished install, by design. Wipe the disk manually to reinstall. |
| Stuck on `Waiting for network...` | Wrong SSID or password, network out of range, or the Wi-Fi chip needs firmware the ISO doesn't include. |
| Wrong disk was wiped | Detection picks the first non-removable disk. On a machine with several disks, unplug the ones you want to keep. |

For evaluation errors, the last lines under `Failed assertions:` on the screen name the problem. To see the full log later, run `journalctl -u unattended-installer` over SSH on the live system.

---


## License

This project is licensed under the [Apache License, Version 2.0](LICENSE).
