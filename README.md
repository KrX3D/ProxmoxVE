# Add extended host backup script with config persistence, cron scheduling, retention, and extras

## Summary

> **Note:** The enhancements in this script were developed with the assistance of Claude AI. While the script has been tested and confirmed working on a personal Proxmox VE 9.1.6 instance, a more experienced contributor should review the code before merging.


This adds feature-complete extension of the existing `tools/pve/host-backup.sh`. The extended version replaces the minimal one-shot wizard with a persistent, menu-driven backup tool that supports saved configuration, scheduled cron jobs, retention cleanup, extra Proxmox-specific backup targets, and a full log viewer.

---

## What changed vs. the original `host-backup.sh`

The original script (~90 lines) provides a simple interactive wizard: ask for a path, pick folders, run `tar`, done. Every setting is discarded after each run and there is no scheduling, retention, or extras support.

The extended script retains the same core logic and all original features and adds everything described below. As of v1.2.0 the community-scripts telemetry hook (`api.func`) is removed: it sourced remote code as root on every run, including every cron run.

---

## New features

### Optional self-update (off by default)
With `AUTO_UPDATE='yes'` in the config (asked by the wizard, default **No**), every run checks the upstream URL for a higher `SCRIPT_VERSION`, replaces the installed script and re-executes it. This runs whatever is on `main` as root, so it is opt-in; the sha256 of each installed update is logged. The check is bounded (3s connect, 5s total) and falls back to the local copy on any error.

### Safety checks and verification (v1.2.0)
- **Unmounted share guard:** if the backup path is under `/mnt/pve/<id>`, `/mnt/<name>` or `/media/<name>`, that directory must be a mountpoint, otherwise the run fails instead of `mkdir -p` filling the root disk. Override with `REQUIRE_MOUNTPOINT='/some/mount'` or disable with `REQUIRE_MOUNTPOINT='none'` in the config.
- **Archive verification:** after `tar`, the archive is checked with `gzip -t` (for `.tar.gz`) and a full `tar -t` listing. A failed check fails the run.
- **tar exit 1** (files changed while being read) is logged as a warning, not a failure; exit 2+ fails the run and removes the partial archive.
- **Push notification:** set `NOTIFY_URL` (asked by the wizard) to an Uptime Kuma push URL. `--run-config` sends `status=up` (with any warnings in `msg`) on success and `status=down` with the error on any failure, including unexpected aborts.
- **Log rotation:** `backup.log` is rotated at 1 MiB, keeping 5 old files (`backup.log.1` … `.5`).
- **pmxcfs SQL dump on by default:** preselected in the wizard, and added to `--run-config` runs unless the config has `PMXCFS_SQL_DUMP='no'`. Needs `sqlite3` (`apt install sqlite3`); if it is missing the run succeeds with a warning.

### Persistent configuration
At the end of the wizard, the script offers to save all settings to `/etc/pve-host-backup/config.conf`. On next launch, the script detects the saved config and offers to run immediately with those settings — no need to step through the wizard every time.

### Cron scheduling
A managed cron job can be created, replaced, or removed from within the script. Three scheduling options are provided (daily, weekly, monthly) plus a custom cron expression input. Two script-source modes are available:
- **Local path** — use an already-installed copy of the script
- **Download once** — fetch the script from the upstream URL, save it locally, then run the local file

The former "Always update before each cron run" mode was removed in v1.2.0 (it executed unverified code from GitHub as root on every run). Existing cron lines are not changed; re-create the cron job from the menu to switch.

Cron output is appended to `/var/log/pve-host-backup/backup.log` instead of `/dev/null`. Cron jobs created before v1.2.0 still discard output; re-create them from the menu to pick this up.

The managed cron entry is tagged with `# PVE_HOST_BACKUP_MANAGED` so it can be identified and updated without touching other cron jobs. Run `pve-host-backup.sh --run-config` to execute a non-interactive backup from cron using the saved config.

### Retention / automatic cleanup
Old backups in the target folder can be automatically deleted after a configurable number of days. Cleanup is scoped strictly to files whose filename starts with the current hostname (e.g. `proxmox2-`) so backups from other Proxmox instances stored in the same shared folder are never touched.

### Recommended extra backup targets
After the primary directory selection, an optional extras screen offers pre-configured paths commonly needed for full Proxmox host recovery:

| Path | Purpose |
|---|---|
| `/etc/` | Main host configuration |
| `/root/` | SSH keys, scripts, notes |
| `/usr/local/` | Custom local tools |
| `/opt/` | Custom applications |
| `/var/spool/cron/` | Cron jobs |
| Root crontab export | Exports `crontab -l` to a file included in the archive |
| `/var/lib/pve-cluster/config.db` | Raw pmxcfs SQLite database |
| pmxcfs SQL dump | Safer `sqlite3 .dump` of the pmxcfs backend |

