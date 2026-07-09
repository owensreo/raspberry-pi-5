<p align="center">
  <img src="assets/blue-ridge-systems-consulting-logo.svg" alt="Blue Ridge Systems Consulting Logo" width="160" />
</p>

# Raspberry Pi 5 Utility Scripts

Practical Raspberry Pi 5 scripts for small infrastructure boxes, homelab hosts, exit-node VMs, and lightweight Linux maintenance.

This repo is built around a simple idea: keep the Pi useful, predictable, and low-drama. No giant orchestration stack. No mystery knobs. Just scripts that solve real little gremlins before they turn into late-night goblin hunts.

## Included Scripts

| Script | Purpose |
|---|---|
| [`malware-scanning/install-br-lite-malware-scan.sh`](malware-scanning/install-br-lite-malware-scan.sh) | Installs a lightweight YARA + Linux Malware Detect scanner with a systemd timer. |
| [`malware-scanning/uninstall-br-lite-malware-scan.sh`](malware-scanning/uninstall-br-lite-malware-scan.sh) | Removes the lightweight scanner, timer, logs, aliases, and optional leftovers. |
| [`scripts/rename-usb-nics-on-pi.sh`](scripts/rename-usb-nics-on-pi.sh) | Pins USB Ethernet adapters to stable names by MAC address so they survive reboot ordering changes. |

---

# Blue Ridge Lightweight Malware Scanner

`malware-scanning/install-br-lite-malware-scan.sh` installs a low-impact YARA + LMD scanner intended for Raspberry Pi and lightweight Linux hosts.

It is intentionally conservative. It scans high-signal host paths and avoids the heavy stuff that usually causes pain on small systems: containers, VM images, caches, media, archives, and large files.

## What It Does

The installer:

- Installs required packages using `dnf` or `apt-get`.
- Installs or updates Linux Malware Detect.
- Creates custom lightweight YARA rules.
- Writes the scanner to `/usr/local/sbin/br-lite-malware-scan.sh`.
- Creates a systemd service and timer.
- Runs automatically every 6 hours.
- Adds convenience aliases to the invoking user’s `.bashrc` when available.
- Runs an initial scan after install.

The scanner focuses on:

- `/tmp`
- `/var/tmp`
- `/dev/shm`
- `/usr/local/bin`
- `/usr/local/sbin`
- `/opt`
- `/home`

It skips common high-cost or noisy paths, including:

- Container storage
- Docker / Podman / containerd paths
- libvirt and VM paths
- VM disk images such as `.qcow2`, `.vmdk`, `.vdi`, `.iso`, `.img`, `.raw`
- Archives such as `.tar`, `.zip`, `.gz`, `.xz`, `.7z`
- Media files such as `.mp4`, `.mov`, `.mkv`, `.mp3`, `.flac`
- Files 25 MB or larger
- Older files outside the recent scan window

## Install

Clone the repo:

```bash
git clone https://github.com/owensreo/raspberry-pi-5.git
cd raspberry-pi-5
```

Run the installer:

```bash
chmod +x malware-scanning/install-br-lite-malware-scan.sh
sudo ./malware-scanning/install-br-lite-malware-scan.sh
```

Or run directly from GitHub:

```bash
curl -fsSL https://raw.githubusercontent.com/owensreo/raspberry-pi-5/main/malware-scanning/install-br-lite-malware-scan.sh -o install-br-lite-malware-scan.sh
chmod +x install-br-lite-malware-scan.sh
sudo ./install-br-lite-malware-scan.sh
```

## Check Status

```bash
sudo systemctl status br-lite-malware-scan.timer --no-pager
sudo systemctl status br-lite-malware-scan.service --no-pager
```

View the log:

```bash
sudo tail -n 160 /var/log/br-lite-malware-scan.log
```

If the installer added aliases to your shell profile, open a new shell or run `source ~/.bashrc`, then use:

```bash
malware-lite
lmd-reports
```

## Run a Manual Scan

```bash
sudo systemctl start br-lite-malware-scan.service
```

