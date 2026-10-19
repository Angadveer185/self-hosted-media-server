# Overview
Prowlarr is an indexer manager used to organize and manage indexers for applications such as Sonarr and Radarr. Instead of configuring indexers separately in each application, Prowlarr allows us to manage them centrally and synchronize them with our connected applications.

This guide covers verifying the existing installation, configuring FlareSolverr, adding indexers, and connecting other services. 

> [!NOTE] Prerequisites
> Before proceeding, ensure that:
> - Docker Engine and Docker Compose are installed and running.
> - The media stack has been deployed.
> - Prowlarr, Sonarr, Radarr, qBittorrent, and Gluetun are configured in Docker Compose, as applicable.
> - FlareSolverr is available if you intend to use it for compatible indexers.
> Refer to [01 Docker](01%20Docker.md) for the initial Docker configuration.
> You have already configured a VPN, either through [02 Gluetun](02%20Gluetun.md) or by running a system-wide VPN

## Table of Contents
- [1. Installation & Verification](#1-installation-verification)
    - [1.1. Prepare the Configuration Directory](#11-prepare-the-configuration-directory)
    - [1.2. Restart Prowlarr](#12-restart-prowlarr)
    - [1.3. Verify the Container](#13-verify-the-container)
    - [1.4. Access the Web Interface](#14-access-the-web-interface)
- [2. Initial Configuration](#2-initial-configuration)
    - [2.1. Verify VPN Connectivity](#21-verify-vpn-connectivity)
    - [2.2. Verify FlareSolverr](#22-verify-flaresolverr)
    - [2.3. Configure FlareSolverr as an Indexer Proxy](#23-configure-flaresolverr-as-an-indexer-proxy)
- [3. Configure Indexers](#3-configure-indexers)
    - [3.1. Add Indexers](#31-add-indexers)
    - [3.2. Test and Troubleshoot Indexers](#32-test-and-troubleshoot-indexers)
- [4. Connect Applications](#4-connect-applications)
    - [4.1. Connect Radarr](#41-connect-radarr)
    - [4.2. Connect Sonarr](#42-connect-sonarr)
    - [4.3. Test Application Connections](#43-test-application-connections)
- [5. Configure the Download Client](#5-configure-the-download-client)
    - [5.1. Add qBittorrent](#51-add-qbittorrent)
    - [5.2. Test the Download Client](#52-test-the-download-client)
- [6. Troubleshooting](#6-troubleshooting)

---
## 1. Installation & Verification

Prowlarr has already been installed as part of the media stack. Therefore, we do not need to install it again. We only need to verify its configuration directory, restart the container if necessary, and confirm that the web interface is accessible.

### 1.1. Prepare the Configuration Directory

Prowlarr requires persistent storage for its configuration, database, settings, and application state.

Create the configuration directory if it does not already exist:

```bash
sudo mkdir -p /srv/docker/configs/prowlarr
```

Set the directory ownership to the user and group IDs used by the container.

For a setup using `PUID=1000` and `PGID=1000`:

```bash
sudo chown -R 1000:1000 /srv/docker/configs/prowlarr
```

Set the directory permissions:

```bash
sudo chmod -R 775 /srv/docker/configs/prowlarr
```


> [!WARNING] Match the container's user IDs
> UID `1000` and GID `1000` are common defaults, not universal values. Confirm that these match the `PUID` and `PGID` configured for your Prowlarr container.
> Also ensure that the Compose file mounts this directory into the correct container path, typically `/config` for LinuxServer.io's Prowlarr image.

### 1.2. Restart Prowlarr

Navigate to the directory containing the media stack's Compose file:

```bash
cd /srv/docker/stacks/media-stack
```

If Prowlarr is defined in the main `docker-compose.yml`, restart only the Prowlarr service:

```bash
docker compose restart prowlarr
```

If you have changed the Compose configuration and want Docker to apply those changes, use:

```bash
docker compose up -d --force-recreate prowlarr
```

### 1.3. Verify the Container

Check whether Prowlarr is running:

```bash
docker ps \
  --filter "name=prowlarr" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

You should see the Prowlarr container with a running status.

If the container is missing, stopped, or repeatedly restarting, inspect its logs:

```bash
docker logs --tail=100 prowlarr
```

### 1.4. Access the Web Interface

Once the container is running, open Prowlarr in your browser:

```bash
http://localhost:9696
```

If you are accessing the server from another device, replace `localhost` with the server's IP address:

```bash
http://SERVER_IP:9696
```

For example:

```bash
http://192.168.1.100:9696
```

This assumes that port `9696` is published on the host. If it is not, use the appropriate network address and port for your configuration.

> [!TIP] Prowlarr is ready
> Once the web interface loads, we can move on to the initial configuration.

---
## 2. Initial Configuration

Before adding indexers, verify that the networking components Prowlarr depends on are functioning correctly.

This setup uses Gluetun as the VPN gateway and FlareSolverr as an optional helper for compatible indexers that encounter certain anti-bot challenges.

### 2.1. Verify VPN Connectivity

If your media stack routes Prowlarr through Gluetun, verify that the VPN container is running.

```bash
docker ps \
  --filter "name=gluetun" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Inspect the Gluetun logs:

```bash
docker logs --tail=100 gluetun
```

Look for successful VPN connection messages and investigate any authentication, connection, or configuration errors.

> [!IMPORTANT] Running is not the same as connected
> A running Gluetun container does not necessarily mean that its VPN tunnel is healthy. Verify the logs and, where appropriate, test outbound connectivity from the network namespace used by Prowlarr.
> If Prowlarr is not yet routed through Gluetun, it will continue using its existing network path. Simply starting the VPN container does not route other containers through it.

### 2.2. Verify FlareSolverr

FlareSolverr is a helper service that can attempt to solve certain browser-based anti-bot challenges for supported indexers.

It is optional. Not every indexer requires it, and it does not reliably bypass every anti-bot mechanism.

#### Step 1: Check the container

```bash
docker ps \
  --filter "name=flaresolverr" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

If the container is not running, inspect its logs:

```bash
docker logs --tail=100 flaresolverr
```

#### Step 2: Inspect its Docker network

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' flaresolverr | jq
```

This displays the network attachments and IP addresses assigned to the container.

If `jq` is not installed, you can use:

```bash
docker inspect flaresolverr
```

> [!NOTE] Container IP addresses
> A container's IP address may change when it is recreated. For communication between containers on the same user-defined Docker network, using the Compose service name is generally more reliable than hardcoding the container IP address.

#### Step 3: Test the FlareSolverr API

If FlareSolverr publishes port `8191` on the host, test it using:

```bash
http://localhost:8191
```

If the API is not published on the host, test it from an appropriate container or network endpoint that can reach FlareSolverr.

FlareSolverr's root endpoint is expected to return a JSON response similar to:

```bash
{
  "msg": "FlareSolverr is ready!",
  "version": "3.4.6",
  "userAgent": "..."
}
```

The exact version and user-agent string may differ.

> [!TIP] Prefer Docker service names
> If Prowlarr and FlareSolverr share a Docker network, the endpoint will typically be:
> `http://flaresolverr:8191`
> This assumes the Compose service name is `flaresolverr` and the service listens on port `8191` inside the container.
> However, if you are routing through gluetun, then use:  `http://gluetun:8191`
> Use the container's IP address only when there is a specific reason to do so.

### 2.3. Configure FlareSolverr as an Indexer Proxy

Once FlareSolverr is running and reachable, configure it in Prowlarr.

#### Step 1: Open the proxy settings

In Prowlarr, navigate to:

**Settings → Indexers → Indexer Proxies → Add Proxy → FlareSolverr**

The exact labels may vary slightly between versions.

#### Step 2: Configure the proxy

Use the following values:

| Setting         | Value                                               |
| --------------- | --------------------------------------------------- |
| Name            | `FlareSolverr`                                      |
| Tags            | `bypass`                                            |
| Host / URL      | `http://flaresolverr:8191` or `http://gluetun:8191` |
| Request Timeout | `180` seconds                                       |

Replace the hostname with the actual Compose service name if it differs.

If you are routing through gluetun, then use:  `http://gluetun:8191`

If Prowlarr and FlareSolverr are not on a shared network, the hostname will not resolve automatically. Fix the network configuration rather than assuming the container IP will work.

#### Step 3: Understand proxy tags

The `bypass` tag is a label used to select which indexers use the configured FlareSolverr proxy.

For example:
- Indexers without the `bypass` tag use their normal configured connection path.
- Indexers assigned the `bypass` tag can use the FlareSolverr proxy, provided the indexer integration supports that proxy.

You can choose a different tag name, but the tag configured on the proxy must match the tag assigned to the relevant indexers.

#### Step 4: Save and test

Save the proxy configuration.

If Prowlarr offers a test button for the proxy, use it to verify connectivity. If the test fails, confirm that:
- FlareSolverr is running.
- The hostname resolves from Prowlarr's network.
- Port `8191` is reachable.
- The proxy URL is correct.
- The service versions and indexer integration remain compatible.

> [!WARNING] FlareSolverr limitations
> FlareSolverr is not a universal Cloudflare bypass. Some indexers may still fail because of changes to their anti-bot systems, unsupported challenges, rate limits, or other restrictions.
> Use it only where necessary and in accordance with the relevant services' terms.

---
## 3. Configure Indexers

Indexers provide the sources that Prowlarr searches for content. Prowlarr can then synchronize compatible indexers with Sonarr and Radarr.

### 3.1. Add Indexers

In Prowlarr, navigate to:

**Indexers → Add Indexer**

Search for the indexer you want to configure and select it.

Examples of indexers that may be available, depending on the current Prowlarr catalog and their compatibility, include:
- 1337x
- The Pirate Bay
- YTS
- Knaben
- EZTV
- TorrentGalaxy
- LimeTorrents

Availability and compatibility can change over time. Not every indexer will be available or functional in every region or configuration.

For each indexer:
1. Select the indexer.
2. Review its configuration options.
3. Enter any required details.
4. Apply the `bypass` tag if that indexer specifically needs the FlareSolverr proxy.
5. Save the configuration.
6. Test the indexer.

Do not assign the FlareSolverr tag to every indexer by default. Apply it selectively to compatible indexers that actually need it.

### 3.2. Test and Troubleshoot Indexers

After adding an indexer, use Prowlarr's test function to check whether it is working.

If an indexer fails, inspect the error message and logs before changing the configuration.

Common causes include:
- The indexer is temporarily unavailable.
- The indexer's address or implementation has changed.
- A required setting or credential is missing.
- The indexer is blocked by an anti-bot challenge.
- FlareSolverr is unreachable or incompatible.
- The network or DNS configuration is incorrect.
- Outbound connectivity is restricted.

If an indexer fails because of a network or region restriction, verify the network configuration and the service's permitted access requirements. Do not assume that installing FlareSolverr alone will solve every connection issue.

> [!TIP] Configure the VPN separately
> If your setup requires Prowlarr's outbound traffic to pass through Gluetun, ensure the Compose configuration actually routes Prowlarr through Gluetun. That is a Docker networking change, not an indexer setting.
> A VPN may help with connectivity in some circumstances, but it does not guarantee that every indexer will become accessible.

---
## 4. Connect Applications

Prowlarr can synchronize indexers with applications such as Radarr and Sonarr. This avoids having to maintain the same indexer configuration independently in each application.

Before configuring these integrations, make sure Radarr and Sonarr are running and accessible.

### 4.1. Connect Radarr

#### Step 1: Retrieve the API key

Open Radarr and navigate to:

**Settings → General → Security → API Key**

Copy the API key. Keep it private because it grants access to Radarr's API.

#### Step 2: Add Radarr in Prowlarr

In Prowlarr, navigate to:

**Settings → Apps → Add App → Radarr**

Configure the following fields:

|Setting|Value|
|---|---|
|Name|`Radarr`|
|Sync Level|`Full Sync`|
|Prowlarr Tag|Leave empty unless you intend to use tag-based filtering|
|URL / Host|`http://radarr:7878`|
|API Key|The API key copied from Radarr|

The hostname `radarr` assumes that the Compose service is named `radarr` and that both applications share a network on which that name resolves.

Since Radarr is not routed through Gluetun, using the container name will work.

If the Compose service has a different name, use that name instead.

If Prowlarr and Radarr run in different Docker environments or networks, you may need to use a reachable host address or another supported network endpoint.

#### Step 3: Understand Full Sync

`Full Sync` allows Prowlarr to synchronize supported indexer configurations with Radarr.

This is generally convenient when you want Prowlarr to be the central manager for indexers.

### 4.2. Connect Sonarr

#### Step 1: Retrieve the API key

Open Sonarr and navigate to:

**Settings → General → Security → API Key**

Copy the API key.

#### Step 2: Add Sonarr in Prowlarr

In Prowlarr, navigate to:

**Settings → Apps → Add App → Sonarr**

Configure the following fields:

|Setting|Value|
|---|---|
|Name|`Sonarr`|
|Sync Level|`Full Sync`|
|Prowlarr Tag|Leave empty unless you intend to use tag-based filtering|
|URL / Host|`http://sonarr:8989`|
|API Key|The API key copied from Sonarr|

The hostname `sonarr` assumes that the Compose service is named `sonarr` and that both applications share a network on which that name resolves.

Since Sonarr is not routed through Gluetun, using the container name will work.

If the service name differs, replace it with the correct name.

### 4.3. Test Application Connections

For both applications:
1. Save the configuration.
2. Use Prowlarr's **Test** button.
3. Confirm that the connection succeeds.
4. Save the integration if necessary.
5. Check whether indexers synchronize correctly.

If a connection fails, verify the following:
- The target application is running.
- The API key is correct.
- The URL uses the correct hostname and port.
- Prowlarr can reach the application over the Docker network.
- The application is not listening exclusively on an unreachable interface.
- No firewall or network policy is blocking the connection.

> [!IMPORTANT] Docker networking
> When two containers share a Docker network, use the target container's service name and its internal listening port.
> For example, Radarr's internal port is typically `7878`, and Sonarr's internal port is typically `8989`. You do not need to use the host-published port for communication between containers on the same Docker network.

---
## 5. Configure the Download Client

Prowlarr manages indexers and synchronizes them with applications. The download client is configured in Sonarr and Radarr, which use it to download the content they manage.

Prowlarr itself does not normally need qBittorrent configured but since the option is present, we might as well fill it.

### 5.1. Add qBittorrent

#### Step 1: Open the download client settings

Navigate to:

**Settings → Download Clients → Add → qBittorrent**

#### Step 2: Configure the connection

Use the following values as a reference:

| Setting         | Value                                                            |
| --------------- | ---------------------------------------------------------------- |
| Name            | `qBittorrent`                                                    |
| Host            | `qbittorrent`or `gluetun`                                        |
| Port            | `8080`                                                           |
| Username        | Your qBittorrent Web UI username                                 |
| Password        | Your qBittorrent Web UI password                                 |
| Category        | Configure a category appropriate for the application, if desired |
| Client Priority | `1`, if available and appropriate                                |

The hostname `qbittorrent` assumes that the Compose service is named `qbittorrent` and is reachable from Sonarr or Radarr over a shared Docker network.

If you are routing QBittorrent through Gluetun, user `gluetun` in the **Host** field.

The port `8080` is an example and must match the actual port on which the qBittorrent Web UI listens inside the container.

If you have configured a different port, use that value instead.

#### Step 3: Enable advanced settings if necessary

Some optional fields, such as client priority, may only appear when advanced settings are enabled.

Enable advanced settings if you need access to these fields.

#### Step 4: Save and test

Save the configuration and use the **Test** button.

Repeat the same process in Sonarr.

> [!NOTE] Credentials
> The username and password must match the credentials configured in qBittorrent's Web UI settings. Do not copy placeholder credentials from documentation into your actual server.

### 5.2. Test the Download Client

If the connection fails, verify the following:
- qBittorrent is running.
- The Web UI is enabled.
- The hostname resolves from the Sonarr or Radarr container.
- The port is correct.
- The credentials are valid.
- qBittorrent permits the incoming connection from the application.
- Both containers can communicate over their Docker network.

If your setup routes qBittorrent through Gluetun, verify that the relevant ports and network configuration are correct for that arrangement.

Also ensure that the download paths configured in qBittorrent match the paths visible to Sonarr and Radarr. Consistent container paths simplify imports and prevent path-mapping problems.

---
## 6. Troubleshooting

### Prowlarr's web interface does not load

Check whether the container is running:

```bash
docker ps --filter "name=prowlarr"
```

Inspect its logs:

```bash
docker logs --tail=100 prowlarr
```

Confirm that the host port mapping includes the expected port, typically `9696`.

### FlareSolverr is unreachable

Inspect the service:

```bash
docker ps --filter "name=flaresolverr"
docker logs --tail=100 flaresolverr
```

Verify that Prowlarr and FlareSolverr share a network and that the proxy URL uses the correct service name and internal port.

### An indexer fails its test

Check the error message in Prowlarr and inspect its logs:

```bash
docker logs --tail=100 prowlarr
```

Check whether the indexer is currently supported, whether its configuration is correct, and whether the failure relates to DNS, network connectivity, rate limits, or anti-bot protection.

### Radarr or Sonarr cannot be reached

Verify that the target container is running and that the API key is correct.

Check the Docker networks attached to both containers:

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' prowlarr | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' radarr | jq
docker inspect -f '{{json .NetworkSettings.Networks}}' sonarr | jq
```

Confirm that the applications share an appropriate network or have another valid route between them.

### qBittorrent connection fails

Check qBittorrent's logs and verify its Web UI settings.

Make sure the configured host and port refer to the Web UI endpoint, not the BitTorrent peer-listening port.

If qBittorrent is behind Gluetun, confirm that its management interface remains reachable by the applications that need to use it.

---

> [!TIP] Setup Complete
> Prowlarr and Falresolverr are ready to act as the indexer providers for your server!

---

> [!TIP] ➜ Up Next
> You’ve completed  [03 Prowlarr and FlareSolverr](#03-prowlarr-and-flaresolverr)
> **Previous:** [02 Gluetun](02%20Gluetun.md)  ------ **Next:** [04 qBittorrent](04%20qBittorrent.md)
