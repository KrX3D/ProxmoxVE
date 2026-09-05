# Add extended host backup script with config persistence, cron scheduling, retention, and extras

## Summary

> **Note:** The enhancements in this script were developed with the assistance of Claude AI. While the script has been tested and confirmed working on a personal Proxmox VE 9.1.6 instance, a more experienced contributor should review the code before merging.


This adds feature-complete extension of the existing `tools/pve/host-backup.sh`. The extended version replaces the minimal one-shot wizard with a persistent, menu-driven backup tool that supports saved configuration, scheduled cron jobs, retention cleanup, extra Proxmox-specific backup targets, and a full log viewer.

---

## What changed vs. the original `host-backup.sh`

The original script (~90 lines) provides a simple interactive wizard: ask for a path, pick folders, run `tar`, done. Every setting is discarded after each run and there is no scheduling, retention, or extras support.

The extended script retains the same core logic and all original features (including telemetry) and adds everything described below.

---

## New features

### Automatic self-update
On every run — interactive or `--run-config` — the script checks the version published at the upstream URL against its own. If the upstream copy is newer, it downloads it, replaces the currently running script file, and re-executes itself so that very run already uses the new version. If the check can't reach the network (offline, DNS failure, timeout) or the upstream copy isn't newer, it silently continues running the local copy — no error, no interruption. The check is bounded (3s to connect, 5s total) so a slow or dead network never stalls a run for long. This only applies when running from a real installed file the script can rewrite (e.g. `/usr/local/sbin/pve-host-backup.sh`); running via `bash <(curl ...)` has no persistent file to update, so the check is skipped.

### Persistent configuration
At the end of the wizard, the script offers to save all settings to `/etc/pve-host-backup/config.conf`. On next launch, the script detects the saved config and offers to run immediately with those settings — no need to step through the wizard every time.

### Cron scheduling
A managed cron job can be created, replaced, or removed from within the script. Three scheduling options are provided (daily, weekly, monthly) plus a custom cron expression input. Three script-source modes are available:
- **Local path** — use an already-installed copy of the script
- **Download once** — fetch the script from the upstream URL, save it locally, then run the local file
- **Always update** — re-download from the upstream URL before every cron run, then run the (possibly just-refreshed) local file either way. If the download fails, it simply runs the existing local copy instead of aborting the backup.

Since the script now checks for updates itself on every run (see [Automatic self-update](#automatic-self-update) above), mode 3 is largely redundant with mode 1 — pick **Local path** unless you specifically want the pre-run refresh to happen as a separate step before the script even starts.

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
8. **Summary** — review all selections before the backup runs
9. **Save settings** — optionally persist config for future runs and cron
10. **Cron setup** — optionally create or update a scheduled job

On subsequent runs with a saved config, steps 1–10 are skipped and the script asks directly: *use saved settings?* → *confirm and run* → done.

### Headless / cron mode

```bash
pve-host-backup.sh --run-config
```

Reads `/etc/pve-host-backup/config.conf` and runs the backup non-interactively. No terminal or whiptail required. This is what the managed cron job calls.

### Installing the script for cron use via option 4

From the **Cron management** menu, option 4 ("Update local cron script from upstream") downloads the latest version of the script to `/usr/local/sbin/pve-host-backup.sh` and makes it executable. This only needs to be done once. After that, any managed cron job already points to that path.

---

## Updating to a newer version

As of v1.1.0, **you normally don't need to do anything** — every run (interactive or `--run-config` from cron) checks the upstream URL for a newer version and, if one is found, downloads it, replaces the installed script file, and re-executes so that same run already uses it. If it can't reach the network, it just runs the local copy it already has, silently. See [Automatic self-update](#automatic-self-update) above for details.

This only applies once the script is running from a real installed file. If you're on a version older than v1.1.0 (no self-update yet), or you're running via `bash <(curl ...)` with nothing installed locally, use one of these instead:

- **From the script itself (recommended if you installed it for cron):** Main menu → **Cron management** → **Update local cron script from upstream**. This re-downloads the latest file to `/usr/local/sbin/pve-host-backup.sh` and makes it executable — safe to run any time, whether or not you actually use cron.
- **Manually, for any install location:**
  ```bash
  curl -fsSL https://raw.githubusercontent.com/KrX3D/ProxmoxVE/refs/heads/main/tools/pve/host-backup.sh -o /usr/local/sbin/pve-host-backup.sh
  chmod +x /usr/local/sbin/pve-host-backup.sh
  ```
  Adjust the destination path if you installed it somewhere other than the default.

The script's current version is shown in its title bar (e.g. `Proxmox VE Host Backup v1.1.0`) and on the **Show status and paths** screen, alongside the upstream URL it checks. Compare that against the version at the top of [`tools/pve/host-backup.sh`](tools/pve/host-backup.sh) on GitHub (or check the [commit history](https://github.com/KrX3D/ProxmoxVE/commits/main/tools/pve/host-backup.sh)) if you want to confirm a newer copy exists before it self-updates on its own.

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
