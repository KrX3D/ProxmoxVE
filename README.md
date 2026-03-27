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

### Persistent configuration
At the end of the wizard, the script offers to save all settings to `/etc/pve-host-backup/config.conf`. On next launch, the script detects the saved config and offers to run immediately with those settings — no need to step through the wizard every time.

### Cron scheduling
A managed cron job can be created, replaced, or removed from within the script. Three scheduling options are provided (daily, weekly, monthly) plus a custom cron expression input. Three script-source modes are available:
- **Local path** — use an already-installed copy of the script
- **Download once** — fetch the script from the upstream URL, save it locally, then run the local file
- **Always update** — re-download from the upstream URL before every cron run. **Note:** uses `&&` chaining, so if the download fails the backup will not run at all. Option 2 (download once) is safer for production use.

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

```bash
bash /path/to/host-backup.sh
```

Or if installed locally:

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
