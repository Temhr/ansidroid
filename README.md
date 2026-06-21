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

```
┌─────────────────────────────────────────────────────────┐
│                     YOU (any machine)                   │
│              git push → github.com/temhr/ansidroid      │
└────────────────────────────┬────────────────────────────┘
                             │  GitHub (source of truth)
          ┌──────────────────┼───────────────┬────────────┐
          │                  │               │            │
    ┌─────▼──────┐   ┌───────▼────┐  ┌──────▼─────┐ ┌───▼──────┐
    │  p8p        │   │  p3axl     │  │  pxl       │ │  n5x     │
    │  Termux     │   │  Termux    │  │  Termux    │ │  Termux  │
    │  crond      │   │  crond     │  │  crond     │ │  crond   │
    │  ansible-   │   │  ansible-  │  │  ansible-  │ │ ansible- │
    │  pull       │   │  pull      │  │  pull      │ │  pull    │
    └─────────────┘   └────────────┘  └────────────┘ └──────────┘
         ↑ hourly cron on each device, minute staggered by device name seed
```

### Layers and division of concerns

```
Layer 1 — Identity        host_vars/<device>.yml
                          Each device knows its own name, OS group, package
                          list, upgrade strategy, and feature flags.

Layer 2 — Group policy    group_vars/grapheneos.yml
                          group_vars/lineageos.yml
                          group_vars/android.yml
                          group_vars/all/     (applies to every device)
                          OS-level defaults shared across all devices
                          running the same OS.

Layer 3 — Roles           roles/
                          Discrete, idempotent units of work. Each role
                          manages one concern and is safe to re-run every
                          hour without side effects.

                          common/        → runs on every device
                            packages     → safe pkg update + upgrade
                            dotfiles     → .bashrc, .profile, aliases
                            logging      → log rotation, audit trail
                          cron/          → installs the ansible-pull loop
                                           and Termux:Boot crond starter
                          obtainium/     → pushes per-device app JSON list
                          syncthing/     → deploys syncthing config
                          grapheneos/    → GOS-specific tasks
                          lineageos/     → su handling, MicroG awareness
                          notifications/ → termux-notification on failure

Layer 4 — Entry point     local.yml
                          ansible-pull default playbook. Loads identity,
                          assigns OS group, runs roles conditionally.

Layer 5 — Bootstrap       bootstrap.sh
                          Run once per device in Termux. Installs deps,
                          writes device identity, fires the first pull.
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

| File                          | Contents                              |
|-------------------------------|---------------------------------------|
| `~/.ansible/ansible-pull.log` | Full output of every ansible-pull run |
| `~/.ansible/upgrade.log`      | Timestamped upgrade result per run    |
| `~/.ansible/pkg_state_before.txt` | Installed packages before upgrade |

Log rotation runs nightly at 03:00, keeping the last 500 lines of the pull log.
