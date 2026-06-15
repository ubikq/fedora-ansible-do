# Fedora Workstation Setup — Ansible Playbook

## Contents

- [About This Project](#about-this-project)
- [Disclaimer](#-disclaimer)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Run Configuration](#run-configuration)
- [Tag Reference](#tag-reference)
- [Adding Applications](#adding-applications)
- [SMB Shares](#smb-shares)
- [Backup & Restore](#backup--restore)
- [GNOME Settings](#gnome-settings)
- [License](#license)

---

## About This Project

This Ansible playbook automates the post-install configuration of Fedora Workstation, systematically transforming a stock setup into a fully provisioned environment. All application lists are centralized in `vars/app_catalog.yml`, and **which task groups run during a default execution is controlled by a single vars file** — `vars/run_config.yml` — making it straightforward to include or exclude features without needing to understand Ansible's tag system.

### Features

- **Configurable Default Run**: A single file — `vars/run_config.yml` — controls which task groups are included in a default playbook run. Set any flag to `false` to skip that group (e.g. exclude SMB mounts or RPM Fusion); set `niri` or `restore` to `true` to include those opt-in groups automatically. No `--tags` knowledge required. See [Run Configuration](#run-configuration).

- **System Maintenance & Upgrades**: Automates the standard system update cycle (`dnf upgrade`), manages official Fedora repositories, and handles package cleanup tasks.

- **Third-Party Repository Integration**: Safely provisions third-party repositories—specifically enabling **RPM Fusion** (`free` and `nonfree`) to seamlessly unlock restricted media codecs and proprietary hardware drivers (such as Nvidia graphics drivers). Can be excluded via `run_config.rpm_fusion: false`.

- **Core Software & Tool Deployment**: Orchestrates the bulk installation of applications, system utilities, and developer dependencies via native package management (`dnf`), eliminating manual setup.

- **Flatpak Integration**: Configures Flatpak functionality and enables the **Flathub repository**, laying down the framework to install sandbox-isolated desktop applications.

- **User Environment Personalization**: Deploys localized dotfiles, system preferences, shell configurations, and workflow tools to match a specific user's desktop routine.

- **Configuration Backup & Restore**: Backs up and restores application configuration to and from a network share, covering both Flatpak and RPM-installed applications. Maintains dated archives, a rolling latest snapshot, and optional monthly snapshots with configurable retention. **Backup never runs automatically** and must be triggered explicitly with `--tags backup`. **Restore** can be included in a default run by setting `run_config.restore: true` in `vars/run_config.yml`. See the [Backup & Restore](#backup--restore) section for details.

- **Niri Compositor & DankMaterialShell** (opt-in): Installs the niri scrolling tiling Wayland compositor alongside the DankMaterialShell desktop shell. Excluded from a default run unless `run_config.niri: true` is set in `vars/run_config.yml`. A default `config.kdl` with customized keybindings is deployed on first run as a ready-to-use starting point for customization. Once personalized, the configuration can be backed up and restored across fresh installations using the playbook's built-in backup system.

### Feature Roadmap

- Placeholder for planned new features

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
fedora-ansible-do/
├── fedora-do.yml                    ← Master playbook — run this
├── .gitignore
│
├── vars/
│   ├── run_config.yml          ← ✏️  Edit this to control which groups run by default
│   ├── app_catalog.yml         ← ✏️  Edit this to add/remove apps (RPM + Flatpak)
│   ├── gnome.yml               ← ✏️  Edit this for GNOME dconf settings
│   ├── smb.yml                 ← ✏️  Edit this for SMB share definitions
│   ├── backup.yml              ← ✏️  Edit this for backup/restore paths and retention
│   ├── config_registry.yml     ← ✏️  Edit this to add/remove config backup entries (RPM, Flatpak overrides, etc.)
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
    ├── niri.yml                ← Niri compositor + DankMaterialShell (opt-in)
    ├── fonts.yml               ← System font installation + fc-cache
    ├── shell.yml               ← fish shell + starship prompt
    ├── git-config.yml          ← Git globals + GNOME Keyring credential helper
    ├── cleanup.yml             ← autoremove + cache clear
    ├── backup.yml              ← Dispatcher — includes the four sub-files below
    └── backup/
        ├── flatpak-backup.yml  ← Flatpak config backup  (tags: backup, flatpak-backup)
        ├── flatpak-restore.yml ← Flatpak config restore (tags: restore, flatpak-restore)
        ├── rpm-backup.yml      ← RPM config backup      (tags: backup, rpm-backup)
        └── rpm-restore.yml     ← RPM config restore     (tags: restore, rpm-restore)
```

---

## Quick Start

The following is required to use the runbook post-installation of the base OS.

```bash
# 1. Install Ansible and required collections
sudo dnf install ansible python3-psutil
ansible-galaxy collection install community.general ansible.posix

# 2. Review and adjust which groups run by default
# Edit vars/run_config.yml — enable niri, restore, or disable groups you don't need

# 3. (Optional) Set up Git identity — only needed if you enable run_config.git_config: true
cp vars/git.yml.template vars/git.yml
# Edit vars/git.yml with your name and email (this file is gitignored)

# 4. (Optional) Set up secrets for SMB mounts — only needed if you enable run_config.smb: true
cp vars/secrets.yml.template vars/secrets.yml
# Edit vars/secrets.yml with real credentials, then encrypt:
ansible-vault encrypt vars/secrets.yml

# 5. Run the full setup (no --tags = groups run per vars/run_config.yml)
ansible-playbook fedora-do.yml -K

# 6. Log out and back in (or reboot)
```

---

## Run Configuration

Edit **`vars/run_config.yml`** to control which task groups run during a default playbook execution (no `--tags` flag).

```yaml
run_config:

  # ── System ──────────────────────────────────────────────────────────────────
  system_update: true   # DNF tuning + dnf upgrade --refresh
  rpm_fusion:    true   # RPM Fusion repos, FFmpeg, @multimedia codecs
  firmware:      true   # fwupdmgr firmware updates

  # ── Applications ────────────────────────────────────────────────────────────
  apps_tools:         true
  apps_desktop:       true
  apps_development:   true
  apps_productivity:  true
  apps_communication: true
  apps_graphics:      true
  apps_media:         true
  apps_gaming:        false  # reserved — no active apps

  # ── System Services ─────────────────────────────────────────────────────────
  docker:         false   # requires Docker CE repo — disabled by default
  virtualization: false   # KVM/QEMU stack — disabled by default
  smb:            false   # requires vars/secrets.yml — disabled by default

  # ── Desktop Environment ──────────────────────────────────────────────────────
  gnome:      true
  fonts:      true
  shell:      true
  git_config: false   # requires vars/git.yml — disabled by default

  # ── Opt-in (set true to include in a default run) ────────────────────────────
  niri:    false   # Niri compositor + DankMaterialShell
  restore: false   # Restore backed-up app configs from SMB share

  # ── Maintenance ──────────────────────────────────────────────────────────────
  cleanup: true
```

### Common scenarios

**Full setup including niri/DMS and config restore in a single run:**
```bash
# Set in vars/run_config.yml:
#   niri: true
#   restore: true
ansible-playbook fedora-do.yml -K -J
```

**Enable Docker and virtualisation for a dev workstation:**
```bash
# Set in vars/run_config.yml:
#   docker:         true
#   virtualization: true
ansible-playbook fedora-do.yml -K
```

**Run a group standalone without editing the vars file:**
```bash
# Explicit --tags always run regardless of run_config settings:
ansible-playbook fedora-do.yml -K --tags niri

# To include niri in all future default runs without editing the file:
ansible-playbook fedora-do.yml -K -e "run_config={niri: true}"
```

> **Note:** Explicit `--tags` always win over `run_config.yml`. Running `--tags foo` will execute that group regardless of the corresponding flag in `run_config.yml`. Use `run_config.yml` flags to control what runs in a *default run* (no `--tags` flag). To include an opt-in group in all future default runs, set it to `true` in `run_config.yml` instead.

---

## Tag Reference

### Run modes

| Mode | How | Result |
|---|---|---|
| **Full run** | no `--tags` flag | Groups run according to `vars/run_config.yml` (see [Run Configuration](#run-configuration)) |
| **Specific tasks only** | `--tags "foo,bar"` | Only tagged tasks run (`always` tasks run automatically in all modes) |
| **Full run, skip something** | `--skip-tags foo` | Everything runs except the skipped tag(s) |

### Tags

| Tag | What it covers |
|---|---|
| `always` | Base packages, Flathub setup, Brave/VS Code repo setup — runs in all modes. Also includes `system-update` and `rpm-fusion` tasks (both carry the `always` tag) |
| `system-update` | DNF config tune + full `dnf upgrade --refresh` — also tagged `always` (runs in all modes); use `--tags system-update` to run standalone |
| `rpm-fusion` | RPM Fusion repos, FFmpeg, `@multimedia` codecs — also tagged `always` (runs in all modes); use `--tags rpm-fusion` to run standalone |
| `all-apps` | Shortcut: installs every app category at once (tools through gaming) |
| `tools` | btrfs-assistant, eza, bat, fzf, fastfetch, micro, tldr, restic, doublecmd-qt6, Cockpit plugins + Bazaar, Warehouse, Flatseal, GearLever, Dolphin, Kate, CockpitClient, Meld |
| `desktop` | GNOME Tweaks, dconf Editor + PeaZip, Extension Manager |
| `development` | *(no active apps by default — uncomment packages in `vars/app_catalog.yml` to enable)* |
| `productivity` | Bitwarden, Obsidian, MarkText, Foliate, Okular |
| `communication` | Brave Browser (RPM, for PWA support), Vivaldi, Floorp |
| `graphics` | nomacs, Krita |
| `media` | mpv, Haruna (RPM), HandBrake, yt-dlp |
| `gaming` | *(no active apps — category reserved)* |
| `docker` | Docker CE, containerd, compose plugin |
| `virt` | KVM/QEMU, libvirt, virt-manager — full virtualisation stack |
| `smb` | Credential files, fstab entries, mounts |
| `gnome` | Dark mode, Dash-to-Dock, Night Light, Nautilus, dconf keys |
| `firmware` | fwupdmgr metadata refresh + firmware updates |
| `fonts` | Noto, Roboto, JetBrains Mono, Fira Code, Cascadia Code |
| `shell` | fish shell (set as default) + Fisher + Tide + starship prompt |
| `git-config` | Git identity, default branch, GNOME Keyring credential helper |
| `cleanup` | dnf autoremove, cache clean, Flatpak updates — runs on full run or `--tags cleanup` |
| `niri` | niri compositor + DankMaterialShell — excluded from default run unless `run_config.niri: true`; use `-e "run_config={niri: true}"` to run standalone |
| `backup` | Run both Flatpak and RPM config backup — **never runs automatically**, must be explicitly tagged |
| `restore` | Run both Flatpak and RPM config restore — runs automatically when `run_config.restore: true` in a default run, or always when explicitly tagged with `--tags restore` |
| `flatpak-backup` | Flatpak config backup only (`~/.var/app` → SMB share) |
| `flatpak-restore` | Flatpak config restore only (SMB share → `~/.var/app`) |
| `rpm-backup` | RPM app config backup only (paths from `vars/config_registry.yml` → SMB share) |
| `rpm-restore` | RPM app config restore only (SMB share → home paths per `vars/config_registry.yml`) |

> **Tip:** To see all available tags without running the playbook:
> ```bash
> ansible-playbook fedora-do.yml --list-tags
> ```

### Examples

> **Flag aliases used below:**
> `-K` = `--ask-become-pass` (sudo password prompt)
> `-J` = `--ask-vault-pass` (vault password prompt)

```bash
# ── Mode 1: full run (no --tags = groups run per vars/run_config.yml) ─────────
ansible-playbook fedora-do.yml -K -J

# ── Mode 2: specific tasks only ───────────────────────────────────────────────
# Only tools and media packages
ansible-playbook fedora-do.yml -K --tags "tools,media"

# Only app categories (all of them, shortcut tag)
ansible-playbook fedora-do.yml -K --tags all-apps

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

# ── Backup (never runs automatically — must be explicitly tagged) ─────────────
# Back up both Flatpak and RPM app configs
ansible-playbook fedora-do.yml -K -J --tags backup

# ── Restore (runs automatically when run_config.restore: true) ────────────────
# Restore both Flatpak and RPM app configs (explicitly tagged)
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

Backs up and restores application configuration to/from a network share.

- **Backup** is tagged `never` and will **never run automatically** — it must be triggered explicitly with `--tags backup`.
- **Restore** runs automatically when `run_config.restore: true` is set in `vars/run_config.yml`, or explicitly with `--tags restore`.

- **Flatpak**: config from `~/.var/app/<app-id>/` — app list sourced from `vars/app_catalog.yml`
- **RPM**: config paths declared in `vars/config_registry.yml` — opt-in registry, covers RPM packages, Flatpak overrides, AppImages, and other non-RPM configs

### Configuration

Edit **`vars/backup.yml`** to set destination paths and retention (shared between Flatpak and RPM):

```yaml
flatpak_backup_base_dir: "~/Mounts/backups/linux-backups/fedora/config-flatpak"
flatpak_backup_tmp_dir:  "~/.cache/flatpak_backup_staging"

rpm_backup_base_dir: "~/Mounts/backups/linux-backups/fedora/config-rpm"
rpm_backup_tmp_dir:  "~/.cache/rpm_backup_staging"

backup_max_archives:  5   # max dated archives to keep
backup_min_archives:  2   # always keep at least this many regardless of age
backup_max_age_days: 30   # delete dated archives older than this
backup_monthly_months: 6  # monthly snapshots to keep (0 = disabled)
```

> **Avoid `/tmp` for staging:** on most Fedora systems `/tmp` is a `tmpfs` (RAM-backed). Large archives can exhaust memory. The defaults use `~/.cache/` which stays on disk.

### Config registry (`vars/config_registry.yml`)

Edit this file to declare which apps have configuration worth backing up. The registry covers RPM packages, Flatpak overrides, AppImages, and other non-RPM entries (fish shell, GNOME desktop, pipewire, etc.):

```yaml
config_registry:
  code:
    paths:
      - ~/.config/Code/User        # multiple paths per app are supported
  darktable:
    paths:
      - ~/.config/darktable
      - ~/.local/share/darktable
  doublecmd-qt6:
    paths:
      - ~/.config/doublecmd
    files:
      - ~/.local/share/applications/doublecmd.desktop   # individual file
  flatpak-overrides:
    no_package_check: true         # skip rpm -q validation for non-RPM entries
    paths:
      - ~/.local/share/flatpak/overrides
```

Use `paths:` for directories and `files:` for individual files. Use `exclude:` to pass rsync exclusion patterns relative to each `paths:` entry — useful for skipping cache subdirectories or other inaccessible paths (e.g. `exclude: [config/restic]`). All three keys are optional. Missing entries are silently skipped. A warning is added to the report if a package is not installed or if all entries for a package are absent.

### Backup

```bash
# Both Flatpak and RPM
ansible-playbook fedora-do.yml -K -J --tags backup

# Individually
ansible-playbook fedora-do.yml -K -J --tags flatpak-backup
ansible-playbook fedora-do.yml -K -J --tags rpm-backup
```

Each backup creates a dated archive and a `*_latest.tar.gz` on the share. Old archives are pruned per `vars/backup.yml`. Requires `rsync` (installed automatically via the `base` category).

### Restore

```bash
# Both Flatpak and RPM
ansible-playbook fedora-do.yml -K -J --tags restore

# Individually
ansible-playbook fedora-do.yml -K -J --tags flatpak-restore
ansible-playbook fedora-do.yml -K -J --tags rpm-restore
```

Extracts `*_latest.tar.gz` from the share and rsync's each entry back to its original location. The simplest way to run a full setup including restore is to set `run_config.restore: true` in `vars/run_config.yml` and run without any `--tags` flag:

```bash
# Set run_config.restore: true (and run_config.niri: true if applicable) then:
ansible-playbook fedora-do.yml -K -J
```

Alternatively, trigger restore explicitly after apps are installed:

```bash
ansible-playbook fedora-do.yml -K -J --tags restore
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
