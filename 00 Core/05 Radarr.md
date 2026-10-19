# Overview

Radarr is a movie collection manager that automates the process of finding, downloading, organizing, and renaming movies. It integrates with indexer managers such as Prowlarr and download clients such as qBittorrent to build an automated movie management workflow.

This guide covers verifying the existing installation, configuring the movie library, setting naming and file management preferences, creating quality and custom format profiles, and connecting qBittorrent as the download client.

> [!NOTE] Prerequisites
> Before proceeding, ensure that:
> - Docker Engine and Docker Compose are installed and running.
> - Radarr has already been deployed as part of the media stack.
> - Prowlarr and qBittorrent are configured, if you intend to use them.
> - The media and download directories exist and have suitable permissions.
> Refer to [01 Docker](01%20Docker.md), [03 Prowlarr and FlareSolverr](03%20Prowlarr%20and%20FlareSolverr.md), and [04 qBittorrent](04%20qBittorrent.md) for the relevant setup instructions.

## Table of Contents
- [1. Installation & Verification](#1-installation-verification)
    - [1.1. Prepare the Directories](#11-prepare-the-directories)
    - [1.2. Start or Restart Radarr](#12-start-or-restart-radarr)
    - [1.3. Access the Web Interface](#13-access-the-web-interface)
- [2. Configure Media Management](#2-configure-media-management)
    - [2.1. Add the Movie Root Folder](#21-add-the-movie-root-folder)
    - [2.2. Configure Movie Naming](#22-configure-movie-naming)
    - [2.3. Configure Importing and File Management](#23-configure-importing-and-file-management)
- [3. Configure Quality Profiles](#3-configure-quality-profiles)
- [4. Configure Custom Formats](#4-configure-custom-formats)
- [5. Configure Profiles and Custom Format Scoring](#5-configure-profiles-and-custom-format-scoring)
- [6. Configure the Download Client](#6-configure-the-download-client)
    - [6.1. Add qBittorrent](#61-add-qbittorrent)
    - [6.2. Configure Completed Download Handling](#62-configure-completed-download-handling)
    - [6.3. Test the Connection](#63-test-the-connection)
- [7. Troubleshooting](#7-troubleshooting)

---
## 1. Installation & Verification

Radarr has already been installed as part of the media stack. We only need to verify its configuration directories, ensure that the container is running, and open the Web UI.

### 1.1. Prepare the Directories

Create the directories required by Radarr if they do not already exist.

```bash
sudo mkdir -p /srv/docker/configs/radarr
sudo mkdir -p /mnt/media-hdd/downloads
sudo mkdir -p /mnt/media-hdd/media/movies
```

Set ownership to UID `1000` and GID `1000`, assuming these are the IDs configured for the relevant containers.

```bash
sudo chown -R 1000:1000 /srv/docker/configs/radarr
sudo chown -R 1000:1000 /mnt/media-hdd/downloads
sudo chown -R 1000:1000 /mnt/media-hdd/media/movies
```

Set basic directory permissions:

```bash
sudo chmod -R 775 /srv/docker/configs/radarr
sudo chmod -R 775 /mnt/media-hdd/downloads
sudo chmod -R 775 /mnt/media-hdd/media/movies
```

> [!WARNING] Verify ownership before applying changes
> UID `1000` and GID `1000` are common defaults, not universal values. Check the `PUID` and `PGID` in your Compose configuration and confirm that the selected directories are intended to be owned by those IDs.
> The recursive ownership and permission commands can affect existing files. Use them only when the resulting changes are appropriate for your server.

### 1.2. Start or Restart Radarr

Navigate to the directory containing the media stack's Compose file:

```bash
cd /srv/docker/stacks/media-stack
```

If Radarr is already running and you have not changed its configuration, you can simply restart it:

```bash
docker compose restart radarr
```

If you have modified its Compose configuration and need Docker to apply those changes, recreate the container:

```bash
docker compose up -d --force-recreate radarr
```

If you want to pull the latest image before recreating it:

```bash
docker compose pull radarr
docker compose up -d radarr
```

Check the container status:

```bash
docker ps \
  --filter "name=radarr" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

If Radarr is not running, inspect its logs:

```bash
docker logs --tail=100 radarr
```

### 1.3. Access the Web Interface

Open Radarr in your browser:

```bash
http://localhost:7878
```

If you are accessing the server from another device, replace `localhost` with the server's IP address:

```bash
http://SERVER_IP:7878
```

This assumes that host port `7878` is published in the Compose configuration.

> [!TIP] Radarr is ready
> Once the Web UI loads, we can configure the movie library and its automation settings.

---
## 2. Configure Media Management

Media Management controls how Radarr stores, names, imports, and organizes movies.

### 2.1. Add the Movie Root Folder

Navigate to:

**Settings → Media Management → Root Folders → Add Root Folder**

Select or enter:

```bash
/movies
```

This path assumes that the Compose file maps the host movie directory into the container.

For example:

```yaml
volumes:
  - ${MOVIES_DIR}:/movies
```

The `.env` file may define the corresponding host directory as:

```properties
MOVIES_DIR=/mnt/media-hdd/media/movies
```

This produces the following mapping:

| Location         | Path                          |
| ---------------- | ----------------------------- |
| Host             | `/mnt/media-hdd/media/movies` |
| Radarr container | `/movies`                     |

Radarr uses `/movies` internally, while the actual files are stored on the host under `/mnt/media-hdd/media/movies`.

> [!IMPORTANT] Use the container path
> When adding a root folder in Radarr, use `/movies`, not `/mnt/media-hdd/media/movies`, because the latter is the host path rather than the path visible inside the container.

### 2.2. Configure Movie Naming

Navigate to:

**Settings → Media Management → Movie Naming**

Enable the following options:

|Setting|Value|
|---|---|
|Rename Movies|Enabled|
|Replace Illegal Characters|Enabled|
|Colon Replacement|Smart Replace|

Configure the naming formats as follows.

**Standard Movie Format**

```yaml
{Movie Title} ({Release Year})
```

Example:

```text
Dune (2021).mkv
```

**Movie Folder Format**

```yaml
{Movie CleanTitle} ({Release Year})
```

Example:

```text
Dune (2021)/
└── Dune (2021).mkv
```

These naming formats help keep the movie library consistent and easy to browse.

> [!TIP] Why consistent naming matters
> Consistent folder and file names make it easier for media servers and other applications to identify movies, match metadata, and organize the library.

### 2.3. Configure Importing and File Management

While still in **Settings → Media Management**, configure the following options according to your preferred workflow.

#### Use Hardlinks Instead of Copy

**Setting:** `Use Hardlinks instead of Copy`
**Value:** Enabled

When hardlinks are supported, Radarr can import a completed download into the movie library without duplicating the underlying file data.

This is especially useful when downloads and the movie library reside on the same filesystem.

For example, a completed movie can remain available in the downloads directory while also appearing in the movie library, without requiring another full copy of the file.

> [!IMPORTANT] Hardlink requirements
> Hardlinks generally require the source and destination to be on the same filesystem. The directories must also be accessible to Radarr, and the underlying filesystem must support hardlinks.
> Separate bind mounts can still support hardlinks if they point to directories on the same underlying filesystem and the paths are configured appropriately. Simply using the same Docker network does not make hardlinks possible.

#### Unmonitor Deleted Movies

**Setting:** `Unmonitor Deleted Movies`
**Value:** Enabled

When enabled, Radarr can automatically unmonitor movies that have been deleted from the library, according to the application's supported behavior.

This can help prevent unwanted monitoring of movies that you have intentionally removed.

---
## 3. Configure Quality Profiles

Quality profiles determine the size of a torrent that Radarr considers acceptable for a movie.

They allow you to choose preferred size for each release type of a movie's quality (720p, 1080p, 2160p, etc).

Navigate to:

**Settings → Quality**

The quality profiles used in this setup are documented separately in: [03 Media Qualities](../01%20Configuration/03%20Media%20Qualities.md).

Refer to that document for the existing configuration and use it as a starting point.


> [!TIP] Customize profiles to your needs
> The profiles in this setup reflect personal preferences. You do not need to reproduce every profile exactly. Create only the profiles you expect to use and adjust their quality ordering and upgrade behavior to suit your storage capacity and viewing preferences.

---
## 4. Configure Custom Formats

Custom Formats allow Radarr to evaluate release characteristics that are not captured by quality alone.

For example, a custom format can help distinguish releases based on:
- HDR or Dolby Vision characteristics.
- Audio formats.
- Video codecs.
- Release editions.
- Other supported attributes found in release names or metadata.

Navigate to:

**Settings → Custom Formats**

The custom formats used in this setup are documented in: [04 Custom Formats](../01%20Configuration/04%20Custom%20Formats.md).

Use that document as the reference when recreating the configuration.

Some custom formats may use regular expressions to identify specific release-name patterns. When the documentation provides a regular expression, copy the exact expression into the relevant condition field.

> [!IMPORTANT] Verify each condition
> Custom format conditions and regular expressions can behave differently depending on the fields and matching rules used. Test the configuration against representative release names to confirm that it matches the intended releases without unintentionally matching unrelated ones.

Custom Formats are particularly useful when combined with quality profiles and custom format scoring, which determine how Radarr ranks otherwise acceptable releases.

---
## 5. Configure Profiles and Custom Format Scoring

Custom format scoring determines how strongly Radarr prefers or rejects releases that match particular custom formats.

Navigate to:

**Settings → Profiles**

Open each relevant quality profile and configure its custom format scores.

The profiles used in this setup are documented in [05 Release Profiles](../01%20Configuration/05%20Release%20Profiles.md).

Use that document as a reference, but feel free to modify the scores and settings.

For example, you may assign higher scores to releases with your preferred video codec or HDR format, while assigning lower scores to releases that do not meet your preferences.

Review the following for each profile:
- Which qualities are allowed.
- Whether quality upgrades are enabled.
- The upgrade cutoff.
- Which custom formats are assigned scores.
- The minimum custom format score, if used.
- Whether the scoring behavior matches your intended release preferences.

> [!NOTE] You do not need every profile
> The configuration is designed to provide flexibility for different preferences. You can simplify it to a few profiles rather than reproducing every available option.
> Be careful with negative scores and minimum score requirements, as these can cause otherwise acceptable releases to be rejected.

---
## 6. Configure the Download Client

Radarr needs a download client to retrieve movies after finding suitable releases.

In this setup, qBittorrent is the download client.

### 6.1. Add qBittorrent

Navigate to:

**Settings → Download Clients → Add → qBittorrent**

Configure the connection using the following reference values.

| Setting         | Value                            |
| --------------- | -------------------------------- |
| Name            | `qBittorrent`                    |
| Host            | `qbittorrent` or `gluetun`       |
| Port            | `8080`                           |
| Username        | Your qBittorrent Web UI username |
| Password        | Your qBittorrent Web UI password |
| Category        | `radarr`                         |
| Client Priority | `1`                              |

The host value `qbittorrent` assumes that the qBittorrent service is named `qbittorrent` and that Radarr and qBittorrent share a Docker network that supports service-name resolution.

If you have routed QBittorrent through Gluetun, use `gluetun` in the **Host** field. 

The port `8080` assumes that the qBittorrent Web UI listens on that port inside the container.

Replace it if your configuration uses a different port.

#### Docker networking options

Depending on how your media stack is configured, the host may be:
- `qbittorrent` if Radarr can reach qBittorrent directly over a shared Docker network.
- `gluetun` if qBittorrent shares Gluetun's network namespace and the required Web UI port is reachable through Gluetun.
- Another reachable hostname or IP address if your network configuration requires it.

> [!WARNING] Avoid hardcoded container IP addresses
> An address such as `172.18.0.1` is not a universal qBittorrent address. It may refer to a Docker bridge gateway or another network interface, depending on the environment.
> Prefer stable service names on a shared Docker network. If you must use an IP address, verify that it identifies the correct endpoint and is stable across container recreation.

If qBittorrent shares Gluetun's network namespace, ensure that the Web UI port is configured and exposed appropriately. Do not assume that the qBittorrent container has its own independently reachable network address.

### 6.2. Configure Completed Download Handling

Review the following settings in Radarr.

|Setting|Value|
|---|---|
|Completed Download Handling|Enabled|
|Remove Completed Downloads|Enabled, if desired|
|Remove Failed Downloads|Enabled, if desired|
|Redownload Failed|Enabled|
|Redownload Failed from Interactive Search|Enabled|

The exact options and their locations may vary slightly between Radarr versions.

#### Completed Download Handling

When enabled, Radarr monitors completed downloads and imports them into the configured movie library when they can be matched and processed successfully.

#### Remove Completed Downloads

This setting controls whether Radarr asks the download client to remove completed downloads after Radarr has finished processing them, subject to the client's supported behavior and the state of the download.

If you want to retain completed torrents for seeding, review this setting carefully before enabling automatic removal.

#### Remove Failed Downloads

This setting controls how Radarr handles failed downloads and whether it attempts to remove them through the download client.

Use it according to your preferred cleanup workflow.

#### Redownload Failed

When enabled, Radarr can search for replacement releases when a download fails, subject to the availability of alternatives and your profile settings.

#### Redownload Failed from Interactive Search

When enabled, Radarr can apply the relevant failed-download behavior to releases initiated through interactive searches.

> [!IMPORTANT] Seeding and removal behavior
> Download-client removal and seeding policies should work together. If qBittorrent is configured to retain torrents for seeding, make sure Radarr's completed-download handling does not remove them earlier than intended.
> Also, ensure that qBittorrent's categories and save paths are compatible with Radarr's import workflow.

### 6.3. Test the Connection

After configuring qBittorrent:
1. Click **Test**.
2. Confirm that Radarr can communicate with qBittorrent.
3. Save the configuration.
4. Verify that the configured category matches the intended movie download workflow.
5. Confirm that Radarr can access the completed download paths.

If the test fails, check the following:
- qBittorrent is running.
- The host and port are correct.
- The username and password are valid.
- The Web UI is reachable from Radarr.
- The Docker network configuration is correct.
- Any Gluetun port mappings or network namespace settings are correct.

---
## 7. Troubleshooting

### Radarr's Web UI does not load

Check the container status:

```bash
docker ps --filter "name=radarr"
```

Inspect the logs:

```bash
docker logs --tail=100 radarr
```

Confirm that the expected host port is published and that the container is running.

### Radarr cannot access the movie root folder

Inspect the host directory:

```bash
ls -ld /mnt/media-hdd/media/movies
```

Verify that the directory is mounted into Radarr at `/movies` and that the container user has the required permissions.

### Radarr cannot connect to qBittorrent

Check that qBittorrent is running and that the Web UI is reachable from Radarr.

Inspect the network attachments if necessary:

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' radarr | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' qbittorrent | jq
```

If qBittorrent shares Gluetun's network namespace, inspect Gluetun's networking configuration instead of expecting qBittorrent to have an independent network address.

### Downloads complete but movies are not imported

Verify the following:
- Completed Download Handling is enabled.
- The download category is correct.
- Radarr can access the download directory.
- The download and library paths are mounted consistently.
- The files have appropriate permissions.
- Radarr can identify the completed release.

If you intend to use hardlinks, confirm that the download and library directories are on the same underlying filesystem.

### Custom format scores are not producing the expected results

Check the custom format conditions and profile scores.

Review the release details and custom format matches displayed by Radarr, then confirm that the minimum score and upgrade cutoff are configured as intended.

---

> [!TIP] Radarr configuration complete
> Radarr is ready to manage the movie library and automate the download and import workflow. Once the indexers, quality profiles, custom formats, and download client are configured correctly, Radarr can search for suitable releases and organize completed downloads in the movie library.

---

> [!TIP] ➜ Up Next
> You’ve completed  [05 Radarr](#05-radarr)
> **Previous:** [04 qBittorrent](04%20qBittorrent.md)  ------ **Next:** [06 Sonarr](06%20Sonarr.md)
