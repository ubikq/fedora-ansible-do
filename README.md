# Fedora Workstation Setup — Ansible Playbook

## About This Project

This Ansible playbook automates the post-install configuration of Fedora Workstation, systematically transforming a stock setup into a fully provisioned environment. To keep the project clean and maintainable, all application lists are centralized in a single file (`vars/app_catalog.yml`), while the logic is organized into dedicated task and variable files. 

### Key Capabilities

- **System Maintenance & Upgrades**: Automates the standard system update cycle (`dnf upgrade`), manages official Fedora repositories, and handles package cleanup tasks.
    
- **Third-Party Repository Integration**: Safely provisions third-party repositories—specifically enabling **RPM Fusion** (`free` and `nonfree`) to seamlessly unlock restricted media codecs and proprietary hardware drivers (such as Nvidia graphics drivers).
    
- **Core Software & Tool Deployment**: Orchestrates the bulk installation of applications, system utilities, and developer dependencies via native package management (`dnf`), eliminating manual setup.
    
- **Flatpak Integration**: Configures Flatpak functionality and enables the **Flathub repository**, laying down the framework to install sandbox-isolated desktop applications.
    
- **User Environment Personalization**: Deploys localized dotfiles, system preferences, shell configurations, and workflow tools to match a specific user's desktop routine.

