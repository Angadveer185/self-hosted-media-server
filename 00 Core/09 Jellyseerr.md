# Overview

Jellyseerr is a media request management application that lets users browse movies and TV shows, submit requests, and track their status. It integrates with Jellyfin for authentication and library synchronization, and with Radarr and Sonarr to manage requested content.

In this media server setup, Jellyseerr acts as the request portal, Radarr and Sonarr handle acquisition, and Jellyfin provides the viewing experience.

> [!NOTE] Overview
> - **Web UI:** `http://localhost:5056` (assuming this is your configured host port)
> - **Docker container:** `jellyseerr`
> - **Configuration directory:** `/srv/docker/configs/jellyseerr`
> - **Jellyfin:** Port `8096` inside the container
> - **Radarr:** Port `7878` inside the container
> - **Sonarr:** Port `8989` inside the container
> - **Metadata provider:** TMDB

## Table of Contents
- [1. Installation Checks](#1-installation-checks)
- [2. First-Time Setup](#2-first-time-setup)
    - [2.1 Connect Jellyfin](#21-connect-jellyfin)
    - [2.2 Import Libraries](#22-import-libraries)
    - [2.3 Add Radarr](#23-add-radarr)
    - [2.4 Add Sonarr](#24-add-sonarr)
- [3. TMDB Connectivity and Gluetun](#3-tmdb-connectivity-and-gluetun)
- [4. Other Settings](#4-other-settings)
    - [4.1 Users and Permissions](#41-users-and-permissions)
- [5. Troubleshooting](#5-troubleshooting)

---
## 1. Installation Checks

Jellyseerr is already installed. Before opening the Web UI, verify its configuration directory, permissions, Docker Compose service, and container status.

### 1.1 Create the Configuration Directory

Run the following commands on the Ubuntu host:

```bash
# Create the Jellyseerr configuration directory
sudo mkdir -p /srv/docker/configs/jellyseerr
```

Set ownership to the UID and GID used by the Jellyseerr container, provided these match your image and Compose configuration:

```bash
# Set ownership
sudo chown -R 1000:1000 /srv/docker/configs/jellyseerr

# Set basic directory permissions
sudo chmod -R 775 /srv/docker/configs/jellyseerr
```

> [!WARNING] Verify the Container User
> `PUID=1000` and `PGID=1000` are common settings for LinuxServer.io images, but not every Jellyseerr image uses these variables. Check the image configured in your Compose file before assuming they apply. The official Jellyseerr image may use different user configuration.
> Avoid recursively changing permissions if the directory already works correctly.

### 1.2 Verify the Docker Compose Configuration

Navigate to the media stack directory:

```bash
cd /srv/docker/stacks/media-stack
```

Validate the Compose file and check the container status:

```bash
# Validate the Compose configuration
sudo docker compose config --quiet

# Check Jellyseerr status
sudo docker compose ps jellyseerr
```

Confirm that the service maps the configuration directory to the path expected by the image. For example, some Jellyseerr images use `/app/config` as their container configuration path.

Use the actual path required by your configured image. Do not change the container path blindly.

### 1.3 Start Jellyseerr

Run:

```bash
cd /srv/docker/stacks/media-stack

# Pull the configured image
sudo docker compose pull jellyseerr

# Recreate and start the service
sudo docker compose up -d jellyseerr

# Check the status
sudo docker compose ps jellyseerr

# Inspect logs if needed
sudo docker compose logs --tail=100 jellyseerr
```

### 1.4 Open the Web UI

On the server, open:

`http://localhost:5056`

This assumes your Compose configuration maps host port `5056` to the Jellyseerr container's configured Web UI port. If you changed the mapping, use the actual host port.

If you are accessing Jellyseerr from another device on the same LAN, use:

`http://SERVER_LAN_IP:5056`

Replace `SERVER_LAN_IP` with your server's actual LAN IP address.

---
## 2. First-Time Setup

When you first open Jellyseerr, the Welcome Wizard guides you through connecting Jellyfin and configuring your request services.

### 2.1 Connect Jellyfin

Select **Jellyfin** as your media server.

Configure the connection using the following settings:

|Setting|Recommended Value|
|---|---|
|Jellyfin URL / Hostname|`jellyfin` if reachable by that service name on a shared Docker network|
|Port|`8096`|
|Email Address|An email address you control|
|Username|Your Jellyfin administrator username|
|Password|Your Jellyfin administrator password|


**Choosing the Jellyfin address**
- **Shared Docker network:** Use `jellyfin` if that is the Compose service name and both containers share a network that supports Docker DNS.
- **Host network address:** Use the server's LAN IP and the published host port if the containers cannot communicate directly over a shared Docker network.
- **Gluetun network namespace:** If Jellyseerr shares Gluetun's network namespace, the correct address and port depend on the network topology and published ports.

Avoid hardcoding a container IP address when a service name works. Container IPs can change when containers are recreated.

> [!TIP] Account Security
> Use an email address you control. A secondary email can help separate service notifications from your primary inbox, but it is not a substitute for securing the account. Use a strong password and keep administrator credentials private.

Test the connection and continue once Jellyseerr successfully connects to Jellyfin.

### 2.2 Import Libraries

After connecting Jellyfin:
1. Click **Sync Libraries**.
2. Wait for Jellyseerr to retrieve the available libraries.
3. Select the libraries you want Jellyseerr to manage.
4. Continue through the wizard.

If your libraries do not appear or their artwork is missing, complete the troubleshooting steps in [5. Troubleshooting](#5-troubleshooting) before changing other settings.

### 2.3 Add Radarr

Radarr manages movie requests.

Add a Radarr server and configure the following settings. Labels may differ slightly between Jellyseerr versions.

|Setting|Recommended Value|
|---|---|
|Default Server|Yes, if this is your primary Radarr instance|
|4K Server|Personal preference; only if you run a separate 4K instance|
|Server Name|`Radarr`|
|Hostname or IP Address|`radarr` on a shared Docker network|
|Port|`7878`|
|API Key|From Radarr → Settings → General|
|Quality Profile|Your preferred Radarr profile|
|Root Folder|`/movies`|
|Minimum Availability|Released|
|Tags|`torrent`, if you use this tag in Radarr|
|Enable Scan|Yes|
|Enable Automatic Search|Yes|
|Tag Requests|Personal preference|

**Connection details**
- If Jellyseerr and Radarr share a Docker network, use the Radarr service name, normally `radarr`, and its internal port `7878`.
- If they do not share a network, use a reachable host address and the appropriate published port.
- Use the API key from Radarr's settings. Treat it like a password and do not share it publicly.

Click **Test** to verify the connection before saving.

**Root folder**

The value `/movies` assumes that Radarr's movie library is mounted at `/movies` inside its container. In this setup, that should correspond to:

`/mnt/media-hdd/media/movies`

The host path and container path are not interchangeable. Jellyseerr should use the path Radarr understands, not the host's filesystem path.

**Tags**

The `torrent` tag is optional. Use it only if you have configured Radarr to use that tag for a particular workflow. Jellyseerr tags and Radarr download-client categories are separate concepts, so do not assume that adding a request tag automatically configures qBittorrent.

### 2.4 Add Sonarr

Sonarr manages TV series and episode requests.

Add a Sonarr server and configure the following settings:

|Setting|Recommended Value|
|---|---|
|Default Server|Yes, if this is your primary Sonarr instance|
|4K Server|Personal preference; only if you run a separate 4K instance|
|Server Name|`Sonarr`|
|Hostname or IP Address|`sonarr` on a shared Docker network|
|Port|`8989`|
|API Key|From Sonarr → Settings → General|
|Series Type|Standard|
|Quality Profile|Your preferred Sonarr profile|
|Root Folder|`/tv`|
|Minimum Availability|Released|
|Tags|`torrent`, if you use this tag in Sonarr|
|Anime Series Type|Standard, if this matches your library|
|Anime Quality Profile|Your preferred anime profile|
|Anime Tags|`torrent`, if desired|
|Season Folders|Yes|
|Enable Scan|Yes|
|Enable Automatic Search|Yes|
|Tag Requests|Personal preference|

Click **Test** to verify the connection before saving.

**Root folder**

The value `/tv` assumes that Sonarr's TV library is mounted at `/tv` inside its container. In this setup, that should correspond to:

`/mnt/media-hdd/media/tv`

Use the actual root folder configured in Sonarr if it differs.

**Anime settings**

The example uses **Standard** for anime. This can work if your anime is managed using Sonarr's standard series and episode structure. Some anime releases and naming conventions require additional configuration, so verify that the chosen series type and quality profile match your Sonarr setup.

**Automatic search**

Enabling automatic search allows requests to trigger searches through Sonarr or Radarr. This is convenient, but it may begin searches immediately after requests are approved. Configure quality profiles, tags, and request permissions before allowing other users to submit requests.

---
## 3. TMDB Connectivity and Gluetun

Jellyseerr uses The Movie Database (TMDB) to retrieve much of the movie and TV show information displayed in its interface. If TMDB is unreachable, you may see missing posters, incomplete metadata, or failed searches.

### 3.1 My Connectivity Issue

In my setup, Jellyseerr was not displaying images or metadata correctly, and media synchronization was not behaving as expected. I traced the issue to TMDB connectivity from my server's network because TMDB was banned in my country.

Routing Jellyseerr through Gluetun was my workaround. This is a configuration-specific solution, not a universal requirement.

> [!WARNING] Diagnose Before Changing the Network
> Missing images and metadata can also result from DNS problems, blocked outbound requests, incorrect system time, provider errors, or an incomplete library sync. Verify that TMDB is actually unreachable before changing Jellyseerr's network configuration.

### 3.2 Route Jellyseerr Through Gluetun

If your network environment requires VPN routing, Jellyseerr can be configured to use Gluetun. The exact Compose configuration depends on how your other containers are connected.

Common approaches include:
- **Shared Gluetun network namespace:** Jellyseerr uses Gluetun's network namespace. Required inbound ports must be published on the Gluetun service, and networking behavior differs from a normal standalone container.
- **Separate network arrangement:** Jellyseerr remains on its own network while outbound traffic is routed through a suitable VPN gateway or other supported network configuration.

Do not simply set the Jellyseerr hostname to `gluetun`. Gluetun is not automatically a substitute for the Jellyfin, Radarr, or Sonarr service addresses. It is a VPN gateway, and the correct connection details depend on the Compose networking configuration.

Before making changes:
1. Back up your existing Compose file.
2. Verify that Jellyseerr can reach Jellyfin, Radarr, and Sonarr.
3. Confirm that Jellyseerr's outbound traffic uses the intended route.
4. Check the container logs for DNS and connection errors.
5. Confirm that the Jellyseerr Web UI remains accessible after the change.

> [!IMPORTANT] VPN Routing Can Affect Internal Connections
> When Jellyseerr shares Gluetun's network namespace, it does not behave like an independent container with its own network interface. You may need to adjust service discovery, firewall settings, and port publishing. Test all integrations after changing the network.

---
## 4. Other Settings

Jellyseerr includes settings for user accounts, request limits, permissions, notifications, and request approval behavior. The following values reflect one possible personal setup rather than universal recommendations.

### 4.1 Users and Permissions

Navigate to the user settings section of Jellyseerr.

#### General Settings

|Setting|Example Value|
|---|---|
|Enable New Jellyfin Sign In|No|
|Global Movie Request Limit|Unlimited|
|Global Series Request Limit|Unlimited|

The exact option names may vary between versions. Review each setting in your installed version before changing it.

**Enable New Jellyfin Sign In**

This controls whether users can sign in through the Jellyfin integration, depending on the available authentication options. Keep it disabled if you want to manage sign-in another way. If you want Jellyfin users to access Jellyseerr, verify the authentication behavior before deciding.

**Global request limits**

Unlimited request limits allow users to submit as many requests as permissions permit. If the server is shared with many people, consider limits to prevent excessive requests or storage growth.

#### Default Permissions

The following permissions are examples of those you may choose to grant, depending on your trust model and Jellyseerr version.

|Permission|Purpose|
|---|---|
|Advanced Requests|Allows access to additional request options|
|View Requests|Allows users to view request status|
|View Recently Added|Allows users to view recently added media|
|Request|Allows users to submit requests|
|Auto-Approve|Allows requests to be approved automatically|
|Auto-Request|Allows eligible requests to be automatically created or submitted, depending on the integration|

> [!WARNING] Be Careful With Auto-Approval
> `Auto-Approve` and `Auto-Request` can reduce manual work, but they also give users more influence over what gets added to your library. For a shared server, grant these permissions only to trusted users and configure quality profiles and search behavior first.

Start with basic request and viewing permissions for ordinary users. Add automatic approval or advanced permissions only when you are comfortable with the consequences.

---
## 5. Troubleshooting

### 5.1 Jellyseerr Cannot Connect to Jellyfin

Check the service status and logs:

```bash
cd /srv/docker/stacks/media-stack

sudo docker compose ps jellyseerr jellyfin

sudo docker compose logs --tail=100 jellyseerr
sudo docker compose logs --tail=100 jellyfin
```

Verify that:
- Both services are running.
- The hostname is resolvable from Jellyseerr.
- The correct internal or published port is being used.
- Both containers can communicate over their configured network.
- The Jellyfin credentials are correct.

Remember that Jellyfin's usual internal port is `8096`. Use a different port only if you deliberately changed it.

### 5.2 Radarr or Sonarr Connection Fails

Verify the connection details:
- Radarr host: `radarr`, if on a shared Docker network.
- Radarr port: `7878`.
- Sonarr host: `sonarr`, if on a shared Docker network.
- Sonarr port: `8989`.
- API keys: copied from each application's settings.
- Root folders: valid paths configured inside the respective containers.

If Jellyseerr is connected through Gluetun or isolated on a separate Docker network, confirm that it can still reach the Radarr and Sonarr services.

### 5.3 Posters or Metadata Are Missing

Check the following:
1. Confirm that Jellyseerr can reach TMDB.
2. Check DNS resolution and outbound connectivity from the container.
3. Inspect Jellyseerr logs for provider errors.
4. Verify that the library sync has completed.
5. Confirm that the relevant libraries are selected in Jellyseerr.
6. Check whether the issue affects all content or only specific titles.

If the problem is network-related, resolve that first. Changing quality profiles or root folders will not fix an outbound metadata connectivity issue.

### 5.4 Libraries Are Not Synchronizing

Verify that Jellyseerr can connect to Jellyfin and that the correct libraries were selected during setup.

Also check that Jellyfin itself can see the media files. Jellyseerr's library synchronization depends on the information Jellyfin exposes through its API, so a missing or empty Jellyfin library can cause synchronization problems upstream.

### 5.5 Jellyseerr Is Not Starting

Check the container status and logs:

```bash
cd /srv/docker/stacks/media-stack

sudo docker compose ps jellyseerr
sudo docker compose logs --tail=200 jellyseerr
```

Look for permission errors, invalid volume mappings, port conflicts, or configuration problems.

---

> [!TIP] Setup Complete
> Jellyseerr is ready to act as the request portal for your media server. Users can discover content and submit requests, while Radarr and Sonarr handle the configured acquisition workflows and Jellyfin provides the media playback experience.

---

> [!TIP] ➜ Up Next
> You’ve completed  [09 Jellyseerr](#09-jellyseerr)
> **Previous:** [08 Jellyfin](08%20Jellyfin.md)  ------ **Next:** [00 Hosting](../04%20Optional/00%20Hosting.md)
