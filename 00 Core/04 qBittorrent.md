# Overview

qBittorrent is the download client used by the media server to download content identified by applications such as Sonarr and Radarr. It provides a Web UI for managing downloads, monitoring progress, and configuring download locations and connection settings.

This guide covers verifying the existing installation and configuring the Web UI.

> [!NOTE] Prerequisites
> Before proceeding, ensure that:
> - Docker Engine and Docker Compose are installed and running.
> - qBittorrent has already been deployed as part of the media stack.
> - The configuration and download directories exist or can be created.
> - Gluetun is running if qBittorrent is configured to use it as its VPN gateway.
> Refer to [01 Docker](01%20Docker.md) for the initial Docker configuration.

## Table of Contents
- [1. Installation & Verification](#1-installation-verification)
- [2. Initial Setup](#2-initial-setup)
    - [2.1. Access the Web UI](#21-access-the-web-ui)
    - [2.2. Configure Login Credentials](#22-configure-login-credentials)
- [3. Configure Download Directories](#3-configure-download-directories)
    - [3.1. Create the Host Directories](#31-create-the-host-directories)
    - [3.2. Configure Docker Bind Mounts](#32-configure-docker-bind-mounts)
    - [3.3. Set the Download Path](#33-set-the-download-path)
- [4. Configure Connection Settings](#4-configure-connection-settings)
    - [4.1. Peer Connection Protocol](#41-peer-connection-protocol)
    - [4.2. Incoming Connection Port](#42-incoming-connection-port)
    - [4.3. Gluetun Port Forwarding](#43-gluetun-port-forwarding)
- [5. Configure Seeding Behavior](#5-configure-seeding-behavior)
- [6. Troubleshooting](#6-troubleshooting)

---
## 1. Installation & Verification

qBittorrent has already been installed as part of the media stack. We only need to verify that the container is running before opening its Web UI.

Check the container status:

```bash
docker ps \
  --filter "name=qbittorrent" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

If the container is running, the output should display its name, status, and port mappings.

If the container is missing or stopped, check all containers:

```bash
docker ps -a --filter "name=qbittorrent"
```

Inspect the logs if necessary:

```bash
docker logs --tail=200 qbittorrent
```

If the container is defined in the main media stack's Compose file, you can start it using:

```bash
cd /srv/docker/stacks/media-stack
docker compose up -d qbittorrent
```

> [!NOTE] Container name
> The commands above assume that the container is named `qbittorrent`. If your Compose configuration uses a different container name, replace it accordingly.

## 2. Initial Setup

### 2.1. Access the Web UI

Open the qBittorrent Web UI in your browser:

```bash
http://localhost:8080
```

If accessing the server from another computer, replace `localhost` with the server's IP address:

```bash
http://SERVER_IP:8080
```

For example:

```bash
http://192.168.1.100:8080
```

This assumes that host port `8080` is published by the container. If the Web UI uses another port, use the configured host port instead.

### 2.2. Configure Login Credentials

On the first login, qBittorrent may require you to authenticate using its initial or automatically generated credentials.

The credentials depend on the image and version being used.
- **Default username:** `admin` for configurations that use the standard default.
- **Default password:** Do not assume `adminadmin` will work. Some versions and container images generate a temporary password, which is printed in the container logs.

If you cannot log in, inspect the logs:

```bash
docker logs --tail=200 qbittorrent
```

Look for a message containing the temporary Web UI password or authentication information.

After logging in, change the credentials.

Navigate to:

**Tools / Settings → Web UI → Authentication**

The exact menu placement may differ between versions.

Configure a strong username and password, then save the settings.

> [!WARNING] Secure the Web UI
> Do not expose the qBittorrent Web UI directly to the public internet. Keep it accessible only through your trusted LAN, VPN, or another appropriately secured access method.
> If qBittorrent is routed through Gluetun, ensure that the Web UI remains reachable from the management devices and containers that need to access it.

---
## 3. Configure Download Directories

qBittorrent needs a persistent directory in which to store downloaded files.

In this setup, the host stores downloads under `/mnt/media-hdd/downloads`, while the container accesses the same files through `/downloads`.

This separation allows the host and containers to use different paths for the same underlying files.

### 3.1. Create the Host Directories

Create the downloads directory:

```bash
sudo mkdir -p /mnt/media-hdd/downloads
```

If the directory already exists, the command will leave it in place.

Ensure the directory ownership matches the UID and GID used by the qBittorrent container.

For a setup using UID `1000` and GID `1000`:

```bash
sudo chown -R 1000:1000 /mnt/media-hdd/downloads
sudo chown -R 1000:1000 /srv/docker/configs/qbittorrent
```

> [!IMPORTANT] Match the container configuration
> UID `1000` and GID `1000` are examples commonly used in LinuxServer.io container configurations. Confirm the actual values in your Compose file and the documentation for your selected qBittorrent image.
> Avoid recursively changing ownership of existing directories without first confirming that the changes are appropriate.

### 3.2. Configure Docker Bind Mounts

The host directory must be mounted into the container using a bind mount.

In the media stack's `.env` file, the downloads root may be defined as:

```properties
DOWNLOADS_ROOT=/mnt/media-hdd/downloads
```

In `docker-compose.yml`, configure the qBittorrent service with the following volume mapping:

```yaml
volumes:
  - ${DOWNLOADS_ROOT}:/downloads
```

This mapping means:

| Location  | Path                       | Purpose                                                 |
| --------- | -------------------------- | ------------------------------------------------------- |
| Host      | `/mnt/media-hdd/downloads` | Stores downloaded files on the server.                  |
| Container | `/downloads`               | Allows qBittorrent to access the same files internally. |

The mapping follows this format:

```yaml
host_path:container_path
```

For example, a file stored at `/downloads/movies/example.mkv` inside the container corresponds to `/srv/docker/downloads/movies/example.mkv` on the host.

> [!TIP] Keep paths consistent
> If Sonarr and Radarr also need to access these downloads, configure their volume mappings so that the applications can see the same underlying files.
> Consistent container paths simplify imports and help prevent path-mapping errors.

### 3.3. Set the Download Path

Open qBittorrent and navigate to:

**Tools / Settings → Downloads**

Set the default save path to:

```
/downloads
```

If you want to organize downloads into separate categories, you can create subdirectories, for example:

```
/downloads/movies
/downloads/tv
/downloads/anime
```

Create the corresponding directories on the host **if required**:

```bash
sudo mkdir -p /srv/docker/downloads/{movies,tv,anime}
```

If your application workflow uses categories, configure the appropriate category names in qBittorrent and ensure Sonarr and Radarr use matching categories.

> [!NOTE] Do not confuse host and container paths
> The path entered in qBittorrent's settings is the path **inside the container**, not the absolute path on the host.
> Use `/downloads`, not `/mnt/media-hdd/downloads`, unless your Compose file explicitly mounts the host directory at that same internal path.

---
## 4. Configure Connection Settings

The connection settings control how qBittorrent communicates with peers and how other peers can establish incoming connections.

### 4.1. Peer Connection Protocol

Navigate to:

**Tools / Settings → Connection**

Under the peer connection protocol setting, select:

**TCP and µTP**

This enables both TCP and µTP peer connections.

The appropriate configuration depends on your network, VPN provider, and performance requirements.

### 4.2. Incoming Connection Port

Under **Connection**, locate the incoming connection port setting.

qBittorrent uses this port to accept incoming peer connections when the network and firewall permit them.

For a VPN-based setup, the required port may be assigned or forwarded by the VPN provider.

> [!IMPORTANT] Do not change the port blindly
> Leave the configured port unchanged unless you have a specific reason to modify it.
> If your VPN provider supports port forwarding, qBittorrent's incoming connection port should match the port forwarded by the VPN gateway, subject to the provider's requirements.

### 4.3. Gluetun Port Forwarding

If qBittorrent routes its traffic through Gluetun and your VPN provider supports port forwarding, you may need to retrieve the forwarded port and configure qBittorrent accordingly.

Check the Gluetun logs:

```bash
docker logs gluetun 2>&1 | grep -i port
```

This searches the logs for messages containing `port`.

Review the output for the port assigned or forwarded by your VPN configuration.

The exact message depends on the Gluetun version, VPN provider, and forwarding configuration. If the command returns nothing, inspect the complete logs:

```bash
docker logs --tail=200 gluetun
```

Then verify the VPN provider's port-forwarding documentation.

> [!WARNING] Port forwarding is provider-dependent
> Not every VPN provider or server supports port forwarding. The forwarded port may also change when the VPN connection is re-established.
> If your provider assigns a new port, update qBittorrent's incoming connection port to match it. Do not assume that the port remains fixed across server restarts or VPN reconnections.

If your VPN setup requires a different mechanism for communicating the assigned port to qBittorrent, follow the relevant Gluetun and container configuration instructions.

---
## 5. Configure Seeding Behavior

Seeding allows qBittorrent to continue uploading downloaded content to other peers after a download completes.

If you want qBittorrent to stop seeding automatically, configure its seeding limits.

Navigate to:

**Tools / Settings → BitTorrent → Seeding Limits**

Enable the relevant seeding limit and configure:

**When ratio reaches:** `0`

This setting is intended to stop seeding once the configured share ratio reaches zero, subject to the seeding-limit options enabled in your version of qBittorrent.

> [!IMPORTANT] Verify the complete seeding policy
> Setting a ratio value alone may not activate the policy. Ensure the relevant seeding-limit option is enabled and configured to apply to the torrents you want to manage.
> If your goal is to stop seeding immediately after a download finishes, check whether your qBittorrent version offers a suitable stop condition or category-specific seeding rule. A ratio limit of zero is not the same as a time-based limit.

If you prefer to continue seeding for a particular period or until a particular ratio is reached, configure the appropriate limits instead.

---
## 6. Troubleshooting

### qBittorrent's Web UI does not load

Check whether the container is running:

```bash
docker ps --filter "name=qbittorrent"
```

Inspect the logs:

```bash
docker logs --tail=200 qbittorrent
```

Verify that the expected host port is published and that no other service is using it.

### Login credentials do not work

Inspect the container logs:

```bash
docker logs --tail=200 qbittorrent
```

Look for temporary Web UI credentials or authentication-related errors. Check the documentation for the container image you are using.

### qBittorrent cannot write to the download directory

Check the host directory's ownership and permissions:

```bash
ls -ld /mnt/media-hdd/downloads
ls -ld /srv/docker/configs/qbittorrent
```

Confirm that the effective user inside the container has permission to write to the mounted directories.

Also verify the volume mapping in `docker-compose.yml`.

### Downloads appear in qBittorrent but not in Sonarr or Radarr

This is commonly caused by inconsistent paths between containers.

For example, qBittorrent may save a file to `/downloads/movies/example.mkv`, while Sonarr expects to find it at another internal path.

Ensure that:
- qBittorrent and the relevant application can access the same underlying directory.
- The internal container paths are consistent or correctly mapped.
- The configured download category matches the application.
- The application has permission to read the downloaded files.

### Incoming connections are not working

Check the configured incoming port and inspect the Gluetun logs if the VPN supports port forwarding.

Confirm that:
- The VPN connection is active.
- The provider supports port forwarding.
- The forwarded port matches the configured qBittorrent port.
- The Docker network and firewall configuration allow the required traffic.

A successful download does not necessarily mean that incoming peer connectivity is configured correctly.

> [!TIP] qBittorrent configuration complete
> qBittorrent is ready to serve as the media server's download client. The next step is to verify its integration with Sonarr and Radarr and confirm that downloaded files can be imported into the media library.

---

> [!TIP] ➜ Up Next
> You’ve completed  [04 qBittorrent](#04-qbittorrent)
> **Previous:** [03 Prowlarr and FlareSolverr](03%20Prowlarr%20and%20FlareSolverr.md)  ------ **Next:** [05 Radarr](05%20Radarr.md)
