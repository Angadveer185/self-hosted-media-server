# Overview

Jellyfin is the media server that brings your movies, TV shows, and anime together in one place. It scans your media library, retrieves metadata and artwork, manages users, and streams content to your devices.

This guide covers the initial setup, library configuration, metadata preferences, playback settings, and recommended plugins for the existing Docker-based media server.

> [!NOTE] Overview
> - **Web UI:** `http://localhost:8096`
> - **Docker container:** `jellyfin`
> - **Configuration directory:** `/srv/docker/configs/jellyfin`
> - **Movies:** `/mnt/media-hdd/media/movies`
> - **TV Shows:** `/mnt/media-hdd/media/tv`
> - **Anime:** `/mnt/media-hdd/media/anime`
> - **Container library paths:** `/media/movies`, `/media/tv`, `/media/anime`

## Table of Contents
- [1. Installation Checks](#1-installation-checks)
- [2. First-Time Setup](#2-first-time-setup)
- [3. Dashboard Settings](#3-dashboard-settings)
    - [3.1 Branding](#31-branding)
    - [3.2 Users](#32-users)
    - [3.3 Libraries](#33-libraries)
    - [3.4 NFO Settings](#34-nfo-settings)
    - [3.5 Playback and Transcoding](#35-playback-and-transcoding)
    - [3.6 Plugins](#36-plugins)
- [4. Troubleshooting](#4-troubleshooting)

---
## 1. Installation Checks

Jellyfin is already installed, so we only need to verify the directories, permissions, Docker configuration, and container status before opening the Web UI.

### 1.1 Create the Required Directories

Run the following commands on the Ubuntu host:

```bash
# Create Jellyfin configuration directory
sudo mkdir -p /srv/docker/configs/jellyfin

# Create download directories
sudo mkdir -p /mnt/media-hdd/downloads

# Create media library directories
sudo mkdir -p /mnt/media-hdd/media/{movies,tv,anime}
```

> [!WARNING] Existing Directory Structure
> If your current download folders are located directly under `/srv/docker/downloads`, do not reorganize them just to match this example. Your existing qBittorrent, Radarr, and Sonarr paths must remain consistent with the Docker Compose volume mappings.

### 1.2 Set Ownership and Permissions

The media stack uses `PUID=1000` and `PGID=1000`. If Jellyfin also runs under this UID and GID, the following commands can be used:

```bash
# Set ownership
sudo chown -R 1000:1000 /srv/docker/configs/jellyfin
sudo chown -R 1000:1000 /mnt/media-hdd/downloads
sudo chown -R 1000:1000 /mnt/media-hdd/media/{movies,tv,anime}

# Set basic permissions
sudo chmod -R 775 /srv/docker/configs/jellyfin
sudo chmod -R 775 /mnt/media-hdd/downloads
sudo chmod -R 775 /mnt/media-hdd/media/{movies,tv,anime}
```

The downloads directory is managed by qBittorrent, so avoid recursively changing its ownership unless you have confirmed that the current permissions are incorrect.

> [!IMPORTANT] Avoid Unnecessary Recursive Changes
> Do not repeatedly run `chown -R` or `chmod -R` against a large media library. These commands can take a long time and may alter permissions that other containers depend on.
> Jellyfin needs read access to media files and write access to its configuration directory. It only needs write access to the media library if you intend to allow it to modify media files or save metadata and subtitles there.

### 1.3 Verify the Docker Compose Configuration

Before starting Jellyfin, confirm that its service definition in `/srv/docker/stacks/media-stack/docker-compose.yml` has the correct volume mappings.

For example:

```yaml
services:
  jellyfin:
    volumes:
      - ${CONFIG_ROOT}/jellyfin:/config
      - ${MEDIA_ROOT}:/media
```

These are example volume entries, not a complete service definition. Preserve any existing ports, environment variables, devices, networks, and other settings in your Compose file.

Check that the service exists and the Compose file is valid:

```bash
cd /srv/docker/stacks/media-stack # This is what I used, might be different for you.

# Validate the Compose configuration
sudo docker compose config --quiet

# Check the current Jellyfin container
sudo docker ps -a --filter name=jellyfin
```

### 1.4 Restart Jellyfin

```bash
cd /srv/docker/stacks/media-stack

# Pull the latest image configured in Compose
sudo docker compose pull jellyfin

# Recreate and start Jellyfin
sudo docker compose up -d jellyfin

# Check the container status
sudo docker compose ps jellyfin

# Inspect recent logs if needed
sudo docker compose logs --tail=100 jellyfin
```

### 1.5 Open the Web UI

Open the following address in a browser on the server:

`http://localhost:8096`

If you are accessing Jellyfin from another device on the same network, use the Ubuntu server's LAN IP address instead:

`http://SERVER_LAN_IP:8096`

For example, replace `SERVER_LAN_IP` with the actual IP address of your server.

---
## 2. First-Time Setup

When you open Jellyfin for the first time, the Welcome Wizard guides you through the initial configuration.

### 2.1 Welcome to Jellyfin

Configure the basic server settings:

|Setting|Recommended Value|
|---|---|
|Server Name|A name of your choice, such as `Media Server`|
|Preferred Display Language|English|

The server name is the name displayed to clients when they connect to Jellyfin.

### 2.2 Create an Administrator Account

Create your initial Jellyfin user.

|Setting|Value|
|---|---|
|Username|Your preferred username|
|Password|A strong, unique password|

Keep the administrator credentials private. Additional accounts can be created later for family members or other users.

### 2.3 Add Your Media Libraries

Create the following libraries. The folder paths below refer to the **paths inside the Jellyfin container**, not the host filesystem.

#### Movies

|Setting|Value|
|---|---|
|Content Type|Movies|
|Display Name|`Movies`|
|Folder|`/media/movies`|

#### TV Shows

|Setting|Value|
|---|---|
|Content Type|Shows|
|Display Name|`TV Shows`|
|Folder|`/media/tv`|

#### Anime

|Setting|Value|
|---|---|
|Content Type|Shows|
|Display Name|`Anime`|
|Folder|`/media/anime`|

For anime, use the **Shows** content type when your anime is organized into series and episodes. If you maintain a separate library of anime movies, you can create an additional Movies library.

Skip the advanced library options for now. We will configure metadata, artwork, subtitles, and other preferences later.

> [!WARNING] Container Paths Must Match
> Jellyfin cannot browse `/mnt/media-hdd/media/movies` unless that host directory is mounted into the container. If you use different volume mappings in Docker Compose, enter the corresponding container paths in the wizard.

### 2.4 Configure Remote Access

The wizard asks whether to allow remote connections to the server.

Choose the option that matches how you intend to use Jellyfin. Remote access can be configured later, so you do not need to expose the server to the internet during initial setup.

> [!IMPORTANT] Remote Access Security
> Enabling remote access in Jellyfin does not, by itself, configure your router, firewall, reverse proxy, or HTTPS. Avoid exposing Jellyfin directly to the public internet without understanding the security implications. For remote access, consider a properly secured reverse proxy or a private VPN.

### 2.5 Finish Setup

Complete the wizard and sign in using the administrator account you created.

Jellyfin should now display your libraries. Depending on the size of your collection and metadata availability, the initial scan may take some time.

---
## 3. Dashboard Settings

Open the Jellyfin Dashboard through the user menu or administration menu. The exact labels and locations may vary slightly between Jellyfin versions.

The settings below are recommended starting points. Some features require compatible plugins, and some options may be unavailable depending on your installed version.

### 3.1 Branding

Navigate to:

`Dashboard → General → Branding`

#### Custom CSS

You can customize the Jellyfin interface with custom CSS. One theme you can try is ElegantFin.

Add the following CSS import to the **Custom CSS** field:

```css
@import url("https://cdn.jsdelivr.net/gh/lscambo13/ElegantFin@main/Theme/ElegantFin-jellyfin-theme-build-latest-minified.css");
```

For more theme options, see the community-maintained collection:
- [Awesome Jellyfin: Themes](https://github.com/awesome-jellyfin/awesome-jellyfin/blob/main/THEMES.md)

> [!WARNING] External CSS
> This stylesheet is loaded from an external CDN. It may stop working if the upstream project changes its files, and it can affect the appearance of the interface after an update. If the theme stops loading, check the project's documentation and current stylesheet URL.

### 3.2 Users

Navigate to the user management section of the Dashboard.

If you plan to share the server with other people, create a separate account for each person rather than sharing the administrator account.

Recommended practices:
- Use a unique username and password for each user.
- Restrict access to libraries that a user should not see.
- Disable administrative permissions for ordinary users.
- Review remote-access permissions according to your setup.
- Use parental controls or content restrictions where appropriate.

Separate accounts provide a cleaner experience and make it easier to manage permissions.

### 3.3 Libraries

Navigate to:

`Dashboard → Libraries`

Library settings control metadata retrieval, artwork, subtitles, and media scanning. Configure Movies, TV Shows, and Anime separately where appropriate.

> [!NOTE] Provider Availability
> Metadata and image providers can change their availability, authentication requirements, and supported features. If a provider is missing from your installation or fails to retrieve information, check its current documentation and the Jellyfin logs.

#### A. TV Shows

Open the library settings for **TV Shows**.

**General settings**

|Setting|Recommended Value|
|---|---|
|Prefer embedded titles over filenames|No|
|Prefer embedded episode information over filenames|No|
|Embedded subtitle handling|Allow All|
|Real-time monitoring|Yes|
|Automatically refresh metadata from the internet|Every 30 days|

Keeping filenames as the preferred source for identification can help maintain consistent naming when your library is managed by Sonarr.

**Metadata downloaders: TV Shows**
- TheTVDB, if available and configured

**Metadata downloaders: Seasons**
- TheTVDB, if available

**Metadata downloaders: Episodes**
- TheTVDB, if available

**Metadata savers**
- NFO, if available

**Image fetchers: TV Shows**
- TheTVDB, if available
- Fanart, if available

**Image fetchers: Seasons**
- TheTVDB, if available
- Fanart, if available

**Image fetchers: Episodes**
- TheTVDB, if available
- Embedded Image Extractor, if available
- Screen Grabber, if available

**Additional options**

|Setting|Recommended Value|
|---|---|
|Automatically merge series spread across multiple folders|Yes, if your folder organization requires it|
|Media segment providers|SkipMe.db and Intro Skipper, if installed and available|
|Enable trickplay image extraction|Yes, if desired|
|Enable chapter image extraction|Yes, if desired|

Trickplay and chapter images can improve browsing by showing preview images during playback and at chapter points. However, generating them can consume CPU time, storage, and disk I/O, particularly during the initial scan.

**Subtitle downloads**

|Setting|Recommended Value|
|---|---|
|Download Language|English|
|Subtitle Downloader|OpenSubtitles, if available and configured|
|Save subtitles into media folders|Yes, if you want subtitle files stored beside the media|

Subtitle downloaders may require a plugin, an account, API credentials, or other configuration. Saving subtitles into media folders also requires write permissions for Jellyfin.

#### B. Movies

Open the library settings for **Movies**.

**General settings**

|Setting|Recommended Value|
|---|---|
|Prefer embedded titles over filenames|No|
|Embedded subtitle handling|Allow All|
|Real-time monitoring|Yes|
|Automatically refresh metadata from the internet|Every 30 days|

**Metadata downloaders: Movies**
- The Open Movie Database (OMDb), if available and configured

**Metadata savers**
- NFO, if available

**Image fetchers: Movies**
- Fanart, if available
- The Open Movie Database, if available
- Embedded Image Extractor, if available
- Screen Grabber, if available

**Additional options**

|Setting|Recommended Value|
|---|---|
|Media segment providers|SkipMe.db and Intro Skipper, if installed and available|
|Enable trickplay image extraction|Yes, if desired|
|Enable chapter image extraction|Yes, if desired|

**Subtitle downloads**

|Setting|Recommended Value|
|---|---|
|Download Language|English|
|Subtitle Downloader|OpenSubtitles, if available and configured|
|Save subtitles into media folders|Yes, if desired|

> [!TIP] Metadata Provider Selection
> Jellyfin's built-in providers and plugin-based providers are not always interchangeable. Select the providers that are actually available in your installation, and verify whether they require API keys or separate plugin installation.

#### C. Anime

Open the library settings for **Anime**.

If your anime is stored as series and episodes, configure it similarly to the TV Shows library.

**General settings**

|Setting|Recommended Value|
|---|---|
|Prefer embedded titles over filenames|No|
|Prefer embedded episode information over filenames|No|
|Embedded subtitle handling|Allow All|
|Real-time monitoring|Yes|
|Automatically refresh metadata from the internet|Every 30 days|

**Metadata downloaders: TV Shows**
- TheTVDB, if available and configured

**Metadata downloaders: Seasons**
- TheTVDB, if available

**Metadata downloaders: Episodes**
- TheTVDB, if available

**Metadata savers**
- NFO, if available

**Image fetchers: TV Shows**
- TheTVDB, if available
- Fanart, if available

**Image fetchers: Seasons**
- TheTVDB, if available
- Fanart, if available

**Image fetchers: Episodes**
- TheTVDB, if available
- Embedded Image Extractor, if available
- Screen Grabber, if available

**Additional options**

|Setting|Recommended Value|
|---|---|
|Automatically merge series spread across multiple folders|Yes, if needed|
|Media segment providers|SkipMe.db and Intro Skipper, if installed and available|
|Enable trickplay image extraction|Yes, if desired|
|Enable chapter image extraction|Yes, if desired|

**Subtitle downloads**

|Setting|Recommended Value|
|---|---|
|Download Language|English|
|Subtitle Downloader|OpenSubtitles, if available and configured|
|Save subtitles into media folders|Yes, if desired|

> [!TIP] Anime Metadata
> Anime metadata can be inconsistent across providers, especially for alternate titles, specials, and season numbering. If Jellyfin identifies a series incorrectly, verify the folder and episode naming in Sonarr and consider adjusting the metadata provider or matching the series manually.

#### D. NFO Settings

If your Jellyfin version exposes these options, configure them under the relevant metadata or NFO settings.

|Setting|Recommended Value|
|---|---|
|Save image paths within NFO files|Yes|
|Enable path substitution|Yes, if required by your setup|
|Copy extrafanart to extrathumbs field|No|

NFO files store metadata alongside your media. Path substitution can help adapt paths between applications, but it should only be enabled when the configured mappings are appropriate for your environment.

> [!IMPORTANT] Shared Metadata and Permissions
> If Jellyfin, Radarr, Sonarr, or another application writes metadata into media directories, make sure the applications have compatible permissions. Keep the source of truth for movie and TV metadata clear to avoid unexpected overwrites.

### 3.4 Playback and Transcoding

Navigate to:

`Dashboard → Playback → Transcoding`

Hardware acceleration can reduce CPU load during supported transcoding operations. The available options depend on the hardware, drivers, operating system, and how the Jellyfin container is configured.

#### Hardware Acceleration

For a supported NVIDIA GPU, select:

`NVIDIA NVENC`

The exact label and available options depend on the installed Jellyfin version.

Before enabling hardware acceleration, verify that:
- The server has a compatible NVIDIA GPU.
- The host has a working NVIDIA driver.
- The NVIDIA Container Toolkit is configured if Jellyfin runs in Docker and requires GPU access.
- The Jellyfin container has the required GPU devices and permissions.
- The codecs and formats you intend to transcode are supported by the GPU.

If your server uses AMD or Intel graphics, choose the compatible acceleration method supported by your hardware and Jellyfin installation instead.

> [!WARNING] Hardware Acceleration Is Not Just a Dashboard Setting
> Selecting NVENC does not automatically grant the Docker container access to the GPU. The Compose service and host must be configured correctly. If the GPU is not detected, check the host drivers, container configuration, and Jellyfin logs before changing additional playback settings.

#### Other Transcoding Settings

The remaining settings depend on your hardware and playback requirements. Start with the defaults, verify that playback works, and then adjust individual settings if necessary.

Useful checks include:
- Confirm that direct playback works for compatible media.
- Test transcoding with a file that requires conversion.
- Inspect the playback information to determine whether the stream is direct playing, remuxing, or transcoding.
- Check CPU and GPU usage during playback.
- Review Jellyfin logs if transcoding fails.

Direct Play is generally preferable when the client supports the media format, because it avoids unnecessary transcoding.

### 3.5 Plugins

Jellyfin plugins add features such as additional metadata providers, subtitle integration, and playback enhancements.

Browse the community-maintained plugin and resource collection:
- [Awesome Jellyfin](https://github.com/awesome-jellyfin/awesome-jellyfin)

You can generally manage plugins from:

`Dashboard → Plugins`

The exact installation steps depend on the plugin and your Jellyfin version. Some plugins require a repository URL, and some require additional credentials or external services.

#### Recommended Plugins to Consider

|Plugin|Purpose|
|---|---|
|Fanart|Additional artwork and images, where supported|
|File Transformation|Modify supported configuration or file-related behavior|
|Intro Skipper|Detect or skip intros, depending on plugin compatibility|
|OMDb|Movie metadata from OMDb, where supported|
|OpenSubtitles|Subtitle discovery and downloads, where supported|
|TheTVDB|TV metadata, where supported|
|TMDb|Movie and TV metadata, where supported|

> [!IMPORTANT] Check Compatibility Before Installing
> Plugin names, repositories, and compatibility can change between Jellyfin releases. Confirm that a plugin supports your installed Jellyfin version before installing it. Avoid installing multiple providers that perform the same function unless you have a reason to use both.

**After installing a plugin:**
1. Follow the plugin's setup instructions.
2. Configure any required API keys or credentials.
3. Restart Jellyfin if the plugin documentation requires it.
4. Check the logs for errors.
5. Test the feature with a small part of your library before relying on it.

---
## 4. Troubleshooting

### 4.1 Jellyfin Cannot Find Media

**Symptoms:** Libraries appear empty, or Jellyfin reports that no media was found.

Check that the expected folders exist inside the container:

```bash
sudo docker exec jellyfin ls -la /media
sudo docker exec jellyfin ls -la /media/movies
sudo docker exec jellyfin ls -la /media/tv
sudo docker exec jellyfin ls -la /media/anime
```

If the paths do not exist, inspect the volume mappings in your Compose file.

```bash
cd /srv/docker/stacks/media-stack
sudo docker compose config
```

If the paths exist but the libraries are empty, check file permissions and trigger a library scan from the Jellyfin interface.

### 4.2 Permission Denied Errors

Inspect ownership and permissions on the host:

```bash
ls -ld /srv/docker/configs/jellyfin
ls -ld /mnt/media-hdd/media/movies
ls -ld /mnt/media-hdd/media/tv
ls -ld /mnt/media-hdd/media/anime
```

Confirm which user Jellyfin runs as, then grant the minimum permissions required. Avoid recursively changing the entire media directory without first identifying the cause.

### 4.3 Jellyfin Is Not Starting

Check the container status and recent logs:

```bash
sudo docker compose -f /srv/docker/stacks/media-stack/docker-compose.yml ps jellyfin

sudo docker compose -f /srv/docker/stacks/media-stack/docker-compose.yml logs --tail=200 jellyfin
```

Look for errors related to configuration files, permissions, port conflicts, invalid volume paths, or missing devices.

### 4.4 Metadata or Artwork Is Missing

Check the following:
- The server can reach the metadata provider.
- The provider is installed and enabled.
- Required API keys or credentials are configured.
- The media is named and organized consistently.
- The library scan has completed.
- The logs do not show provider errors.

Some providers may be unavailable or rate-limited in certain regions. If a provider fails, check its current service status and documentation before changing the rest of the library configuration.

### 4.5 Hardware Transcoding Is Not Working

Check that the host detects the GPU and that the container has access to it.

For an NVIDIA GPU, run this on the host if the NVIDIA driver is installed:

```bash
nvidia-smi
```

Then inspect the Jellyfin logs:

```bash
sudo docker logs --tail=200 jellyfin
```

If the GPU is visible on the host but not inside the container, verify the NVIDIA Container Toolkit and the device configuration in Docker Compose. Also confirm that the media file actually requires transcoding and that its codecs are supported.

---

> [!TIP] Setup Complete
> Jellyfin is ready to serve your media library. Once the initial scan and metadata retrieval finish, you can browse your movies, TV shows, and anime from compatible Jellyfin clients.

---

> [!TIP] ➜ Up Next
> You’ve completed  [08 Jellyfin](#08-jellyfin)
> **Previous:** [07 Bazarr](07%20Bazarr.md)  ------ **Next:** [09 Jellyseerr](09%20Jellyseerr.md)
