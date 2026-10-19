# Overview

Bazarr is a subtitle management application that integrates with Sonarr and Radarr to find, download, and organize subtitles for TV episodes and movies.

Instead of searching for subtitles manually, Bazarr can monitor your media libraries, identify missing subtitles, search configured providers, and synchronize subtitle files with the corresponding media.

This guide covers verifying the existing installation, and configuration.

> [!NOTE] Prerequisites
> Before proceeding, ensure that:
> - Docker Engine and Docker Compose are installed and running.
> - Bazarr has already been deployed as part of the media stack.
> - Sonarr and Radarr are running and accessible.
> - The media directories are mounted into Bazarr with appropriate permissions.
> - You have accounts for any subtitle providers that require authentication.
> Refer to [01 Docker](01%20Docker.md), [06 Sonarr](06%20Sonarr.md), and [05 Radarr](05%20Radarr.md) for the corresponding setup instructions.

## Table of Contents
- [1. Installation & Verification](#1-installation-verification)
    - [1.1. Prepare the Directories](#11-prepare-the-directories)
    - [1.2. Start or Restart Bazarr](#12-start-or-restart-bazarr)
    - [1.3. Access the Web Interface](#13-access-the-web-interface)
- [2. Configure Languages](#2-configure-languages)
    - [2.1. Set the Subtitle Language Filter](#21-set-the-subtitle-language-filter)
    - [2.2. Create a Language Profile](#22-create-a-language-profile)
    - [2.3. Set Default Profiles](#23-set-default-profiles)
- [3. Configure Subtitle Providers](#3-configure-subtitle-providers)
- [4. Configure Subtitle Processing](#4-configure-subtitle-processing)
    - [4.1. Subtitle File Options](#41-subtitle-file-options)
    - [4.2. Embedded Subtitle Handling](#42-embedded-subtitle-handling)
    - [4.3. Subtitle Content Modifications](#43-subtitle-content-modifications)
    - [4.4. Audio Synchronization](#44-audio-synchronization)
    - [4.5. Advanced FFsubsync Options](#45-advanced-ffsubsync-options)
- [5. Integrate Sonarr](#5-integrate-sonarr)
    - [5.1. Configure the Connection](#51-configure-the-connection)
    - [5.2. Configure Options](#52-configure-options)
    - [5.3. Configure Path Mappings](#53-configure-path-mappings)
- [6. Integrate Radarr](#6-integrate-radarr)
    - [6.1. Configure the Connection](#61-configure-the-connection)
    - [6.2. Configure Options](#62-configure-options)
    - [6.3. Configure Path Mappings](#63-configure-path-mappings)
- [7. Future Integration: Jellyfin](#7-future-integration-jellyfin)
- [8. Troubleshooting](#8-troubleshooting)

---
## 1. Installation & Verification

Bazarr has already been installed as part of the media stack. We only need to verify its configuration directory, ensure the container is running, and open the Web UI.

### 1.1. Prepare the Directories

Create the directories required by Bazarr if they do not already exist.

```bash
sudo mkdir -p /srv/docker/configs/bazarr
sudo mkdir -p /mnt/media-hdd/downloads
sudo mkdir -p /mnt/media-hdd/media
```

Set ownership to UID `1000` and GID `1000`, assuming these are the IDs configured for the relevant containers.

```bash
sudo chown -R 1000:1000 /srv/docker/configs/bazarr
sudo chown -R 1000:1000 /mnt/media-hdd/downloads
sudo chown -R 1000:1000 /mnt/media-hdd/media
```

Set basic directory permissions:

```bash
sudo chmod -R 775 /srv/docker/configs/bazarr
sudo chmod -R 775 /mnt/media-hdd/downloads
sudo chmod -R 775 /mnt/media-hdd/media
```

These commands create the directories, assign ownership, and configure basic filesystem permissions.

> [!WARNING] Review permissions before changing them
> UID `1000` and GID `1000` are common defaults, not universal values. Confirm that they match the configuration of your containers.
> The recursive commands above affect existing files and directories as well. Avoid changing the ownership of your entire media library unless you have confirmed that doing so is appropriate.

### 1.2. Start or Restart Bazarr

Navigate to the directory containing the media stack's Compose file:

```bash
cd /srv/docker/stacks/media-stack
```

If Bazarr is already running and its Compose configuration has not changed, restart it:

```bash
docker compose restart bazarr
```

If you have modified the Compose configuration and need Docker to apply those changes, recreate the container:

```bash
docker compose up -d --force-recreate bazarr
```

To pull the configured image before deploying:

```bash
docker compose pull bazarr
docker compose up -d bazarr
```

Check the container status:

```bash
docker ps \
  --filter "name=bazarr" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

If Bazarr is not running, inspect its logs:

```bash
docker logs --tail=100 bazarr
```

> [!NOTE] Compose commands
> These commands use Docker Compose V2, invoked as `docker compose`. Use the Compose file that actually defines your Bazarr service.
> Pulling the latest image is optional. For a stable media server, review release notes before updating.

### 1.3. Access the Web Interface

Open Bazarr in your browser:

```bash
http://localhost:6767
```

If you are accessing the server from another computer, replace `localhost` with the server's IP address:

```bash
http://SERVER_IP:6767
```

This assumes host port `6767` is published in the Compose configuration.

> [!TIP] Bazarr is ready
> Once the Web UI loads, configure the language preferences before connecting your media applications.

---
## 2. Configure Languages

Language settings determine which subtitles Bazarr searches for and which language profiles it applies to movies and TV series.

### 2.1. Set the Subtitle Language Filter

Navigate to:

**Settings → Languages**

Under **Subtitles Language**, configure:

|Setting|Value|
|---|---|
|Languages Filter|English|

This limits the available language choices and simplifies the configuration if English is your primary subtitle language.

### 2.2. Create a Language Profile

Under **Language Profiles**, create a profile with the following settings.

|Setting|Value|
|---|---|
|Name|`English-Standard`|
|Language|English|
|Subtitles Type|Normal or Hearing Impaired|
|Search Only When|Always|
|Cutoff|`en`|

The exact fields and labels can vary between Bazarr versions. Configure the corresponding options available in your installation.

**Understanding the settings:**
- **Name:** Identifies the language profile.
- **Language:** Specifies the desired subtitle language.
- **Subtitles Type:** Determines whether the profile accepts standard subtitles, hearing-impaired subtitles, or both, depending on the available options.
- **Search Only When:** Controls when Bazarr searches for subtitles according to the selected policy.
- **Cutoff:** Determines the point at which the configured language requirement is considered satisfied, based on the options available in your version.

> [!NOTE] Normal versus hearing-impaired subtitles
> Standard subtitles generally contain spoken dialogue and relevant sounds when included by the subtitle author. Hearing-impaired subtitles also include additional sound cues, such as `[door slams]` or `[phone ringing]`.
> Choose the type that best matches your preferences. If you want to keep both versions, configure the profile accordingly rather than assuming a single subtitle type will cover both.

### 2.3. Set Default Profiles

Configure the default language profile for newly added media.

|Media Type|Enabled|Profile|
|---|---|---|
|Series|Yes|`English-Standard`|
|Movies|Yes|`English-Standard`|

This ensures that newly added TV series and movies inherit the intended language profile.

If your Bazarr version offers separate controls for series and movies, configure each independently.

> [!TIP] Existing media
> Default profiles for newly added content may not automatically update every existing movie or series. Review the existing library and apply the intended profile where necessary.

---
## 3. Configure Subtitle Providers

Subtitle providers are the sources Bazarr searches when looking for matching subtitles.

Navigate to:

**Settings → Providers**

Enable and configure the providers you intend to use.

Examples include:
- [OpenSubtitles.com](https://www.opensubtitles.com/)
- YIFY Subtitles
- Embedded Subtitles
- TVSubtitles
- Wizdom
- Anime Tosho

Provider availability and compatibility can change over time. Some providers may be unavailable, require particular authentication methods, or work only with certain content types.

### OpenSubtitles.com

OpenSubtitles.com may require an account and valid authentication details.
1. Create an account if required.
2. Open Bazarr's provider settings.
3. Enter the credentials or API details requested by the provider integration.
4. Save and test the configuration, if a test option is available.

### Other Providers

For each provider:
1. Enable the provider if it is supported by your Bazarr version.
2. Enter any required credentials or settings.
3. Save the configuration.
4. Test subtitle searches against a known movie or episode.

> [!TIP] Enable providers selectively
> You do not need to enable every available provider. Start with a few reliable sources, verify their results, and add others when they provide useful additional coverage.
> Provider availability, rate limits, and account requirements may change over time.

---
## 4. Configure Subtitle Processing

Subtitle processing settings determine how Bazarr handles downloaded subtitle files, modifies their content, and attempts to synchronize them with the media.

The following configuration reflects the preferences used in this setup.

### 4.1. Subtitle File Options

Navigate to the subtitle processing settings and configure:

|Setting|Value|
|---|---|
|Encode Subtitles to UTF-8|Enabled|

UTF-8 encoding helps maintain compatibility with a wide range of characters and languages.

It is especially useful when subtitle files contain accented characters, non-Latin scripts, or other text that may not be represented correctly by older encodings.

### 4.2. Embedded Subtitle Handling

Configure:

|Setting|Value|
|---|---|
|Treat Embedded Subtitles as Downloaded|Disabled|

This controls how Bazarr treats subtitles embedded within the media file when evaluating whether subtitles have already been obtained.

With this option disabled, embedded subtitles are not automatically treated as downloaded subtitles by this setting. The exact search behavior depends on the rest of your subtitle configuration.

> [!NOTE] Embedded versus external subtitles
> Embedded subtitles are stored inside the media container, such as an MKV file. External subtitles are separate files, such as `.srt` or `.ass`.
> Your preference here should reflect whether you want Bazarr to consider embedded tracks sufficient or continue looking for external subtitle files.

### 4.3. Subtitle Content Modifications

Under **Sub-Zero Subtitle Content Modifications**, configure the following options where available:

|Setting|Value|
|---|---|
|Hearing Impaired|Disabled|
|Remove Tags|Enabled|
|OCR Fixes|Enabled|
|Common Fixes|Enabled|

**What these settings do:**
- **Hearing Impaired:** Controls the handling of hearing-impaired subtitle content.
- **Remove Tags:** Removes supported formatting or markup tags from subtitle text.
- **OCR Fixes:** Applies supported corrections to errors commonly introduced by optical character recognition.
- **Common Fixes:** Applies supported corrections to common subtitle formatting or text problems.

These options can improve readability, but the result depends on the source subtitle and the type of corrections applied.

> [!TIP] Verify the results
> Subtitle processing can occasionally alter text that was already correct. Test the settings against a few files before applying the same configuration broadly.

### 4.4. Audio Synchronization

Bazarr can attempt to synchronize subtitles with the media's audio when a subtitle is out of sync.

Configure the following options if they are available in your version.

|Setting|Value|
|---|---|
|Automatic Subtitles Audio Synchronization|Enabled|
|Series Score Threshold for Audio Sync|`90`|
|Movie Score Threshold for Audio Sync|`80`|

The score thresholds control which synchronization candidates meet the configured confidence requirements.

The values above are the preferences used in this setup, not universal recommendations. Higher thresholds may be more selective, while lower thresholds may allow more candidates to be processed.

> [!WARNING] Synchronization can be imperfect
> Audio synchronization cannot reliably fix every subtitle mismatch. Different edits, missing scenes, alternate releases, and substantial timing differences can affect the result.
> If a subtitle remains out of sync after processing, verify that it corresponds to the exact media release and runtime.

### 4.5. Advanced FFsubsync Options

The following options reflect the current experimental configuration used in this setup.

|Setting|Value|
|---|---|
|Synchronization Reference|Use Audio Track as Reference|
|Do Not Fix Framerate Mismatch|Disabled|
|Golden Section Search|Enabled|

**Synchronization Reference: Use Audio Track as Reference**

This selects the audio track as the reference for synchronization, where supported by the integration.

**Do Not Fix Framerate Mismatch: Disabled**

This allows the synchronization workflow to attempt its supported frame-rate mismatch corrections rather than explicitly disabling them.

**Golden Section Search: Enabled**

This enables the relevant search strategy in supported synchronization workflows.

> [!WARNING] Experimental settings
> These synchronization settings are still being tested in this setup. Results may vary depending on the media, subtitle source, frame rate, and synchronization method.
> If synchronization introduces new timing problems, disable the relevant option or compare the result against the original subtitle before keeping the modified file.

---
## 5. Integrate Sonarr

Sonarr manages TV series and episodes. Connecting it to Bazarr allows Bazarr to discover the series library and manage subtitles for its episodes.

### 5.1. Configure the Connection

In Bazarr, navigate to the Sonarr integration settings.

Configure the connection as follows:

|Setting|Value|
|---|---|
|Address / Host|`sonarr`|
|Port|`8989`|
|Base URL|`/`|
|HTTP Timeout|`60` seconds|
|API Key|The API key from Sonarr|

The hostname `sonarr` assumes that Bazarr and Sonarr share a Docker network on which the Sonarr service name resolves. You may even try putting in the Server IP. If routing through Gluetun, use `gluetun`

#### Retrieve the Sonarr API Key
1. Open Sonarr at `http://localhost:8989`.
2. Navigate to **Settings → General**.
3. Locate the API key.
4. Copy it into Bazarr's Sonarr integration settings.

Keep the API key private because it grants access to Sonarr's API.

#### Test the Connection

Use Bazarr's **Test** button to verify that it can communicate with Sonarr.

If the test fails, check:
- Sonarr is running.
- The API key is correct.
- Bazarr can resolve the `sonarr` hostname.
- Both containers share an appropriate Docker network.
- The port and base URL match Sonarr's configuration.

> [!IMPORTANT] Use the container's internal port
> When Bazarr and Sonarr share a Docker network, use the service name and internal port, typically `sonarr:8989`, rather than relying on the host's published port.

### 5.2. Configure Options

Configure the following options according to your preferences.

|Setting|Value|
|---|---|
|Minimum Score for Episodes|`85`|
|Exclude Season Zero (Extras)|Your choice|

**Minimum Score for Episodes:** `85`

This sets the minimum score Bazarr accepts when evaluating subtitle candidates, according to the scoring system supported by the relevant provider and Bazarr version.

Higher thresholds may reduce mismatched subtitles but can also exclude valid subtitles when matching information is incomplete.

**Exclude Season Zero (Extras)**

Enable this if you do not want Bazarr to search for or manage subtitles for special episodes and extras categorized under Season 0.

Disable it if you want Bazarr to handle those items as well.

### 5.3. Configure Path Mappings

Path mappings tell Bazarr how to translate file paths reported by Sonarr into paths that Bazarr can access.

For example, Sonarr may report a series path as:

```bash
/tv/Breaking Bad/Season 01/Breaking Bad - S01E01 - Pilot.mkv
```

Bazarr must be able to resolve that file using its own container filesystem.

If Sonarr and Bazarr use different paths for the same host directory, configure the mapping accordingly.

Example:

|Application|Path|
|---|---|
|Sonarr|`/tv/`|
|Bazarr|`/mnt/media/tv/`|

This mapping assumes that:
- Sonarr sees the TV library under `/tv`.
- Bazarr sees the same underlying library under `/mnt/media/tv`.

> [!IMPORTANT] Match the actual Compose mounts
> The paths above are examples based on the supplied configuration. If your Bazarr Compose service mounts the TV directory at `/tv`, then the paths may already match and no mapping may be necessary.
> Do not add a mapping solely because another guide uses different paths. Confirm the actual volume mounts for both containers.

---
## 6. Integrate Radarr

Radarr manages the movie library. Connecting it to Bazarr allows Bazarr to discover movies and manage their subtitles.

### 6.1. Configure the Connection

In Bazarr, navigate to the Radarr integration settings.

Configure the following connection details:

|Setting|Value|
|---|---|
|Address / Host|`radarr`|
|Port|`7878`|
|Base URL|`/`|
|HTTP Timeout|`60` seconds|
|API Key|The API key from Radarr|

The hostname `radarr` assumes that Bazarr and Radarr share a Docker network on which the Radarr service name resolves. You may even try putting in the Server IP. If routing through Gluetun, use `gluetun`

#### Retrieve the Radarr API Key
1. Open Radarr at `http://localhost:7878`.
2. Navigate to **Settings → General**.
3. Locate the API key.
4. Copy it into Bazarr's Radarr integration settings.

Keep the API key private.

#### Test the Connection

Use Bazarr's **Test** button to verify that the integration works.

If the test fails, verify that Radarr is running, the API key is correct, and the containers can communicate over the Docker network.

### 6.2. Configure Options

Configure the following option according to your preferences.

|Setting|Value|
|---|---|
|Minimum Score for Movies|`80`|

This sets the minimum score Bazarr accepts when evaluating movie subtitle candidates, according to the matching system supported by the relevant provider and Bazarr version.

A higher score can make matching more selective, but it may also exclude valid subtitles when the available metadata is incomplete.

### 6.3. Configure Path Mappings

Path mappings tell Bazarr how to translate file paths reported by Radarr into paths that Bazarr can access.

Example:

|Application|Path|
|---|---|
|Radarr|`/movies/`|
|Bazarr|`/mnt/media/movies/`|

This mapping assumes that:
- Radarr sees the movie library under `/movies`.
- Bazarr sees the same underlying directory under `/mnt/media/movies`.

For example, Radarr may report:

```bash
/movies/Dune (2021)/Dune (2021).mkv
```

Bazarr must be able to translate this to the corresponding path in its own container.

> [!IMPORTANT] Confirm the actual paths
> If Bazarr and Radarr both mount the movie library at `/movies`, the mapping may not be necessary.
> The source and destination must refer to the same underlying media files. A path mapping does not mount a directory or grant filesystem permissions by itself.

---
## 7. Future Integration: Jellyfin

Jellyfin is planned as a future integration. Once Jellyfin is installed and configured, Bazarr can be connected to it if the relevant integration is supported by your Bazarr version.

The purpose of this integration is to notify Jellyfin when subtitle changes occur, helping it refresh the relevant media information.

### 7.1. Prepare the Connection

The following values are examples to use once Jellyfin has been deployed.

|Setting|Value|
|---|---|
|Server URL|`http://jellyfin:8096`|
|API Key|A key created in Jellyfin's administration dashboard|
|Notify After Subtitle Changes|`Immediate`|
|Movie Library|Your movie library name|
|Series Library|Your series library name|

The hostname `jellyfin` assumes that Jellyfin and Bazarr share a Docker network on which the Jellyfin service name resolves.

If Jellyfin is exposed through another address, use the appropriate reachable endpoint instead.

Possible alternatives include a reachable server IP address or a correctly configured endpoint through Gluetun, depending on the Docker networking arrangement.

> [!WARNING] Network reachability
> Use the address that Bazarr can actually reach from its container. `localhost` inside Bazarr refers to Bazarr itself, not the Jellyfin container or the Docker host.

### 7.2. Retrieve the Jellyfin API Key

Once Jellyfin is installed:
1. Open the Jellyfin administration dashboard.
2. Navigate to **Dashboard → API Keys**.
3. Create an API key for Bazarr, if appropriate.
4. Copy the key into Bazarr's Jellyfin integration settings.

API keys should be treated as credentials and kept private.

### 7.3. Configure Libraries

Select the appropriate movie and series libraries from the available settings.

For example:
- **Movie Library:** The name assigned to your movie library in Jellyfin.
- **Series Library:** The name assigned to your TV series library in Jellyfin.

Use the actual library names configured in Jellyfin rather than assuming that they are named `Movies` and `Series`.

### 7.4. Configure Notifications

If the integration provides a setting for notifying Jellyfin after subtitle changes, set it to:

**Immediate**

This is intended to notify Jellyfin promptly when supported subtitle changes occur.

Verify the integration after Jellyfin has been deployed, as the available options and behavior may vary by version.

---
## 8. Troubleshooting

### Bazarr's Web UI does not load

Check the container status:

```bash
docker ps --filter "name=bazarr"
```

Inspect the logs:

```bash
docker logs --tail=100 bazarr
```

Confirm that host port `6767` is published and that the container is running.

### Bazarr cannot connect to Sonarr or Radarr

Check that the relevant application is running and that the API key is correct.

Inspect the Docker networks:

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' bazarr | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' sonarr | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' radarr | jq
```

Ensure that Bazarr can resolve the application service names and communicate over the configured ports.

### Bazarr cannot find media files

Verify that the media directories are mounted into Bazarr.

Inspect the volume mappings in your Compose configuration and confirm that the path mappings match the paths reported by Sonarr and Radarr.

Remember that path mappings translate paths. They do not create missing mounts.

### No subtitles are found

Check the following:
- The desired language is enabled.
- The correct language profile is assigned to the media.
- At least one compatible subtitle provider is enabled.
- Provider credentials are correct, where required.
- The subtitle score thresholds are not too restrictive.
- The media files are accessible to Bazarr.
- The provider supports the content being searched.

### Subtitle synchronization produces poor results

Inspect the original subtitle and the synchronized output.

If the timing becomes worse, disable the relevant synchronization option or restore the original subtitle. Check that the subtitle corresponds to the exact media release before attempting further synchronization.

### Jellyfin integration fails

Because Jellyfin is not yet configured, this section is preparatory. Once Jellyfin is installed, verify the service URL, API key, library names, and network connectivity before testing the integration.

---

> [!TIP] Bazarr configuration complete
> Bazarr is ready to manage subtitles for the TV episodes and movies tracked by Sonarr and Radarr. Once the provider configuration and path mappings are correct, you can test subtitle searches on a few files before relying on automatic subtitle management across the entire library.

---

> [!TIP] ➜ Up Next
> You’ve completed  [07 Bazarr](#07-bazarr)
> **Previous:** [06 Sonarr](06%20Sonarr.md)  ------ **Next:** [08 Jellyfin](08%20Jellyfin.md)
