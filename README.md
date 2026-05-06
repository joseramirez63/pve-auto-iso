# pve-auto-iso

Automatically builds the latest Proxmox VE (PVE) unattended installation ISO using GitHub Actions.  
The workflow checks for new PVE releases daily, and when a new version is detected it creates a customized ISO that installs Proxmox VE fully automatically — no keyboard input required during setup.

---

## What it does

1. **Detects new PVE releases** — a scheduled GitHub Actions job polls the official Proxmox download server every day and compares the latest version against the most recent GitHub Release in this repository.
2. **Downloads the official ISO** — if a newer version exists (or the workflow is triggered manually), it downloads the unmodified upstream ISO from `enterprise.proxmox.com`.
3. **Embeds the answer file** — it uses `proxmox-auto-install-assistant` to inject the `pve-unattended.toml` answer file directly into the ISO, so the installer reads all settings from that file without any user interaction.
4. **Publishes a GitHub Release** — the resulting ISO is uploaded as a release asset with a tag that matches the PVE version.

---

## How it works

```
Schedule / Manual trigger
        │
        ▼
  check-version job
  ┌──────────────────────────────────────┐
  │ 1. Scrape the latest ISO filename    │
  │    from enterprise.proxmox.com/iso/  │
  │ 2. Compare with latest GitHub tag    │
  │ 3. Set need_build=true / false       │
  └──────────────────────────────────────┘
        │  need_build == true
        ▼
  build-iso job
  ┌──────────────────────────────────────┐
  │ 1. Install proxmox-auto-install-     │
  │    assistant + tools                 │
  │ 2. Read pve-unattended.toml          │
  │ 3. Download original PVE ISO         │
  │ 4. Inject answer file into ISO       │
  │ 5. Publish ISO as a GitHub Release   │
  └──────────────────────────────────────┘
```

The core tool is [`proxmox-auto-install-assistant`](https://pve.proxmox.com/wiki/Automated_Installation), which is shipped in the official Proxmox package repository.  
It takes any PVE ISO and a TOML answer file, then produces a new ISO that reads all installer answers from that embedded file.

---

## Repository structure

```
.
├── .github/
│   └── workflows/
│       └── build-pve-iso.yml   # GitHub Actions workflow
├── pve-unattended.toml         # Unattended answer file (edit this to customize)
└── README.md
```

---

## How to create your own unattended answer file

The answer file is `pve-unattended.toml`. It follows the format expected by `proxmox-auto-install-assistant`. Here is a fully annotated example:

```toml
[global]
keyboard = "es"                    # Keyboard layout (e.g. "us", "es", "de")
country = "cl"                     # Two-letter country code (ISO 3166-1 alpha-2)
fqdn = "proxmox.local"             # Fully qualified domain name for the host
mailto = "admin@example.com"       # Email address for system notifications
timezone = "America/Santiago"      # Timezone (see /usr/share/zoneinfo/)
root_password = "YourSecurePass!"  # Root password — change this!

[network]
source = "from-answer"             # "from-answer" = static IP, "from-dhcp" = DHCP
cidr = "192.168.1.10/24"          # Static IP with subnet mask (only for "from-answer")
gateway = "192.168.1.1"            # Default gateway (only for "from-answer")
dns = "1.1.1.1"                   # DNS server (only for "from-answer")

[network.filter]
type = "nic0"                      # Use the first detected NIC

[disk-setup]
filesystem = "ext4"                # "ext4" or "xfs" or "zfs"
disk_list = ["sda"]               # List of target disks (e.g. "sda", "nvme0n1")
lvm.maxroot = 60                   # Maximum size in GB for the root LV
lvm.swapsize = 2                   # Swap size in GB
```

### Key fields

| Field | Description |
|---|---|
| `keyboard` | Keyboard layout code. Common values: `us`, `es`, `de`, `fr`. |
| `country` | Two-letter country code used for locale settings. |
| `fqdn` | Hostname of the Proxmox node (must be a valid FQDN). |
| `timezone` | IANA timezone string (e.g. `Europe/Berlin`, `America/New_York`). |
| `root_password` | Password for the `root` account. Use a strong password. |
| `source` | `from-answer` uses static IP settings below; `from-dhcp` ignores them. |
| `cidr` | Static IP address with CIDR prefix (e.g. `192.168.1.10/24`). |
| `gateway` | Default gateway IP. |
| `dns` | Primary DNS resolver IP. |
| `filesystem` | Root filesystem type: `ext4`, `xfs`, or `zfs`. |
| `disk_list` | Array of disk device names to install onto (e.g. `["sda"]`, `["nvme0n1"]`). |
| `lvm.maxroot` | Size in GB to allocate for the root logical volume. |
| `lvm.swapsize` | Size in GB to allocate for swap. |

> **Official documentation**: https://pve.proxmox.com/wiki/Automated_Installation

---

## Usage

### Run automatically (default)

The workflow runs every day at **18:00 UTC**. No manual action needed — it will only build and publish when a new PVE version is detected.

### Run manually

1. Go to **Actions** → **Build Latest PVE Unattended Installation ISO**.
2. Click **Run workflow**.
3. Wait for the job to finish and download the ISO from the **Releases** page.

### Customize for your environment

1. Fork this repository.
2. Edit `pve-unattended.toml` with your IP address, disk, timezone, password, etc.
3. Commit and push — the next scheduled run (or a manual trigger) will produce an ISO tailored to your setup.