- **Configuration Backup & Restore**: Backs up and restores application configuration to and from a network share, covering both Flatpak and RPM-installed applications. Maintains dated archives, a rolling latest snapshot, and optional monthly snapshots with configurable retention. See the [Backup & Restore](#backup--restore) section for details.

### Planned Roadmap

The following changes are envisioned:

- Incorporate the fantastic _niri_ scrolling tiling compositor with the _DankMaterialShell_ desktop shell

---

## ⚠️ Disclaimer

**Use at your own risk.** This Ansible playbook is designed to configure my personal Fedora 44 environment. Because every system configuration, hardware setup, and user need is different, running this playbook may cause data loss, system instability, or unexpected behavior on your machine. 

Before running this playbook:

- **Backup your data**: Ensure you have a complete, external backup of all critical files.
- **Review the tasks**: Read through the YAML files to understand exactly what changes will be made to your system.
- **Test safely**: If possible, test the playbook inside a virtual machine (VM) first.

The author accepts no liability or responsibility for any damage, data loss, or system breakage caused by using this repository.

### 📦 Third-Party Repositories (RPM Fusion)

This playbook automatically enables the [RPM Fusion](https://rpmfusion.org) repositories. The `nonfree` repository includes proprietary software; the `free` repository may include patent-restricted codecs. By running this playbook you accept responsibility for compliance with your local laws and organisational policies.


---

## Project Structure

```
fedora-setup/
├── fedora-do.yml                    ← Master playbook — run this
├── .gitignore
│
├── vars/
│   ├── app_catalog.yml         ← ✏️  Edit this to add/remove apps (RPM + Flatpak)
│   ├── gnome.yml               ← ✏️  Edit this for GNOME dconf settings
│   ├── smb.yml                 ← ✏️  Edit this for SMB share definitions
│   ├── backup.yml              ← ✏️  Edit this for backup/restore paths and retention
│   ├── rpm_configs.yml         ← ✏️  Edit this to add/remove RPM config backup entries
│   ├── git.yml.template        ← Copy → vars/git.yml, fill in (not committed)
│   └── secrets.yml.template    ← Copy → vars/secrets.yml, fill in, vault-encrypt
│
└── tasks/
    ├── dnf.yml                 ← DNF tuning + system update
    ├── firmware.yml            ← fwupdmgr firmware updates
    ├── repos.yml               ← RPM Fusion, FFmpeg, VS Code repo
    ├── packages.yml            ← RPM installs (reads from vars/app_catalog.yml)
    ├── flatpaks.yml            ← Flathub setup + Flatpak installs (same source)
    ├── docker.yml              ← Docker CE + Compose plugin
    ├── virtualization.yml      ← KVM/QEMU/libvirt stack
    ├── smb.yml                 ← Credential files + fstab mounts
    ├── gnome.yml               ← dconf settings + extensions
    ├── fonts.yml               ← System font installation + fc-cache
    ├── shell.yml               ← fish shell + starship prompt
    ├── git-config.yml          ← Git globals + GNOME Keyring credential helper
    ├── cleanup.yml             ← autoremove + cache clear
    └── backup.yml              ← Flatpak config backup / restore (tags: backup, restore)
```

---

## Quick Start

The following is required to use the runbook post-installation of the base OS.

```bash
# 1. Install Ansible and required collections
sudo dnf install ansible python3-psutil
ansible-galaxy collection install community.general ansible.posix

# 2. Set up Git identity
cp vars/git.yml.template vars/git.yml
# Edit vars/git.yml with your name and email (this file is gitignored)

# 3. Set up secrets for SMB mounts
cp vars/secrets.yml.template vars/secrets.yml
# Edit vars/secrets.yml with real credentials, then encrypt:
ansible-vault encrypt vars/secrets.yml

# 4. Run the full setup (no --tags = everything runs)
ansible-playbook fedora-do.yml -K -J

# 5. Log out and back in (or reboot)
```

---

## Tag Reference

### Run modes

| Mode | How | Result |
|---|---|---|
| **Full run** | no `--tags` flag | Every task runs |
| **Specific tasks only** | `--tags foo,bar` | Only tagged tasks run (plus `always`) |
| **Full run, skip something** | `--skip-tags foo` | Everything runs except the skipped tag(s) |

### Tags

| Tag | What it covers |
|---|---|
| `always` | DNF tune, update, RPM Fusion, FFmpeg, base packages, Flathub setup, cleanup — runs in all modes |
| `all-apps` | Shortcut: installs every app category at once (tools through gaming) |
| `tools` | btrfs-assistant, restic, bat, eza, fzf, fastfetch + Flatseal, GearLever |
| `desktop` | GNOME Tweaks, dconf Editor + PeaZip, Extension Manager |
| `development` | GCC, Node, Go, Python, VS Code |
| `productivity` | LibreOffice, Obsidian |
| `communication` | Signal, Discord |
| `graphics` | Krita, GIMP, Inkscape, Darktable, OBS Studio |
| `media` | mpv + Haruna, HandBrake, VLC, Kdenlive |
| `gaming` | Steam, Lutris, RetroArch |
| `docker` | Docker CE, containerd, compose plugin |
| `virt` | KVM/QEMU, libvirt, virt-manager — full virtualisation stack |
| `smb` | Credential files, fstab entries, mounts |
| `gnome` | Dark mode, Dash-to-Dock, Night Light, Nautilus, dconf keys |
| `firmware` | fwupdmgr metadata refresh + firmware updates |
| `fonts` | Noto, Roboto, JetBrains Mono, Fira Code, Cascadia Code |
| `shell` | fish shell (set as default) + starship prompt |
| `git-config` | Git identity, default branch, GNOME Keyring credential helper |
| `backup` | Run both Flatpak and RPM config backup (never runs automatically) |
| `restore` | Run both Flatpak and RPM config restore (never runs automatically) |
| `flatpak-backup` | Flatpak config backup only (`~/.var/app` → SMB share) |
| `flatpak-restore` | Flatpak config restore only (SMB share → `~/.var/app`) |
| `rpm-backup` | RPM app config backup only (paths from `vars/rpm_configs.yml` → SMB share) |
| `rpm-restore` | RPM app config restore only (SMB share → home paths per `vars/rpm_configs.yml`) |

### Examples

> **Flag aliases used below:**
> `-K` = `--ask-become-pass` (sudo password prompt)
> `-J` = `--ask-vault-pass` (vault password prompt)

```bash
# ── Mode 1: full run (no --tags = everything) ─────────────────────────────────
ansible-playbook fedora-do.yml -K -J

# ── Mode 2: specific tasks only ───────────────────────────────────────────────
# Only tools and media packages
ansible-playbook fedora-do.yml -K --tags "tools,media"

# Only app categories (all of them, shortcut tag)
ansible-playbook fedora-do.yml -K --tags "always,all-apps"

# Refresh GNOME settings only
ansible-playbook fedora-do.yml -K --tags gnome

# Set up Docker only
ansible-playbook fedora-do.yml -K --tags docker

# Git config only
ansible-playbook fedora-do.yml -K --tags git-config

# SMB mounts only (vault credentials required)
ansible-playbook fedora-do.yml -K -J --tags smb

# ── Mode 3: full run, skip specific tasks ─────────────────────────────────────
# Everything except gaming
ansible-playbook fedora-do.yml -K --skip-tags gaming

# Everything except virtualisation and firmware
ansible-playbook fedora-do.yml -K --skip-tags "virt,firmware"

# ── Backup & restore (never run automatically — must be explicitly tagged) ────
# Back up both Flatpak and RPM app configs
ansible-playbook fedora-do.yml -K -J --tags backup

# Restore both Flatpak and RPM app configs (run after a fresh install)
ansible-playbook fedora-do.yml -K -J --tags restore

# Flatpak only
ansible-playbook fedora-do.yml -K -J --tags flatpak-backup
ansible-playbook fedora-do.yml -K -J --tags flatpak-restore

# RPM only
ansible-playbook fedora-do.yml -K -J --tags rpm-backup
ansible-playbook fedora-do.yml -K -J --tags rpm-restore
```

---

## Adding Applications

Open **`vars/app_catalog.yml`** — this is the only file you need to edit for apps.

Each category has two lists:

```yaml
tools:
  rpm:
    - btrfs-assistant   # installed via dnf
    - restic
  flatpak:
    - com.github.tchx84.Flatseal   # installed via Flathub
    - io.missioncenter.MissionCenter
```

Both lists are installed when you run `--tags tools`. To skip Flatpaks for a
category, leave `flatpak: []`. To skip RPMs, leave `rpm: []`.

### Adding a New Category

1. Add a block in `vars/app_catalog.yml`:
   ```yaml
   security:
     rpm:
       - nmap
       - wireshark
     flatpak:
       - org.wireshark.Wireshark
   ```

2. Add a task block in `tasks/packages.yml`:
   ```yaml
   - name: "Packages [security] | Install security RPM packages"
     become: true
     ansible.builtin.dnf:
       name: "{{ app_catalog.security.rpm }}"
       state: present
     when: app_catalog.security.rpm | length > 0
     tags: security
   ```

3. Add a task block in `tasks/flatpaks.yml`:
   ```yaml
   - name: "Flatpak [security] | Install security Flatpak apps"
     community.general.flatpak:
       name: "{{ app_catalog.security.flatpak }}"
       state: present
       method: system
       remote: flathub
     when: app_catalog.security.flatpak | length > 0
     tags: security
   ```

---

## SMB Shares

**`vars/smb.yml`** — non-secret share config (server, mount point, options, which credential file to use).  
**`vars/secrets.yml`** — vault-encrypted credentials (username/password per credential key).

The `cred_key` in `smb.yml` links each share to its credentials in `secrets.yml`.
`mount_point` accepts either an absolute path or a `~/…` path (expanded to `$HOME` at run time):

```yaml
# vars/smb.yml
smb_shares:
  - name: "Media"
    src: "//192.168.1.100/Media"
    mount_point: "~/Mounts/Media"        # ~/... expanded to $HOME/... at run time
    # mount_point: "/mnt/nas/Media"      # absolute path also accepted
    cred_key: nas_primary                # ← matches key in secrets.yml
    cred_file: /etc/samba/.creds_nas_primary
    options: "uid=1000,gid=1000,vers=3.0,_netdev"

# vars/secrets.yml (encrypted)
smb_credentials:
  nas_primary:
    username: "myuser"
    password: "mypassword"
```

Multiple shares can share the same `cred_key` / `cred_file` (e.g. two shares
on the same NAS with the same login). The playbook writes each unique
credential file only once.

---

## Backup & Restore

Backs up and restores application configuration to/from a network share. All operations are tagged `never` and **will not run** during a normal full run — they must be explicitly requested.

- **Flatpak**: config from `~/.var/app/<app-id>/` — app list sourced from `vars/app_catalog.yml`
- **RPM**: config paths declared in `vars/rpm_configs.yml` — opt-in registry, supports multiple directories per app

### Configuration

Edit **`vars/backup.yml`** to set destination paths and retention (shared between Flatpak and RPM):

```yaml
flatpak_backup_base_dir: "~/Mounts/backups/config-backups/flatpak"
flatpak_backup_tmp_dir:  "~/.cache/flatpak_backup_staging"

rpm_backup_base_dir: "~/Mounts/backups/config-backups/rpm"
rpm_backup_tmp_dir:  "~/.cache/rpm_backup_staging"

backup_max_archives:  5   # max dated archives to keep
backup_min_archives:  2   # always keep at least this many regardless of age
backup_max_age_days: 30   # delete dated archives older than this
backup_monthly_months: 6  # monthly snapshots to keep (0 = disabled)
```

> **Avoid `/tmp` for staging:** on most Fedora systems `/tmp` is a `tmpfs` (RAM-backed). Large archives can exhaust memory. The defaults use `~/.cache/` which stays on disk.

### RPM config registry (`vars/rpm_configs.yml`)

Edit this file to declare which RPM-installed apps have configuration worth backing up:

```yaml
rpm_configs:
  code:
    paths:
      - ~/.config/Code/User        # multiple paths per app are supported
  darktable:
    paths:
      - ~/.config/darktable
      - ~/.local/share/darktable
  flatpak-overrides:
    no_package_check: true         # skip rpm -q validation for non-RPM entries
    paths:
      - ~/.local/share/flatpak/overrides
```

Paths must be **directories** (not individual files). Missing paths are silently skipped. A warning is added to the report if a package is not installed or if all paths for an entry are absent.

### Backup

```bash
# Both Flatpak and RPM
ansible-playbook fedora-do.yml -K -J --tags backup

# Individually
ansible-playbook fedora-do.yml -K -J --tags flatpak-backup
ansible-playbook fedora-do.yml -K -J --tags rpm-backup
```

Each backup creates a dated archive and a `*_latest.tar.gz` on the share. Old archives are pruned per `vars/backup.yml`. Requires `rsync` (included in the `tools` tag).

### Restore

```bash
# Both Flatpak and RPM
ansible-playbook fedora-do.yml -K -J --tags restore

# Individually
ansible-playbook fedora-do.yml -K -J --tags flatpak-restore
ansible-playbook fedora-do.yml -K -J --tags rpm-restore
```

Extracts `*_latest.tar.gz` from the share and rsync's each entry back to its original location. Run after a fresh install once apps have been installed:

```bash
ansible-playbook fedora-do.yml -K -J --tags "always,all-apps,restore"
```

---

## GNOME Settings

Edit **`vars/gnome.yml`** to change any dconf value. Values must be in
GVariant format — the same format the `dconf` command uses:

| Type | Example |
|---|---|
| String | `"'prefer-dark'"` (single-quoted inside double-quoted) |
| Boolean | `"true"` or `"false"` |
| Integer | `"4"` |
| Array | `"['ext1@uuid', 'ext2@uuid']"` |

To enable additional GNOME Shell extensions, add their UUID to
`gnome_enabled_extensions` in `vars/gnome.yml`. The extension must already be
installed (either via RPM or Extension Manager).

---

## License

Copyright (C) 2026 Pieter van Heerden

This project is licensed under the **GNU General Public License v2.0 only** (GPL-2.0-only). See the [LICENSE](LICENSE) file for the full text.

In plain terms: you are free to use, modify, and distribute this playbook, but any distributed modifications must also be released under GPL-2.0.

### Third-party content note

This playbook *installs* third-party software at runtime but does not bundle or redistribute any of it. Each piece of software retains its own license. Notably:

- **RPM Fusion nonfree** packages may include proprietary software (see the disclaimer section above).
- **Fisher** (MIT) and **Tide** (MIT) are downloaded at runtime; their licenses are independent of this project.
- **Ansible collections** used (`community.general`, `ansible.posix`) are GPL-compatible.
