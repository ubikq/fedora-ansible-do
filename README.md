# Fedora Workstation Setup — Ansible Playbook

Automated post-install configuration for Fedora Workstation. All application
lists live in a single file (`vars/packages.yml`). Everything else is in
dedicated task and var files.

---

## Project Structure

```
fedora-setup/
├── fedora-do.yml                    ← Master playbook — run this
├── .gitignore
│
├── vars/
│   ├── packages.yml            ← ✏️  Edit this to add/remove apps (RPM + Flatpak)
│   ├── gnome.yml               ← ✏️  Edit this for GNOME dconf settings
│   ├── smb.yml                 ← ✏️  Edit this for SMB share definitions
│   ├── git.yml.template        ← Copy → vars/git.yml, fill in (not committed)
│   └── secrets.yml.template    ← Copy → vars/secrets.yml, fill in, vault-encrypt
│
└── tasks/
    ├── dnf.yml                 ← DNF tuning + system update
    ├── firmware.yml            ← fwupdmgr firmware updates
    ├── repos.yml               ← RPM Fusion, FFmpeg, VS Code repo
    ├── packages.yml            ← RPM installs (reads from vars/packages.yml)
    ├── flatpaks.yml            ← Flathub setup + Flatpak installs (same source)
    ├── docker.yml              ← Docker CE + Compose plugin
    ├── virtualization.yml      ← KVM/QEMU/libvirt stack
    ├── smb.yml                 ← Credential files + fstab mounts
    ├── gnome.yml               ← dconf settings + extensions
    ├── fonts.yml               ← System font installation + fc-cache
    ├── shell.yml               ← fish shell + starship prompt
    ├── git-config.yml          ← Git globals + GNOME Keyring credential helper
    └── cleanup.yml             ← autoremove + cache clear
```

---

## Quick Start

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
| `all-apps` | Shortcut: installs every app category at once (desktop through gaming) |
| `desktop` | GNOME tweaks, Extension Manager, Boxes |
| `tools` | btrfs-assistant, restic, bat, eza, fzf + Flatseal, MissionCenter |
| `graphics` | GIMP, Inkscape, Darktable + Krita, OBS Studio |
| `media` | VLC, mpv, HandBrake + Celluloid, Spotify, Kdenlive |
| `development` | GCC, Node, Go, Python, VS Code |
| `productivity` | LibreOffice, Obsidian, Rnote, Evolution |
| `communication` | Signal, Element, Discord, Zoom |
| `gaming` | Steam, Lutris, Gamemode, RetroArch |
| `docker` | Docker CE, containerd, compose plugin |
| `virt` | KVM/QEMU, libvirt, virt-manager — full virtualisation stack |
| `smb` | Credential files, fstab entries, mounts |
| `gnome` | Dark mode, Dash-to-Dock, Night Light, Nautilus, dconf keys |
| `firmware` | fwupdmgr metadata refresh + firmware updates |
| `fonts` | Noto, Roboto, JetBrains Mono, Fira Code, Cascadia Code |
| `shell` | fish shell (set as default) + starship prompt |
| `git-config` | Git identity, default branch, GNOME Keyring credential helper |

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
```

---

## Adding Applications

Open **`vars/packages.yml`** — this is the only file you need to edit for apps.

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

1. Add a block in `vars/packages.yml`:
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