Then check the log:

```bash
sudo tail -n 160 /var/log/br-lite-malware-scan.log
```

## Uninstall

Default uninstall removes the known scanner files, systemd unit, timer, log, aliases, and LMD install paths:

```bash
chmod +x malware-scanning/uninstall-br-lite-malware-scan.sh
sudo ./malware-scanning/uninstall-br-lite-malware-scan.sh
```

Deep cleanup mode searches for old custom YARA/LMD scanner leftovers when service names are unknown:

```bash
sudo ./malware-scanning/uninstall-br-lite-malware-scan.sh --deep
```

Audit first without removing anything:

```bash
sudo ./malware-scanning/uninstall-br-lite-malware-scan.sh --deep --dry-run
```

Remove package-managed YARA too:

```bash
sudo ./malware-scanning/uninstall-br-lite-malware-scan.sh --deep --remove-yara
```

The uninstaller writes a report to:

```text
/root/br-lite-malware-uninstall-YYYY-MM-DD-HHMMSS.txt
```

## Notes and Limits

This is a lightweight host scanner, not a full EDR platform. It is meant to catch suspicious shell droppers, base64 execution patterns, miner strings, and common persistence behavior without beating up a Raspberry Pi.

For servers with high-risk exposure, public workloads, multi-user shell access, or compliance requirements, pair this with normal hardening: updates, firewall rules, SSH key hygiene, least privilege, backups, logging, and a real incident response plan.

---

# Stable USB NIC Names on Raspberry Pi

`rename-usb-nics-on-pi.sh` gives USB Ethernet adapters stable Linux interface names by MAC address.

This is useful when a Raspberry Pi has multiple USB Ethernet dongles and the kernel changes interface names after reboot. For example, a USB NIC that was `eth1` one day might become `eth2` or `enu1c2` after the next boot. If a VM is attached directly to that interface using libvirt/Cockpit/macvtap, the VM can bind to the wrong NIC or fail until manually fixed.

The fix: pin each USB NIC to a stable name such as `usb-lan-a`, `exit-node-a`, or `private-lan`.

## Find MAC Addresses

```bash
ip -br link
```

Example output:

```text
end0    UP  2c:cf:67:aa:bb:cc
enu1c2  UP  a0:ce:c8:11:22:33
eth2    UP  a0:ce:c8:44:55:66
```

## Dry Run

```bash
sudo ./scripts/rename-usb-nics-on-pi.sh --dry-run \
  usb-lan-a=a0:ce:c8:11:22:33 \
  usb-lan-b=a0:ce:c8:44:55:66
```

## Apply

```bash
sudo ./scripts/rename-usb-nics-on-pi.sh \
  usb-lan-a=a0:ce:c8:11:22:33 \
  usb-lan-b=a0:ce:c8:44:55:66
```

Then reboot:

```bash
sudo reboot
```

Verify after reboot:

```bash
ip -br link
```

## Libvirt / Cockpit Note

If a VM uses a direct/macvtap NIC, update the VM NIC source once after the stable names exist.

For example, change the VM NIC source from:

```text
eth2
```

to:

```text
usb-lan-b
```

Then verify:

```bash
sudo virsh domiflist VM_NAME
sudo virsh list --all
```

The `macvtap0`, `macvtap1`, and similar numbers may still change between reboots. That is normal. The important part is that the VM’s `Source` stays pinned to the stable interface name.

---

## Branding

This repository includes Blue Ridge Systems Consulting branding assets. See [`BRANDING.md`](BRANDING.md) for logo and brand usage guidance.

---

## Security Audit

<a href="https://app.aikido.dev/audit-report/external/WUuAYeTGe5MdKOz7TJTyBMJl/request" target="_blank">
    <img src="https://app.aikido.dev/assets/badges/full-light-theme.svg" alt="Aikido Security Audit Report" height="40" />    
</a>

This repository is regularly scanned by **Aikido Security** to help identify potential security issues and support ongoing code quality improvements.
