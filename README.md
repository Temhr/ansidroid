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
├── local.yml                       entry point; runs roles conditionally by OS group
├── bootstrap.sh                    initial; installs deps, writes identity, fires first pull
├── ansible.cfg                     tuned for Termux (log path, no retry files)
├── inventory                       targets localhost only — the device provisions itself
│
├── host_vars/                      per-device identity and overrides
│   ├── p8p.yml                     conservative upgrades · syncthing + notifications
│   ├── p3axl.yml                   conservative upgrades
│   ├── pxl.yml                     full upgrade strategy
│   └── n5x.yml                     skip upgrades · minimal packages
│
├── group_vars/                     shared OS-level defaults (host_vars takes precedence)
│   ├── all/
│   │   ├── main.yml                universal vars: repo URL, log paths, cron timing
│   │   └── vault.yml               encrypted secrets via ansible-vault (API keys, passwords)
│   ├── grapheneos.yml              syncthing on, notifications on, no GMS by default
│   ├── lineageos.yml               root and MicroG off by default; override in host_vars
│   └── android.yml                 upgrade_strategy: skip, minimal feature set
│
├── roles/                          idempotent units of work, safe to run hourly
│   ├── common/                     all runs, each device, every pull
│   │   ├── tasks/
│   │   │   ├── main.yml            calls the three sub-task files in order
│   │   │   ├── packages.yml        update index · upgrade & install pkgs · verify ansible
│   │   │   ├── dotfiles.yml        render .bashrc with device name in prompt
│   │   │   └── logging.yml         nightly cron to trim ansible-pull.log to last 500 lines
│   │   ├── templates/
│   │   │   └── bashrc.j2           shell config; prompt shows device handle and OS group
│   │   └── defaults/main.yml
│   │
│   ├── cron/                       installs the self-updating hourly loop
│   │   ├── tasks/main.yml          deploys run-pull.sh · Termux:Boot crond script
│   │   └── templates/
│   │       └── run-pull.sh.j2      ansible-pull wrapper with lockfile guard and retry logic
│   │
│   ├── obtainium/                  manages per-device app list
│   │   └── tasks/main.yml          pushes files/obtainium/<device>_apps.json to data dir
│   │
│   ├── syncthing/                  deploys Syncthing — p8p only
│   │   ├── tasks/main.yml          writes config, installs boot script, restarts daemon
│   │   └── templates/
│   │       └── syncthing_config.xml.j2     folder definitions + API key from vault
│   │
│   ├── grapheneos/                 p8p only
│   │   ├── tasks/main.yml          writes manual-step checklist (PIN, DNS, auto-reboot)
│   │   └── defaults/main.yml
│   │
│   ├── lineageos/                  p3axl and pxl
│   │   ├── tasks/main.yml          detects su and MicroG; warns if host_vars disagrees
│   │   └── defaults/main.yml
│   │
│   └── notifications/              p8p only (requires Termux:API)
│       ├── tasks/main.yml          fires on pull failure; clears sentinel on success
│       └── defaults/main.yml
│
└── files/                          static assets copied verbatim to devices
    ├── packages/
    │   ├── common.txt              every device: git python ansible cronie openssh …
    │   ├── grapheneos.txt          ref list for p8p extras (add to host_vars extra_packages)
    │   ├── lineageos.txt           ref list for p3axl/pxl extras
    │   └── android.txt             (intentionally minimal)
    └── obtainium/
        ├── p8p_apps.json           per-device app lists in Obtainium's native export format
        ├── p3axl_apps.json         edit to add/remove apps; device picks up within the hour
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

---

## Design patterns

### Error handling

Every role wraps its tasks in `block / rescue` so a failure in one task logs an
error and skips that task, but execution continues with the rest of the playbook.
The cron loop never aborts mid-run and leaves the device in a half-configured state.

```yaml
- name: Do something
  block:
    - name: The actual task
      command: ...
  rescue:
    - name: Log failure and carry on
      lineinfile:
        path: "{{ ansible_log_file }}"
        line: "{{ ansible_date_time.iso8601 }} | {{ device_label }} | ERROR: ... — {{ ansible_failed_result.msg | default('unknown') }}"
        create: true
```

