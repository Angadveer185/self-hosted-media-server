# Overview

Cloudflare Tunnel (`cloudflared`) creates an outbound connection from your server to Cloudflare. This allows you to make selected self-hosted applications available through a domain name without directly exposing your server's inbound ports to the internet.

For example, you could access Jellyfin through a domain such as `watch.example.com` instead of connecting to your server's IP address and port.

The traffic flow looks like this:

```text
User's Browser
      |
      v
Cloudflare Edge
      |
      v
Cloudflare Tunnel
      |
      v
cloudflared Container
      |
      v
Jellyfin / Other Configured Service
```

> [!NOTE] Overview
> - **Service:** Cloudflare Tunnel
> - **Docker container:** `cloudflared`
> - **Configuration:** Cloudflare Tunnel token stored in `.env`
> - **Domain management:** Cloudflare DNS
> - **Purpose:** Securely route external requests to selected self-hosted applications
> - **Inbound router port forwarding:** Not required for the tunnel itself

## Table of Contents
- [1. Purchase a Custom Domain](#1-purchase-a-custom-domain)
- [2. Add Your Domain to Cloudflare](#2-add-your-domain-to-cloudflare)
- [3. Configure Your Domain's Nameservers](#3-configure-your-domains-nameservers)
- [4. Create a Cloudflare Tunnel](#4-create-a-cloudflare-tunnel)
- [5. Configure the Docker Service](#5-configure-the-docker-service)
- [6. Verify the Tunnel Connection](#6-verify-the-tunnel-connection)
- [7. Publish Applications Through the Tunnel](#7-publish-applications-through-the-tunnel)
- [8. Configure Access Protection](#8-configure-access-protection)
- [9. Troubleshooting](#9-troubleshooting)

---
## 1. Purchase a Custom Domain

To access your media server using a memorable URL, you need a domain name.

Some domain registrars you can explore include:
- [Namecheap](https://www.namecheap.com/)
- [Spaceship](https://www.spaceship.com/)
- Other domain registrars of your choice

For example, you might purchase a domain such as:

`example.me`

You can then create subdomains for individual services:
- `watch.example.me` for Jellyfin
- `requests.example.me` for Jellyseerr
- `status.example.me` for a status page, if you run one

These are examples only. Replace them with your own domain and preferred subdomain names.

> [!TIP] Choose a Domain You Can Maintain
> Compare renewal prices, DNS management options, and domain transfer policies before purchasing. A low introductory price does not necessarily mean a low renewal price.

---
## 2. Add Your Domain to Cloudflare

Cloudflare will manage the domain's DNS records and route requests to your tunnel.

### 2.1 Open the Cloudflare Dashboard
1. Visit the [Cloudflare Dashboard](https://dash.cloudflare.com/).
2. Sign in or create a Cloudflare account.
3. From **Account Home**, select the option to add a domain or website.
4. Enter your domain name, such as `example.me`.
5. Select the Free plan if it meets your requirements.
6. Continue through the activation process.

Cloudflare will scan for existing DNS records where possible. Review the imported records carefully before proceeding.

> [!WARNING] Existing DNS Records
> If your domain already hosts a website, email service, or other services, verify that their DNS records are preserved. Changing nameservers transfers DNS management to Cloudflare, and missing records can interrupt existing services.

### 2.2 Obtain the Cloudflare Nameservers

Cloudflare will provide two authoritative nameservers for your domain.

Example:

```text
jasmine.ns.cloudflare.com
miles.ns.cloudflare.com
```

These names are illustrative. Use the exact nameservers shown in your Cloudflare account.

Keep the Cloudflare dashboard open because you will need these values in the next step.

---
## 3. Configure Your Domain's Nameservers

You must tell your domain registrar to delegate DNS management to Cloudflare.

### 3.1 Update the Nameservers at Your Registrar

For a domain registered through Namecheap, the process is generally:
1. Sign in to your [Namecheap account](https://www.namecheap.com/).
2. Open **Domain List**.
3. Select **Manage** next to your domain.
4. Find the **Nameservers** setting.
5. Select **Custom DNS**.
6. Enter the two nameservers provided by Cloudflare.
7. Save the changes.

Other registrars have similar settings, although the labels may differ.

### 3.2 Wait for Activation

Return to the Cloudflare Dashboard and check your domain's status.

Nameserver changes can take time to propagate. Cloudflare will indicate when it has detected the new configuration.

> [!TIP] Verify Before Continuing
> Do not assume that the domain is active immediately after saving the nameservers. Wait until Cloudflare confirms activation before troubleshooting the tunnel or published application routes.

---
## 4. Create a Cloudflare Tunnel

A Cloudflare Tunnel connects your server to Cloudflare through an outbound connection.

### 4.1 Create the Tunnel
1. Open the [Cloudflare Dashboard](https://dash.cloudflare.com/).
2. Select your account.
3. Navigate to **Networking → Tunnels**, or the equivalent Zero Trust networking section in your dashboard.
4. Select **Create Tunnel**.
5. Choose the Cloudflared connector.
6. Give the tunnel a meaningful name, such as `media-server`.
7. Select Docker as the installation environment if prompted.

Cloudflare will display a command for running the tunnel connector in Docker.

### 4.2 Copy the Tunnel Token

The generated command may resemble:

```bash
docker run cloudflare/cloudflared:latest tunnel \
  --no-autoupdate run \
  --token YOUR_LONG_TUNNEL_TOKEN
```

The actual token is the long value supplied after `--token`.

Copy the token from your own Cloudflare dashboard. Never use the example token or publish your real token in documentation.

> [!IMPORTANT] Treat the Tunnel Token Like a Secret
> Anyone who obtains the token may be able to run a connector for your tunnel. Store it in `.env`, exclude that file from version control, and rotate the token if it is exposed.

### 4.3 Add the Token to `.env`

Open the `.env` file used by your media stack and add:

```properties
CLOUDFLARE_TUNNEL_TOKEN=YOUR_LONG_TUNNEL_TOKEN
```

Replace the placeholder with the actual token.

If your Compose configuration uses a different environment variable name, keep the name expected by the existing service definition.

---
## 5. Configure the Docker Service

Your media stack already uses Docker Compose, so manage Cloudflared through the existing Compose file rather than starting a separate unmanaged container.

### 5.1 Start or Restart Cloudflared

Run:

```bash
cd /srv/docker/stacks/media-stack

# Validate the Compose configuration
sudo docker compose config --quiet

# Start or recreate Cloudflared
sudo docker compose up -d cloudflared

# Check the container status
sudo docker compose ps cloudflared
```

If you have only changed the token and the existing container configuration is correct, a restart may be sufficient:

```bash
sudo docker compose restart cloudflared
```

Use `docker compose up -d cloudflared` after changes to the Compose service definition, because a plain restart does not apply changed container configuration.

---
## 6. Verify the Tunnel Connection

### 6.1 Inspect the Logs

Run:

```bash
sudo docker logs --tail=100 cloudflared
```

Or use Compose:

```bash
cd /srv/docker/stacks/media-stack
sudo docker compose logs --tail=100 cloudflared
```

Look for messages indicating that the connector has registered and established connections with Cloudflare.

The exact log wording may vary by version. A container that is merely running does not necessarily mean the tunnel is connected successfully.

### 6.2 Check the Cloudflare Dashboard

Return to the Cloudflare Dashboard and open your tunnel.

Verify that its connector status shows as **Healthy** or **Connected**, depending on the dashboard version.

If it remains disconnected, check the logs for authentication failures, DNS problems, network restrictions, or an invalid token.

> [!TIP] Tunnel Connected
> Once Cloudflare confirms the tunnel is connected, you can configure published application routes. The tunnel itself does not automatically publish every application on your server.

---
## 7. Publish Applications Through the Tunnel

Published application routes tell Cloudflare which internal service should receive requests for a particular hostname.

### 7.1 Add a Published Application Route

In the Cloudflare Dashboard:
1. Navigate to **Networking → Tunnels**.
2. Select your tunnel.
3. Open the route or published-application configuration section.
4. Choose **Add Route** or **Add a published application**, depending on the current dashboard interface.
5. Enter the hostname and internal service address.

For example, a Jellyfin route could use the following settings:

| Setting     | Example                |
| ----------- | ---------------------- |
| Subdomain   | `watch`                |
| Domain      | `example.me`           |
| Service URL | `http://jellyfin:8096` |

This example assumes Jellyfin is reachable by the Docker service name `jellyfin` from the Cloudflared container, and that Jellyfin listens on port `8096` inside the container.

The resulting public URL would be:

`https://watch.example.me`

Replace the example domain with your actual domain.

### 7.2 Example Routes for the Media Server

You could create routes such as these:

| Application | Example Hostname    | Example Internal URL   |
| ----------- | ------------------- | ---------------------- |
| Jellyfin    | `watch.example.me`  | `http://jellyfin:8096` |
| Radarr      | `radarr.example.me` | `http://radarr:7878`   |
| Sonarr      | `sonarr.example.me` | `http://sonarr:8989`   |

These are examples, not a recommendation to expose every service publicly. Verify each application's actual internal port and network accessibility before creating a route.

### 7.3 Services Routed Through Gluetun

For a service that shares Gluetun's network namespace, the original service name may not resolve from Cloudflared.

| Application | Example Hostname      | Example Internal URL  |
| ----------- | --------------------- | --------------------- |
| Jellyseerr  | `request.example.me`  | `http://gluetun:5056` |
| QBittorrent | `download.example.me` | `http://gluetun:8080` |
| Prowlarr    | `indexers.example.me` | `http://gluetun:8989` |

This only works if Cloudflared can resolve and reach `gluetun` over a shared or otherwise reachable Docker network, and qBittorrent's Web UI is listening on port `8080` in that namespace.

> [!WARNING] Jellyseerr Port
> The commonly used internal Web UI port for Jellyseerr is `5055`. Your host may publish it as `5056:5055`, which means the external host port is `5056` while other containers on the Docker network typically use `5055`.
> Confirm your Compose configuration before creating the route.

Do not use a container IP copied from `docker inspect` as a permanent configuration unless you have a deliberate reason to do so. Container IPs can change when containers are recreated.

> [!IMPORTANT] Do Not Publish Everything
> Jellyfin is a common candidate for remote access. Administrative applications such as qBittorrent, Radarr, Sonarr, and Prowlarr should generally remain private unless you have a specific need to access them remotely and have configured appropriate authentication and access controls.

---
## 8. Configure Access Protection

A tunnel encrypts the connection between your server and Cloudflare, and Cloudflare provides HTTPS for the public hostname. However, this does not automatically mean that every application behind the tunnel is protected against unauthorized access.

### 8.1 Protect Administrative Applications

For management interfaces such as Radarr, Sonarr, Prowlarr, and qBittorrent, consider one or more of the following:
- Keep the service accessible only on your private LAN or VPN.
- Use Cloudflare Access to require authentication before reaching the application.
- Use strong, unique application passwords.
- Keep applications and container images updated.
- Avoid exposing unnecessary ports directly on the router.
- Restrict access to trusted users and devices.

Cloudflare Access policies can be configured in the Cloudflare Zero Trust dashboard. Configure and test the policy before relying on it to protect an application.

### 8.2 Jellyfin Remote Access

For Jellyfin, decide whether you want public access or access limited to specific users.

If you publish Jellyfin directly through a tunnel:
- Use a strong Jellyfin password.
- Keep Jellyfin updated.
- Review account permissions.
- Consider whether Cloudflare Access is compatible with the Jellyfin clients you plan to use.
- Test playback from outside your home network.

Some native media clients may not work with an interactive Cloudflare Access login flow. Choose the protection method based on the clients you intend to use.

### 8.3 Understand the Traffic Path

With Cloudflare Tunnel, your server establishes an outbound connection to Cloudflare. Incoming requests to the configured hostname are routed through that connection to the internal service.

This means you generally do not need to forward the application's port on your router for the tunnel to function.

However, any ports that you have already published on the host remain published unless you remove or restrict those mappings.

> [!TIP] Review Existing Port Mappings
> Check your Compose configuration and host firewall to make sure you are not unintentionally exposing services directly when you intend to access them only through Cloudflare Tunnel.

---
## 9. Troubleshooting

### 9.1 Cloudflared Cannot Connect

Inspect the logs:

```bash
sudo docker compose -f /srv/docker/stacks/media-stack/docker-compose.yml logs --tail=200 cloudflared
```

Check:
- The tunnel token is correct and has not expired or been revoked.
- The `.env` file is loaded by Compose.
- The container has outbound internet connectivity.
- DNS resolution works.
- The Cloudflare service is reachable.
- The Compose service uses the correct command and environment variables.

If you changed the token or Compose configuration, recreate the service when necessary:

```bash
cd /srv/docker/stacks/media-stack
sudo docker compose up -d cloudflared
```

### 9.2 Tunnel Is Connected, but the Application Does Not Load

Check the route's internal service URL and port.

For example, if Jellyfin is configured as:

`http://jellyfin:8096`

verify that:
- The Jellyfin container is running.
- Cloudflared and Jellyfin share a network or have another valid route between them.
- The hostname `jellyfin` resolves from Cloudflared's network.
- Jellyfin is listening on port `8096`.
- The route uses the correct protocol, usually HTTP for the internal connection when configured that way.

A tunnel can be connected while an individual application route is misconfigured.

### 9.3 Service Behind Gluetun Is Unreachable

Check whether the application uses:

```bash
network_mode: "service:gluetun"
```

If so, confirm that Cloudflared can reach Gluetun over a suitable network and that the application's port is reachable through the shared network namespace.

Remember that a service sharing Gluetun's namespace does not have the same independent network identity as a normal container on `media_net`.

### 9.4 Domain Is Not Resolving

Check:
- Cloudflare has activated the domain.
- The registrar is using the correct Cloudflare nameservers.
- The tunnel route has been saved.
- The hostname and domain are spelled correctly.
- DNS changes have had time to propagate.

If the domain previously hosted other services, verify that their DNS records remain intact.

### 9.5 Application Loads but Authentication Fails

Cloudflare Tunnel does not replace the application's own authentication.

Check the application's login configuration first. If you use Cloudflare Access, confirm that its policy is compatible with the browser or client being used and that the expected users are allowed through.

---

> [!TIP] Setup Complete
> Cloudflare Tunnel can now provide domain-based access to the applications you deliberately publish. Keep the route list small, protect administrative interfaces, and verify the network path for each service before making it available externally.

---

> [!TIP] ➜ Up Next
> You’ve completed  [00 Hosting](#00-hosting)
> **Previous:** [09 Jellyseerr](../00%20Core/09%20Jellyseerr.md)  ------ **Next:** [01 Backup](01%20Backup.md)
