# Overview

Gluetun is a VPN client container for Docker. It allows selected containers to route their network traffic through a VPN without requiring the entire Ubuntu server to use that VPN.

In this media server setup, Gluetun can provide VPN connectivity for qBittorrent and other selected services while the remaining applications continue using their normal Docker network connections.

> [!NOTE] Overview
> - **Docker container:** `gluetun`
> - **Configuration directory:** `/srv/docker/configs/gluetun`
> - **Docker network:** `media_net` for services that use the regular Docker network
> - **VPN provider:** Proton VPN
> - **VPN configuration:** Environment variables in `.env`
> - **Purpose:** Route selected containers through a VPN

> [!Important] Gluetun Is Optional
> Gluetun is not required for the media server to function. You can use a system-wide VPN or another supported networking arrangement instead.
> The main advantage of Gluetun is that it allows you to choose which Docker containers use the VPN, rather than routing all host traffic through it.

> [!Note] Alternative
> If you do not want to go through the configuration of Gluetun and, are fine with running a system-wide VPN, you can just use that and skip this entire file. 

## Table of Contents
- [1. Installation Checks](#1-installation-checks)
- [2. Configure VPN Credentials](#2-configure-vpn-credentials)
- [3. Start and Verify Gluetun](#3-start-and-verify-gluetun)
- [4. Route Services Through Gluetun](#4-route-services-through-gluetun)
- [5. Connecting to Services Behind Gluetun](#5-connecting-to-services-behind-gluetun)
- [6. Troubleshooting](#6-troubleshooting)

---
## 1. Installation Checks

Gluetun is already installed, so we will verify its configuration directory, Compose service, and current status before configuring VPN routing.

### 1.1 Create the Configuration Directory

Run the following command on the Ubuntu host:

```bash
# Create the Gluetun configuration directory
sudo mkdir -p /srv/docker/configs/gluetun
```

If your Gluetun image or Compose configuration requires this directory to be writable, set its ownership and permissions accordingly:

```bash
# Set ownership, if required by your configuration
sudo chown -R 1000:1000 /srv/docker/configs/gluetun

# Set basic permissions
sudo chmod -R 775 /srv/docker/configs/gluetun
```

> [!TIP] Permissions
> Gluetun's VPN configuration is commonly supplied through environment variables rather than files in this directory. If the directory is used for configuration files, certificates, or other data, verify the image's requirements before changing ownership or permissions.

### 1.2 Verify the Docker Compose Configuration

Navigate to the media stack directory:

```bash
cd /srv/docker/stacks/media-stack
```

Validate the Compose file and inspect the container:

```bash
# Validate the Compose configuration
sudo docker compose config --quiet

# Check Gluetun's current status
sudo docker compose ps gluetun

# Inspect recent logs
sudo docker compose logs --tail=100 gluetun
```

Make sure the service uses the intended VPN provider, VPN protocol, credentials, and region settings.

---
## 2. Configure VPN Credentials

Each VPN provider has its own method of issuing credentials for VPN connections. Some providers use account credentials, while others require dedicated service credentials or configuration files.

For Proton VPN, follow the current official documentation to obtain the credentials and configuration required by your chosen protocol.
- [Proton VPN Support](https://protonvpn.com/support/)
- [Gluetun Documentation](https://github.com/qdm12/gluetun)

### 2.1 Update the `.env` File

Open your media stack's `.env` file and update the Gluetun variables to match the provider's current requirements.

For example, your existing configuration may contain variables such as:

```properties
VPN_SERVICE_PROVIDER=protonvpn
VPN_TYPE=wireguard
WIREGUARD_MTU=1420
```

These are examples based on the existing setup. Keep the other required variables and credentials that your current Gluetun version expects.

> [!IMPORTANT] Keep VPN Credentials Private
> - Do not paste VPN credentials into screenshots, public issue trackers, or shared logs.
> - Keep `.env` permissions restricted to the users who need access.
> - Use a `.env.example` file with placeholder values when documenting the configuration.
> Environment variable names and authentication requirements can change between Gluetun versions and VPN providers. Follow the documentation for the exact image version you use.

### 2.2 Choose the VPN Region

Configure the country, region, or server selection according to the options supported by your provider and Gluetun version.

The chosen location can affect:
- The public IP address visible to external services.
- Connection latency and throughput.
- Access to region-specific services.
- The availability of port forwarding, if supported by the provider and server.

Do not assume that every VPN server supports port forwarding. This matters for torrent clients that rely on an incoming peer port.

---
## 3. Start and Verify Gluetun

### 3.1 Start the Container

Use Docker Compose V2:

```bash
cd /srv/docker/stacks/media-stack

# Pull the configured Gluetun image
sudo docker compose pull gluetun

# Start or recreate Gluetun
sudo docker compose up -d gluetun

# Check its status
sudo docker compose ps gluetun

# Follow its logs
sudo docker compose logs -f gluetun
```

Wait for the logs to indicate that the VPN connection has been established. Press `Ctrl+C` to stop following the logs without stopping the container.

### 3.2 Verify the Public IP Address

Once Gluetun reports a successful connection, run:

```bash
sudo docker exec gluetun wget -qO- https://ipinfo.io
```

This requests public IP information from inside the Gluetun container. If the command succeeds, the response should contain details similar to the following:

```json
{
  "ip": "YOUR_VPN_PUBLIC_IP",
  "city": "VPN_SERVER_CITY",
  "region": "VPN_SERVER_REGION",
  "country": "VPN_SERVER_COUNTRY",
  "org": "VPN_PROVIDER_OR_NETWORK"
}
```

The exact fields depend on the lookup service's response.

Verify that the returned public IP and location match the VPN connection you intended to use. The `org` field may identify the VPN provider or the network through which the connection exits.

> [!WARNING] A Successful IP Lookup Is Not a Complete Leak Test
> This confirms the public IP seen by that request. It does not, by itself, prove that every routed container is protected against leaks or that traffic cannot escape through another route. Verify the networking configuration and test each routed service separately.

If the command fails, check the Gluetun logs before assuming that the VPN connection is broken. The image may lack the requested utility, or the external lookup service may be unreachable.

### 3.3 Verify the Container Health

Run:

```bash
sudo docker inspect --format='{{.State.Status}}' gluetun
```

If the container has a configured health check, inspect its health status:

```bash
sudo docker inspect --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}No health check configured{{end}}' gluetun
```

A running container is not necessarily a successfully connected VPN. Confirm the connection in the logs and verify the public IP as described above.

---
## 4. Route Services Through Gluetun

Gluetun does not automatically route every container through the VPN. You must explicitly configure the services that should share its network namespace.

In this media stack, the intended routed services are:
- qBittorrent
- Jellyseerr
- FlareSolverr
- Prowlarr

Only route a service through Gluetun if its traffic should use the VPN and you have verified that its required connections will still work.

> [!Important] Provided compose file handles this already
> The Compose file provided in [01 Docker Compose](../01%20Configuration/01%20Docker%20Compose.md) already handles the routing of the required services through gluetun.
> The content written below is only for understanding or if you want to tweak your routing.

### 4.1 Understand the Network Configuration

A typical standalone service might use its own port mappings and join `media_net`:

```yaml
services:
  qbittorrent:
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    networks:
      - media_net
```

To make qBittorrent share Gluetun's network namespace, the service configuration changes to:

```yaml
services:
  qbittorrent:
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
```

This is an illustrative excerpt, not a complete Compose service definition. Preserve qBittorrent's image, environment variables, volumes, restart policy, and other required settings.

> [!IMPORTANT] Do Not Combine `network_mode` With `networks`
> A service using `network_mode: "service:gluetun"` cannot also declare its own Compose `networks` list. It shares Gluetun's network namespace instead.
> This also means qBittorrent does not get its own network interface or independent IP address. Its network traffic uses Gluetun's network namespace.

### 4.2 Publish the Required Ports on Gluetun

When qBittorrent shares Gluetun's network namespace, publish qBittorrent's required ports on the **Gluetun service**, not on the qBittorrent service.

For example:

```yaml
services:
  gluetun:
    ports:
      - "8080:8080"       # qBittorrent Web UI
      - "6881:6881"       # Example incoming peer port
      - "6881:6881/udp"   # Example UDP peer traffic
```

These mappings are examples. Keep any existing Gluetun ports and add only those required by your actual configuration.

**About the Gluetun API port:** Only publish an API port such as `8888` if you have enabled and configured the API, understand its access controls, and actually need it. Do not expose administrative APIs to untrusted networks.

**About the qBittorrent peer port:** `6881` is an example, not a requirement. Use the incoming port configured in qBittorrent and any port-forwarding configuration required by your VPN provider. Publishing a port on the Docker host does not automatically make it reachable through the VPN provider's network.

### 4.3 Apply the Changes

After editing the Compose file, validate it before recreating the services:

```bash
cd /srv/docker/stacks/media-stack

# Validate the configuration
sudo docker compose config --quiet

# Recreate the affected services
sudo docker compose up -d gluetun qbittorrent
```

If you change the network configuration of multiple services, recreate each affected service as appropriate.

Verify that Gluetun connects successfully and that the dependent applications start correctly:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 gluetun
sudo docker compose logs --tail=100 qbittorrent
```

### 4.4 Repeat for Other Services

For each additional service that should share Gluetun's network namespace, use the same network-mode pattern:

```bash
network_mode: "service:gluetun"
depends_on:
  - gluetun
```

However, do not blindly copy the configuration to every application. Jellyseerr, Prowlarr, and FlareSolverr have different connection requirements, and their internal service-to-service communication may need to be reconfigured after routing changes.

---
## 5. Connecting to Services Behind Gluetun

This is the part that can be confusing when using Gluetun. Containers sharing Gluetun's network namespace do not behave like independent containers on `media_net`.

### 5.1 Example: Accessing qBittorrent From Radarr or Sonarr

Suppose qBittorrent shares Gluetun's network namespace.

Radarr and Sonarr still run as independent containers. They cannot necessarily resolve `qbittorrent` as a separate network endpoint because qBittorrent no longer has its own network identity.

In that case, the connection may use a reachable host address and the published qBittorrent Web UI port, such as:

| Setting            | Example                      |
| ------------------ | ---------------------------- |
| Host               | `gluetun` or `SERVER_LAN_IP` |
| Port               | `8080`                       |
| Category in Radarr | `radarr`                     |
| Category in Sonarr | `sonarr`                     |

Replace `SERVER_LAN_IP` with the actual address reachable from the Radarr and Sonarr containers. This approach depends on the host's network configuration and the port being published correctly.

> [!TIP] Alternative: Shared Network Namespace
> If a client is also configured to share Gluetun's network namespace, it can reach qBittorrent through `localhost` and the relevant internal port. This is different from connecting to qBittorrent from a normal independent container.

Keep Radarr and Sonarr's download-client settings aligned with the actual network topology.

### 5.2 Example: Connecting Prowlarr to Radarr and Sonarr

Prowlarr must be able to reach Radarr and Sonarr to synchronize indexers.

If Prowlarr shares Gluetun's network namespace while Radarr and Sonarr remain on `media_net`, the service names `radarr` and `sonarr` may resolve from Prowlarr's network namespace bun under **Prowlarr Server**, you may have to fill `http://gluetun:9696`.

### 5.3 Example: Connecting Prowlarr to FlareSolverr

If Prowlarr and FlareSolverr both share Gluetun's network namespace, they share the same network interfaces. In this arrangement, FlareSolverr can commonly be reached through:

`http://127.0.0.1:8191`
or
`http://gluetun:8191`

This assumes FlareSolverr listens on port `8191` and both services really do share the same network namespace.

If they use separate Docker networks instead, use a reachable service name and port.

---

## 6. Troubleshooting

### 6.1 Gluetun Starts but Cannot Connect to the VPN

Inspect the logs:

```bash
sudo docker compose -f /srv/docker/stacks/media-stack/docker-compose.yml logs --tail=200 gluetun
```

Check:
- VPN provider and protocol settings.
- Credentials and required configuration variables.
- Country or server selection.
- Host DNS and general internet connectivity.
- Provider status and current authentication requirements.
- Whether the container is using the expected Compose environment file.

Do not paste unredacted credentials into public support posts.

### 6.2 Public IP Does Not Match the Expected VPN Location

First confirm that the IP lookup was executed inside Gluetun:

```bash
sudo docker exec gluetun wget -qO- https://ipinfo.io
```

Then check the VPN connection status in the logs. A VPN exit server may report a different city or region than expected, so verify the public IP and provider rather than relying exclusively on the city field.

If a dependent container is supposed to use Gluetun, test that container's outbound IP as well. Do not assume that all containers are routed through the VPN just because Gluetun is connected.

### 6.3 qBittorrent Web UI Is Unreachable

Check that:
- The Web UI is enabled in qBittorrent.
- Its configured port matches the Compose port mapping.
- The mapping is declared on Gluetun when qBittorrent shares Gluetun's network namespace.
- Gluetun is running and healthy.
- The host firewall allows access from the intended network.
- The Web UI is not bound to an unexpected address or blocked by application settings.

Inspect logs:

```bash
sudo docker compose logs --tail=100 gluetun
sudo docker compose logs --tail=100 qbittorrent
```

### 6.4 Radarr or Sonarr Cannot Connect to qBittorrent

Confirm the host and port in the download-client settings. If qBittorrent shares Gluetun's network namespace, the hostname `qbittorrent` may not work from Radarr or Sonarr.

Use an endpoint reachable from the Radarr or Sonarr container, and confirm that the qBittorrent Web UI port is published appropriately.

Also verify that the qBittorrent username and password are correct.

### 6.5 Prowlarr or Jellyseerr Cannot Reach Other Services

Check which containers share Gluetun's network namespace and which remain on `media_net`.

Services sharing a network namespace may use `localhost` to communicate with one another, provided the target application listens on the expected port. Independent containers usually need a reachable service name or host address.

Inspect the Compose configuration and logs to identify which endpoint is failing before changing the networking arrangement.

### 6.6 VPN Port Forwarding Is Not Working

A Docker port mapping alone does not guarantee inbound peer connectivity through the VPN.

Verify:
- Whether your VPN provider supports port forwarding on the selected server.
- Which incoming port the provider has assigned.
- Whether qBittorrent is configured to use that port.
- Whether Gluetun and the host firewall permit the required traffic.
- Whether the published Docker port matches the actual listening port.

Use the provider's supported port-forwarding procedure rather than assuming a fixed port such as `6881` will work.

---

> [!TIP] Setup Complete
> Gluetun is ready to provide VPN connectivity to the services configured to use it. Verify the routing of each container individually, since a working VPN connection in Gluetun does not automatically mean every application is using it.

---

> [!TIP] ➜ Up Next
> You’ve completed  [02 Gluetun](#02-gluetun)
> **Previous:** [01 Docker](01%20Docker.md)  ------ **Next:** [03 Prowlarr and FlareSolverr](03%20Prowlarr%20and%20FlareSolverr.md)