`any_errors_fatal: false` is set on every play in `local.yml` as an outer safety
net — even if a task escapes its `rescue`, subsequent plays still run.

Failures are never silently swallowed. Every `rescue` block writes a timestamped
line to `~/.ansible/ansible-pull.log` or `~/.ansible/upgrade.log` so you always
know what failed and when.

**Failure behaviour by role:**

| Role | If it fails | Recovery |
|------|-------------|----------|
| `common/packages` — `pkg update` | Logs "no network?", skips upgrade | Tries package installs from local cache |
| `common/packages` — upgrade | Logs failure, continues | Package installs still run |
| `common/packages` — individual install | Logs each failed package | Rest of the install list continues |
| `common/dotfiles` | Logs error, leaves existing `.bashrc` | Previous version preserved as `.bashrc~` |
| `cron` — `run-pull.sh` deploy | Logs error | Existing `run-pull.sh` stays in place; loop keeps running |
| `cron` — Termux:Boot script | Logs error | Cron job still works until next reboot |
| `obtainium` | Logs with hint to open app once | Previous `app_list.json` unchanged |
| `syncthing` | Three independent blocks (config / boot / daemon) | Each fails separately; others unaffected |
| `grapheneos` / `lineageos` | `debug` warning only | Never fatal |
| `notifications` | Logs hint about Termux:API | Never aborts |

### Idempotency

Every task is safe to re-run every hour without side effects:

- `pkg install` skips already-installed packages
- `template` and `copy` only write when content has changed
- `cron` module checks for an existing entry by name before adding
- `lineinfile` checks for the line before appending
- `file: state: directory` is a no-op if the directory exists

### Variable precedence

Ansible's precedence chain (lowest → highest) maps cleanly onto the repo layers:

```
roles/*/defaults/main.yml     ← fallback defaults, lowest priority
group_vars/all/main.yml       ← universal vars
group_vars/<os_group>.yml     ← OS-level defaults (grapheneos, lineageos, android)
host_vars/<device>.yml        ← per-device overrides, highest priority
```

A variable set in `host_vars/p8p.yml` always wins over the same variable in
`group_vars/grapheneos.yml`. This means OS-group files hold sensible defaults
and host_vars only need to list exceptions.

### Staggered cron timing

All four devices poll GitHub every hour, but not at the same minute. The cron
minute is seeded from `device_name` using Ansible's `random(seed=)` filter:

```yaml
minute: "{{ 59 | random(seed=device_name) }}"
```

This spreads the load across the hour and avoids rate-limit issues on GitHub.
The minute is deterministic — the same device always runs at the same minute —
so logs are predictable.

### Secrets

Secrets never appear in the repo in plaintext. `group_vars/all/vault.yml` is
encrypted with `ansible-vault` and the vault password lives only on each device
at `~/.ansible/.vault_pass` (written by `bootstrap.sh`, never committed).

Reference secrets in any task or template as `{{ vault_some_key }}`. To rotate
a secret, update the vault and push — devices pick it up within the hour.

### Lockfile guard in `run-pull.sh`

The hourly cron wrapper checks for a lockfile before starting. If a previous
pull is still running (e.g. a slow network), the new invocation exits immediately
rather than running two concurrent Ansible processes:

```bash
if [ -f "$LOCKFILE" ]; then
  LOCKED_PID=$(cat "$LOCKFILE")
  if kill -0 "$LOCKED_PID" 2>/dev/null; then
    echo "already running, skipping"
    exit 0
  fi
  rm -f "$LOCKFILE"   # stale lock from a crashed run
fi
```

The lockfile stores the PID so stale locks from crashed runs are detected and
cleaned up automatically.

### Retry logic in `run-pull.sh`

Mobile network connectivity is unreliable. `run-pull.sh` retries a failed
`ansible-pull` up to three times with a 60-second delay between attempts before
giving up until the next cron invocation.
