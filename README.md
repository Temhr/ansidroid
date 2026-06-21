# ansidroid

Declarative, self-provisioning configuration management for a fleet of Android devices
using `ansible-pull`. Each device clones this repo and applies its own configuration
autonomously. You push a change to GitHub; every device converges within the hour.

---

## Devices

| Handle  | Model          | OS          | Role               |
|---------|----------------|-------------|--------------------|
| `p8p`   | Pixel 8 Pro    | GrapheneOS  | Primary / hardened |
| `p3axl` | Pixel 3a XL    | LineageOS   | Secondary          |
| `pxl`   | Pixel XL       | LineageOS   | Tertiary           |
| `n5x`   | Nexus 5X       | Android     | Legacy / testing   |

---

## Architecture

### How it flows

```
YOU (any machine)
└── git push → github.com/temhr/ansidroid
                    │
                    │  each device polls GitHub every hour
                    │  and applies any changes to itself
                    │
        ┌───────────┼───────────┬───────────┐
        ▼           ▼           ▼           ▼
      p8p         p3axl        pxl         n5x
   (GrapheneOS) (LineageOS) (LineageOS)  (Android)
        │
        │  on each device, inside Termux:
        │
        ├── crond            runs the pull loop on a schedule
        ├── ansible-pull     clones the repo, runs local.yml
        └── local.yml        applies roles based on device identity
```

### Repo layout

```
ansidroid/
│   local.yml                       ansible-pull entry point; runs roles conditionally by OS group
│   bootstrap.sh                    run once per device; installs deps, writes identity, fires first pull
│   ansible.cfg                     Ansible settings tuned for Termux (log path, no retry files)
│   inventory                       targets localhost only — the device provisions itself
│
├── host_vars/                      LAYER 1 — per-device identity and overrides
│   ├── p8p.yml                     Pixel 8 Pro   · GrapheneOS · conservative upgrades · syncthing + notifications
│   ├── p3axl.yml                   Pixel 3a XL  · LineageOS  · conservative upgrades
│   ├── pxl.yml                     Pixel XL     · LineageOS  · full upgrade strategy
│   └── n5x.yml                     Nexus 5X     · Android    · skip upgrades · minimal packages
│
├── group_vars/                     LAYER 2 — shared OS-level defaults (host_vars takes precedence)
│   ├── all/
│   │   ├── main.yml                vars for every device: repo URL, log paths, cron timing
│   │   └── vault.yml               encrypted secrets via ansible-vault (API keys, passwords)
│   ├── grapheneos.yml              syncthing on, notifications on, no GMS by default
│   ├── lineageos.yml               root and MicroG off by default; override in host_vars
│   └── android.yml                 upgrade_strategy: skip, minimal feature set
│
├── roles/                          LAYER 3 — idempotent units of work, safe to re-run hourly
│   ├── common/                     runs on every device every pull
│   │   ├── tasks/
│   │   │   ├── main.yml            calls the three sub-task files in order
│   │   │   ├── packages.yml        update index · upgrade (per strategy) · install packages · verify ansible
│   │   │   ├── dotfiles.yml        render .bashrc with device name in prompt
│   │   │   └── logging.yml         nightly cron to trim ansible-pull.log to last 500 lines
│   │   ├── templates/
│   │   │   └── bashrc.j2           shell config; prompt shows device handle and OS group
│   │   └── defaults/main.yml
│   │
│   ├── cron/                       installs the self-updating hourly loop
│   │   ├── tasks/main.yml          deploys run-pull.sh · staggered cron job · Termux:Boot crond script
│   │   └── templates/
│   │       └── run-pull.sh.j2      ansible-pull wrapper with lockfile guard and retry logic
│   │
│   ├── obtainium/                  manages per-device app list
│   │   └── tasks/main.yml          pushes files/obtainium/<device>_apps.json to Obtainium data dir
│   │
│   ├── syncthing/                  deploys Syncthing — p8p only
│   │   ├── tasks/main.yml          writes config, installs boot script, starts daemon if stopped
│   │   └── templates/
│   │       └── syncthing_config.xml.j2     folder definitions + API key from vault
│   │
│   ├── grapheneos/                 p8p only
│   │   ├── tasks/main.yml          writes manual-step checklist (PIN, DNS, auto-reboot); checks Termux:API
│   │   └── defaults/main.yml
│   │
│   ├── lineageos/                  p3axl and pxl
│   │   ├── tasks/main.yml          detects su and MicroG; warns if host_vars disagrees with reality
│   │   └── defaults/main.yml
│   │
│   └── notifications/              p8p only (requires Termux:API)
│       ├── tasks/main.yml          fires Android notification on pull failure; clears sentinel on success
│       └── defaults/main.yml
│
└── files/                          LAYER 4 — static assets copied verbatim to devices
    ├── packages/
    │   ├── common.txt              installed on every device: git python ansible cronie openssh …
    │   ├── grapheneos.txt          reference list for p8p extras (add to host_vars extra_packages)
    │   ├── lineageos.txt           reference list for p3axl/pxl extras
    │   └── android.txt             (intentionally minimal)
    └── obtainium/
        ├── p8p_apps.json           per-device app lists in Obtainium's native export format
        ├── p3axl_apps.json         edit and push to add/remove apps; device picks up within the hour
        ├── pxl_apps.json
        └── n5x_apps.json
```

---

## First-time setup (per device)

1. Install **Termux** and **Termux:Boot** from F-Droid (NOT the Play Store version).
2. Open Termux and run:

```bash
curl -fsSL https://raw.githubusercontent.com/temhr/ansidroid/main/bootstrap.sh \
  | bash -s <device_handle>
```

Replace `<device_handle>` with one of: `p8p`, `p3axl`, `pxl`, `n5x`

3. When prompted, add the printed SSH deploy key to your GitHub repo
   (Settings → Deploy keys → Add key → read-only is sufficient).
4. Done. Ansible will run every hour automatically, even after reboots.

---

## Day-to-day workflow

```bash
# On your PC — make a change, push it
vim host_vars/p8p.yml
git add -A && git commit -m "add ripgrep to p8p packages"
git push
# The device picks it up within the hour.

# Or trigger a manual pull immediately from Termux on the device:
~/.ansible/run-pull.sh
```

---

## Secrets

Sensitive values (API keys, passwords) are encrypted with `ansible-vault`:

```bash
# Create or edit the vault
ansible-vault edit group_vars/all/vault.yml

# The vault password lives only on each device, never in the repo:
~/.ansible/.vault_pass      ← written by bootstrap.sh
```

Reference vault vars in playbooks as `{{ vault_some_secret }}`.

---

## Upgrade strategies (per host_vars)

| Strategy       | Behaviour                                           | Default for  |
|----------------|-----------------------------------------------------|--------------|
| `conservative` | `pkg upgrade --no-full-upgrade` — skips risky deps  | p8p, p3axl   |
| `full`         | `pkg upgrade` — upgrades everything                 | pxl          |
| `skip`         | Only installs new packages, never upgrades existing | n5x          |

---

## Logs

| File                              | Contents                               |
|-----------------------------------|----------------------------------------|
| `~/.ansible/ansible-pull.log`     | Full output of every ansible-pull run  |
| `~/.ansible/upgrade.log`          | Timestamped upgrade result per run     |
| `~/.ansible/pkg_state_before.txt` | Installed packages snapshot pre-upgrade|
| `~/.ansible/setup_checklist.txt`  | Manual setup steps for this device     |

Log rotation runs nightly at 03:00, keeping the last 500 lines of the pull log.
