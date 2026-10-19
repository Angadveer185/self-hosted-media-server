# Overview

Backups protect your media server's configuration, Docker Compose files, and application data from accidental deletion, configuration mistakes, and system failures.

This guide creates a Bash script that backs up your media server's configuration, excludes selected Jellyfin cache files, retains the seven most recent archives, and runs automatically through cron.

> [!NOTE] Backup Overview
> - **Backup directory:** `/srv/docker/backups`
> - **Script:** `/srv/docker/backups/backup-media-server.sh`
> - **Compose directory:** `/srv/docker/stacks/media-stack`
> - **Configuration directory:** `/srv/docker/configs`
> - **Schedule:** Every Saturday, at 00:10
> - **Retention:** Seven most recent successful backup archives

## Table of Contents
- [1. Prepare the Backup Directory](#1-prepare-the-backup-directory)
- [2. Create the Backup Script](#2-create-the-backup-script)
- [3. Make the Script Executable](#3-make-the-script-executable)
- [4. Run a Test Backup](#4-run-a-test-backup)
- [5. Verify the Backup Archive](#5-verify-the-backup-archive)
- [6. Schedule Automatic Backups](#6-schedule-automatic-backups)
- [7. Restore From a Backup](#7-restore-from-a-backup)
- [8. Troubleshooting](#8-troubleshooting)

---
## 1. Prepare the Backup Directory

Create a directory to store backup archives and the backup script.

```bash
# Create the backup directory
sudo mkdir -p /srv/docker/backups

# Set ownership to the user and group with UID/GID 1000
sudo chown 1000:1000 /srv/docker/backups

# Restrict access to the directory owner
sudo chmod 700 /srv/docker/backups
```

> [!IMPORTANT] Protect Your Backups
> The backup contains your `.env` file, which may include VPN credentials, API keys, and other secrets. Restrict access to the archive files and keep backups out of public repositories.
> The commands above assume the user running the backup script has UID `1000`. Confirm this with `id` before proceeding.

---
## 2. Create the Backup Script

Create the script file:

```bash
nano /srv/docker/backups/backup-media-server.sh
```

Paste the following script:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
umask 077

# --------------------------------------------------
# Configuration
# --------------------------------------------------

BACKUP_DIR="/srv/docker/backups"
SOURCE_ROOT="/srv/docker"

STACK_DIR="$SOURCE_ROOT/stacks/media-stack"
CONFIG_DIR="$SOURCE_ROOT/configs"

TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

BACKUP_FILE="$BACKUP_DIR/media-server-backup-$TIMESTAMP.tar.gz"
TEMP_FILE="$BACKUP_FILE.tmp"

# --------------------------------------------------
# Prepare and Validate
# --------------------------------------------------

mkdir -p "$BACKUP_DIR"

# Ensure required source files and directories exist.
if [[ ! -f "$STACK_DIR/docker-compose.yml" ]]; then
    echo "ERROR: Docker Compose file not found: $STACK_DIR/docker-compose.yml" >&2
    exit 1
fi

if [[ ! -f "$STACK_DIR/.env" ]]; then
    echo "ERROR: Environment file not found: $STACK_DIR/.env" >&2
    exit 1
fi

if [[ ! -d "$CONFIG_DIR" ]]; then
    echo "ERROR: Configuration directory not found: $CONFIG_DIR" >&2
    exit 1
fi

# Remove an incomplete temporary archive if the script fails.
trap 'rm -f -- "$TEMP_FILE"' EXIT

# --------------------------------------------------
# Create Backup
# --------------------------------------------------

echo "Starting media server backup..."
echo "Destination: $BACKUP_FILE"

tar \
    --exclude='configs/jellyfin/cache/transcodes' \
    --exclude='configs/jellyfin/cache/transcodes/*' \
    --exclude='configs/jellyfin/cache/temp' \
    --exclude='configs/jellyfin/cache/temp/*' \
    -czf "$TEMP_FILE" \
    -C "$SOURCE_ROOT" \
    "stacks/media-stack/docker-compose.yml" \
    "stacks/media-stack/.env" \
    "configs"

# Verify the archive can be read before keeping it.
tar -tzf "$TEMP_FILE" > /dev/null

# Set restrictive permissions and finalize the archive.
chmod 600 "$TEMP_FILE"
mv -- "$TEMP_FILE" "$BACKUP_FILE"

echo "Backup created successfully: $BACKUP_FILE"

# --------------------------------------------------
# Retention Policy
# Keep only the 7 newest backup archives.
# --------------------------------------------------

mapfile -t BACKUPS < <(
    find "$BACKUP_DIR" \
        -maxdepth 1 \
        -type f \
        -name 'media-server-backup-*.tar.gz' \
        -printf '%T@ %p\n' |
        sort -nr |
        cut -d ' ' -f 2-
)

if (( ${#BACKUPS[@]} > 7 )); then
    for OLD_BACKUP in "${BACKUPS[@]:7}"; do
        echo "Removing old backup: $OLD_BACKUP"
        rm -f -- "$OLD_BACKUP"
    done
fi

echo "Backup process completed."
```

Save the file and exit Nano.

### How the Script Works

| Component            | Purpose                                                                      |
| -------------------- | ---------------------------------------------------------------------------- |
| `set -Eeuo pipefail` | Makes the script stop on most command failures and unset variables           |
| `umask 077`          | Restricts permissions on newly created files                                 |
| `TIMESTAMP`          | Generates a unique timestamp for the archive filename                        |
| Validation checks    | Ensures the required Compose file, `.env`, and configuration directory exist |
| `tar -czf`           | Creates a compressed `.tar.gz` archive                                       |
| `--exclude`          | Skips selected Jellyfin cache directories                                    |
| `tar -tzf`           | Checks that the generated archive can be listed                              |
| Temporary archive    | Prevents incomplete archives from being treated as successful backups        |
| `mapfile` and `sort` | Identify backup archives from newest to oldest                               |
| Retention loop       | Deletes archives beyond the seven most recent                                |

> [!NOTE] What Is Included?
> This script backs up:
> - `/srv/docker/stacks/media-stack/docker-compose.yml`
> - `/srv/docker/stacks/media-stack/.env`
> - `/srv/docker/configs/`
> It does **not** back up your actual movie, TV, anime, or download files. It also excludes the specified Jellyfin cache paths. Other cache data may still be included if it lives elsewhere under `/srv/docker/configs`.

> [!WARNING] Review the Exclusion Paths
> The exclusions assume Jellyfin's configuration is stored at `/srv/docker/configs/jellyfin` and that the cache directories use the paths shown. Verify your actual directory structure before relying on the script.

---
## 3. Make the Script Executable

Grant the script executable permissions:

```bash
chmod 750 /srv/docker/backups/backup-media-server.sh
```

Verify the permissions:

```bash
ls -l /srv/docker/backups/backup-media-server.sh
```

The script should be executable by its owner. Run it as the user that owns the backup directory and has read access to the source files.

---
## 4. Run a Test Backup

Execute the script manually:

```bash
/srv/docker/backups/backup-media-server.sh
```

A successful run should display messages similar to:

```bash
Starting media server backup...
Destination: /srv/docker/backups/media-server-backup-2026-10-18_00-10-00.tar.gz
Backup created successfully: /srv/docker/backups/media-server-backup-2026-10-18_00-10-00.tar.gz
Backup process completed.
```

The date and time will reflect when the script runs.

> [!IMPORTANT] Do Not Ignore Errors
> If the script fails, investigate the error before scheduling it. Common causes include incorrect ownership, unreadable configuration files, missing source paths, and insufficient disk space.

---
## 5. Verify the Backup Archive

### 5.1 List the Backup Files

Run:

```bash
ls -lh /srv/docker/backups/
```

Example output:

```bash
-rwxr-x--- 1 user user  3.2K Oct 18 00:10 backup-media-server.sh
-rw------- 1 user user  250M Oct 18 00:10 media-server-backup-2026-10-18_00-10-00.tar.gz
```

The archive size depends on the number and size of your application configuration files. The example is illustrative, not an expected size.

### 5.2 Inspect the Archive Contents

List the files inside the archive:

```bash
tar -tzf /srv/docker/backups/media-server-backup-YYYY-MM-DD_HH-MM-SS.tar.gz | less
```

Replace the example filename with the actual archive name.

You should see entries resembling:

```bash
stacks/media-stack/docker-compose.yml
stacks/media-stack/.env
configs/
configs/jellyfin/
configs/radarr/
configs/sonarr/
configs/prowlarr/
configs/qbittorrent/
configs/gluetun/
configs/jellyseerr/
```

The exact entries depend on which applications are present in your configuration directory.

### 5.3 Verify the Archive Integrity

Run:

```bash
gzip -t /srv/docker/backups/media-server-backup-YYYY-MM-DD_HH-MM-SS.tar.gz
```

If the command returns without an error, the gzip stream passed its integrity check.

You can also test whether `tar` can list the archive:

```bash
tar -tzf /srv/docker/backups/media-server-backup-YYYY-MM-DD_HH-MM-SS.tar.gz > /dev/null \
  && echo "Archive listing successful"
```

These checks are useful, but they do not prove that every application can be restored successfully. Test restoration periodically.

---
## 6. Schedule Automatic Backups

We will use cron to run the backup script automatically.

### 6.1 Open Your User's Crontab

Run:

```bash
crontab -e
```

This edits the crontab for your current user. Make sure that user has permission to read the source files and write to the backup directory.

### 6.2 Add the Daily Backup Schedule

Add the following line at the bottom:

```bash
10 0 * * 6 /srv/docker/backups/backup-media-server.sh >> /srv/docker/backups/backup.log 2>&1
```

This runs the backup every Saturday at 00:10 and appends standard output and error output to `backup.log`.

> [!TIP] Why Log the Output?
> Without logging, a scheduled backup can fail silently. The log gives you a record of whether the script ran and whether it reported an error.

### 6.3 Understand the Cron Schedule

Cron uses five time fields:

| Field        | Value | Meaning                |
| ------------ | ----- | ---------------------- |
| Minute       | `10`  | At minute 10           |
| Hour         | `0`   | At midnight hour       |
| Day of month | `*`   | Every day of the month |
| Month        | `*`   | Every month            |
| Day of week  | `6`   | Every Saturday         |

Therefore, `10 0 * * *` means **every Saturday at 00:10**, according to the server's local time.

### 6.4 Verify the Crontab Entry

Save the file, then run:

```bash
crontab -l
```

Confirm that the daily schedule appears exactly once.

Check the server's timezone if the job runs at an unexpected time:

```bash
timedatectl
```

### 6.5 Check the Backup Log

After the scheduled run, inspect the log:

```bash
tail -n 100 /srv/docker/backups/backup.log
```

You can also verify the most recent archive:

```bash
ls -lt /srv/docker/backups/media-server-backup-*.tar.gz | head
```

> [!IMPORTANT] Verify Scheduled Runs
> Do not assume cron is working simply because the entry exists. Confirm that a new archive is created and that the log shows a successful run. The cron environment can differ from your interactive shell, so use absolute paths as shown.

---
## 7. Restore From a Backup

A backup is only useful if you can restore it. The following steps provide a basic way to inspect and extract an archive.

### 7.1 Choose the Correct Archive

List the available archives:

```bash
ls -lh /srv/docker/backups/media-server-backup-*.tar.gz
```

Select the archive you want to restore.

### 7.2 Inspect Before Extracting

List the archive contents:

```bash
tar -tzf /srv/docker/backups/media-server-backup-YYYY-MM-DD_HH-MM-SS.tar.gz | less
```

Verify that it contains the Compose file, `.env`, and application configuration directories you need.

### 7.3 Extract to a Temporary Directory

Before overwriting live configuration, extract the archive to a separate location:

```bash
mkdir -p "$HOME/media-server-restore-test"

tar -xzf /srv/docker/backups/media-server-backup-YYYY-MM-DD_HH-MM-SS.tar.gz \
  -C "$HOME/media-server-restore-test"
```

Inspect the extracted files before attempting a real restore.

> [!WARNING] Restoring Live Configuration
> Do not blindly extract a backup over a running server. Stop affected services when appropriate, preserve the current configuration, and verify file ownership and permissions before bringing services back online. A complete disaster recovery procedure should also cover Docker installation, environment variables, external storage, and any data not included in this archive.

---
## 8. Troubleshooting

### 8.1 Permission Denied

Check the backup directory:

```bash
ls -ld /srv/docker/backups
```

Check the script's owner and permissions:

```bash
ls -l /srv/docker/backups/backup-media-server.sh
```

Make sure the user running the script can read the Compose file, `.env`, and application configurations, and can write to the backup directory.

### 8.2 No Backup Archive Was Created

Run the script manually and read the error message:

```bash
/srv/docker/backups/backup-media-server.sh
```

Check the required source files:

```bash
ls -l /srv/docker/stacks/media-stack/docker-compose.yml
ls -l /srv/docker/stacks/media-stack/.env
ls -ld /srv/docker/configs
```

Also check available disk space:

```bash
df -h /srv/docker
```

### 8.3 Cron Did Not Run

Verify the crontab:

```bash
crontab -l
```

Inspect the backup log:

```bash
tail -n 100 /srv/docker/backups/backup.log
```

Check that the cron service is running on Ubuntu:

```bash
systemctl status cron
```

If necessary, enable it:

```bash
sudo systemctl enable --now cron
```

### 8.4 Backups Are Taking Too Much Space

Check the size of the backup directory:

```bash
du -sh /srv/docker/backups
```

The script retains seven archives, but configuration directories can still grow substantially. Review which application data must be included and whether any additional cache directories can safely be excluded.

Do not exclude application databases or configuration files without understanding the effect on restoration.

### 8.5 The Archive Contains Sensitive Information

The `.env` file may contain secrets, and application configuration directories can contain credentials or API keys.

Keep the backup directory private, restrict archive permissions, and store any off-site copies in an appropriately protected location. If an archive is accidentally exposed, rotate the affected credentials.

---

> [!TIP] Backup Automation Complete
> Your media server configuration now has a scheduled backup process with archive verification and automatic retention. For stronger disaster recovery, keep an additional copy on a separate disk or remote storage location, and periodically test restoring your configuration.

---

> [!TIP] ➜ Up Next
> You’ve completed  [01 Backup](#01-backup)
> **Previous:** [00 Hosting](00%20Hosting.md)  ------ **Next:** [02 Tailscale](02%20Tailscale.md)