### Compression choice
Choose between `tar.gz` (smaller, slower) or plain `tar` (faster, larger) per run. The choice is saved in the config.

### Log file options
- Include the current log file inside the backup archive
- Copy the log file next to the archive with the same basename (`.log` extension)

### Full log viewer
The backup log at `/var/log/pve-host-backup/backup.log` can be viewed from the main menu using `less` with full keyboard navigation (`q` quit, arrow keys / PgUp / PgDn scroll, `g` top, `G` bottom). Falls back to a whiptail textbox if `less` is not available.

### Status and paths screen
Menu option 3 shows all relevant paths (config file, log file, runtime script path, upstream URL) and the live cron job status — schedule and script path if set, or "Not set" if no managed cron entry exists.

### Deduplication
The selected item list is deduplicated before backup and before display. A plain path (e.g. `/root/`) that is already covered by an ALL marker for the same directory is automatically dropped to avoid redundant entries in the summary and the archive.

### Dynamic summary height
The backup summary and confirmation dialogs grow in height to fit all selected items rather than clipping long lists.

---

## Setup and usage

### Interactive mode

Run directly from GitHub, no download step needed:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/KrX3D/ProxmoxVE/main/tools/pve/host-backup.sh)"
```

Or if you already have a local copy at some path:

```bash
bash /path/to/host-backup.sh
```

Or if installed locally at the default cron path:

```bash
chmod +x /usr/local/sbin/pve-host-backup.sh
pve-host-backup.sh
```

On first run the wizard steps through:
1. **Backup target** — where to write the archive (e.g. `/mnt/pve/smb-backup/host`)
2. **Working directory** — one or more source directories separated by commas (e.g. `/etc/, /root/`)
3. **Compression** — `tar.gz` or `tar`
4. **Retention** — days to keep old backups (0 = keep all)
5. **Item selection** — pick individual files/folders or select ALL for each working directory
6. **Extras** — optional recommended Proxmox paths and virtual exports
7. **Log options** — include log in archive and/or copy it next to the archive
8. **Push URL / auto-update** — optional Uptime Kuma push URL, opt-in self-update
9. **Summary** — review all selections before the backup runs
10. **Save settings** — optionally persist config for future runs and cron
11. **Cron setup** — optionally create or update a scheduled job

On subsequent runs with a saved config, steps 1–11 are skipped and the script asks directly: *use saved settings?* → *confirm and run* → done.

### Headless / cron mode

```bash
pve-host-backup.sh --run-config
```

Reads `/etc/pve-host-backup/config.conf` and runs the backup non-interactively. No terminal or whiptail required. This is what the managed cron job calls.

### Installing the script for cron use via option 4

From the **Cron management** menu, option 4 ("Update local cron script from upstream") downloads the latest version of the script to `/usr/local/sbin/pve-host-backup.sh` and makes it executable. This only needs to be done once. After that, any managed cron job already points to that path.

---

## Updating to a newer version

Since v1.2.0 updates are manual unless you set `AUTO_UPDATE='yes'` (see [Optional self-update](#optional-self-update-off-by-default)). To update:

- **From the script itself (recommended if you installed it for cron):** Main menu → **Cron management** → **Update local cron script from upstream**. This re-downloads the latest file to `/usr/local/sbin/pve-host-backup.sh` and makes it executable — safe to run any time, whether or not you actually use cron.
- **Manually, for any install location:**
  ```bash
  curl -fsSL https://raw.githubusercontent.com/KrX3D/ProxmoxVE/refs/heads/main/tools/pve/host-backup.sh -o /usr/local/sbin/pve-host-backup.sh
  chmod +x /usr/local/sbin/pve-host-backup.sh
  ```
  Adjust the destination path if you installed it somewhere other than the default.

The script's current version is shown in its title bar (e.g. `Proxmox VE Host Backup v1.1.0`) and on the **Show status and paths** screen, alongside the upstream URL it checks. Compare that against the version at the top of [`tools/pve/host-backup.sh`](tools/pve/host-backup.sh) on GitHub (or check the [commit history](https://github.com/KrX3D/ProxmoxVE/commits/main/tools/pve/host-backup.sh)) if you want to confirm a newer copy exists.

Updating the script — automatic or manual — only ever replaces the script file itself. Your saved settings (`/etc/pve-host-backup/config.conf`) and any managed cron job are untouched and continue to work with the new version.

---

## File locations

| Path | Purpose |
|---|---|
| `/etc/pve-host-backup/config.conf` | Saved backup settings |
| `/var/log/pve-host-backup/backup.log` | Timestamped backup log |
| `/usr/local/sbin/pve-host-backup.sh` | Default local install path for cron |

---

## Compatibility

- Requires: `bash`, `tar`, `whiptail`, `crontab`, `curl`
- Optional: `sqlite3` (for pmxcfs SQL dump), `less` (for log viewer)
- Tested on Proxmox VE 9.1.6
