# Overview
This file defines the services that make up the media server, their persistent configuration directories, network settings, exposed ports, restart policies, and health checks.

| Service          | Purpose                                                    | Network         |
| ---------------- | ---------------------------------------------------------- | --------------- |
| **Gluetun**      | VPN gateway for services routed through the VPN            | `media_net`     |
| **Prowlarr**     | Indexer management                                         | Gluetun network |
| **qBittorrent**  | Torrent downloads                                          | Gluetun network |
| **FlareSolverr** | Helps supported indexers handle certain browser challenges | Gluetun network |
| **Sonarr**       | TV show management                                         | `media_net`     |
| **Radarr**       | Movie management                                           | `media_net`     |
| **Bazarr**       | Subtitle management                                        | `media_net`     |
| **Jellyfin**     | Media streaming server with NVIDIA GPU access configured   | `media_net`     |
| **Jellyseerr**   | Media request management                                   | Gluetun network |
| **Cloudflared**  | Cloudflare Tunnel connection                               | `media_net`     |

## Configuration notes
- Create a `.env` file beside `docker-compose.yml` and define the variables referenced below.
- The external Docker network must already exist.
- The host directories referenced by the volume mappings should exist and be accessible to the relevant containers.
- Keep secrets such as `WIREGUARD_PRIVATE_KEY` and `CLOUDFLARE_TUNNEL_TOKEN` out of version control.
- `latest` image tags can change over time. Pin versions if reproducible deployments are important.
- Services using `network_mode: "service:gluetun"` share Gluetun's network namespace. Their web interfaces are exposed through the ports published by Gluetun, not by publishing ports on each dependent service.

---

# Full File

```yaml
# Server's main compose file


networks:
  media_net:
    external: true
    name: ${MEDIA_NETWORK}

services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      - TZ=${TZ}
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=${VPN_TYPE}
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - VPN_PORT_FORWARDING=${VPN_PORT_FORWARDING}
      - VPN_PORT_FORWARDING_PROVIDER=${VPN_SERVICE_PROVIDER}
      - WIREGUARD_MTU=${WIREGUARD_MTU}
      - FIREWALL=${FIREWALL}
      - BLOCK_MALICIOUS=${BLOCK_MALICIOUS}
      - BLOCK_ADS=${BLOCK_ADS}
    volumes:
      - ${CONFIG_ROOT}/gluetun:/gluetun
    ports:
      - "8888:8888"
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
      - "9696:9696"
      - "8191:8191"
      - "5056:5055"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "https://ipinfo.io/ip"]
      interval: 30s
      timeout: 10s
      retries: 3

  prowlarr:
    image: ghcr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/prowlarr:/config
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9696"]
      interval: 30s
      timeout: 10s
      retries: 5

  qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=${QBIT_WEBUI_PORT}
      - UMASK_SET=${QBIT_UMASK}
    volumes:
      - ${CONFIG_ROOT}/qbittorrent:/config
      - ${DOWNLOADS_ROOT}:/downloads
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:${QBIT_WEBUI_PORT}"]
      interval: 30s
      timeout: 10s
      retries: 5

  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8191"]
      interval: 30s
      timeout: 10s
      retries: 5

  sonarr:
    image: ghcr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/sonarr:/config
      - ${DOWNLOADS_ROOT}:/downloads
      - ${TV_DIR}:/tv
    ports:
      - "8989:8989"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8989"]
      interval: 30s
      timeout: 10s
      retries: 5

  radarr:
    image: ghcr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/radarr:/config
      - ${DOWNLOADS_ROOT}:/downloads
      - ${MOVIES_DIR}:/movies
    ports:
      - "7878:7878"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:7878"]
      interval: 30s
      timeout: 10s
      retries: 5

  bazarr:
    image: ghcr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/bazarr:/config
      - ${MEDIA_ROOT}:/mnt/media
      - ${DOWNLOADS_ROOT}:/mnt/downloads
    ports:
      - "6767:6767"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:6767"]
      interval: 30s
      timeout: 10s
      retries: 5

  jellyfin:
    image: ghcr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    runtime: nvidia
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=${JELLYFIN_UMASK}
      - DOTNET_SYSTEM_NET_DISABLEIPV6=true
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
    volumes:
      - ${CONFIG_ROOT}/jellyfin:/config
      - ${MEDIA_ROOT}:/media
    ports:
      - "8096:8096"
      - "8920:8920"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "curl", "-f", "-s", "http://localhost:8096"]
      interval: 30s
      timeout: 10s
      retries: 5

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/jellyseerr:/app/config
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:5055"]
      interval: 30s
      timeout: 10s
      retries: 5

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel --metrics 0.0.0.0:20241 run
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - media_net
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "--metrics", "127.0.0.1:20241", "ready"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

# End of file :)
```

# Gluetun

```yaml
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      - TZ=${TZ}

      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=${VPN_TYPE}

      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}

      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - VPN_PORT_FORWARDING=${VPN_PORT_FORWARDING}
      - VPN_PORT_FORWARDING_PROVIDER=${VPN_SERVICE_PROVIDER}

      - WIREGUARD_MTU=${WIREGUARD_MTU}
      - FIREWALL=${FIREWALL}

      - BLOCK_MALICIOUS=${BLOCK_MALICIOUS}
      - BLOCK_ADS=${BLOCK_ADS}
    volumes:
      - ${CONFIG_ROOT}/gluetun:/gluetun
    ports:
      - "8888:8888"

      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"

      - "9696:9696"
      - "8191:8191"

      - "5056:5055"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "https://ipinfo.io/ip"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

# Prowlarr

```yaml
  prowlarr:
    image: ghcr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/prowlarr:/config
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9696"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# QBitTorrent

```yaml
  qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=${QBIT_WEBUI_PORT}
      - UMASK_SET=${QBIT_UMASK}
    volumes:
      - ${CONFIG_ROOT}/qbittorrent:/config
      - ${DOWNLOADS_ROOT}:/downloads
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:${QBIT_WEBUI_PORT}"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Flaresolverr

```yaml
  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8191"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Sonarr

```yaml
  sonarr:
    image: ghcr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/sonarr:/config
      - ${DOWNLOADS_ROOT}:/downloads
      - ${TV_DIR}:/tv
    ports:
      - "8989:8989"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8989"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Radarr

```yaml
  radarr:
    image: ghcr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/radarr:/config
      - ${DOWNLOADS_ROOT}:/downloads
      - ${MOVIES_DIR}:/movies
    ports:
      - "7878:7878"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:7878"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Bazarr

```yaml
  bazarr:
    image: ghcr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/bazarr:/config
      - ${MEDIA_ROOT}:/mnt/media
      - ${DOWNLOADS_ROOT}:/mnt/downloads
    ports:
      - "6767:6767"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:6767"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Jellyfin

```yaml
  jellyfin:
    image: ghcr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    runtime: nvidia
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=${JELLYFIN_UMASK}
      - DOTNET_SYSTEM_NET_DISABLEIPV6=true
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
    volumes:
      - ${CONFIG_ROOT}/jellyfin:/config
      - ${MEDIA_ROOT}:/media
    ports:
      - "8096:8096"
      - "8920:8920"
    restart: unless-stopped
    networks:
      - ${MEDIA_NETWORK}
    healthcheck:
      test: ["CMD", "curl", "-f", "-s", "http://localhost:8096"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Jellyseerr

```yaml
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_ROOT}/jellyseerr:/app/config
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:5055"]
      interval: 30s
      timeout: 10s
      retries: 5
```

---

# Cloudflared

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel --metrics 0.0.0.0:20241 run
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - media_net
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "--metrics", "127.0.0.1:20241", "ready"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
```

---

# Things to verify before deployment
1. **Cloudflared network:** the `cloudflared` service uses the literal network name `media_net`, while the network declaration uses `${MEDIA_NETWORK}`. Ensure these resolve to the same actual Docker network, or make the service use the variable consistently.
2. **Jellyseerr published port:** Gluetun publishes host port `5056` to container port `5055`. This is intentional if Jellyseerr should be reached on host port `5056`.
3. **VPN-dependent ports:** confirm the published ports match the ports configured by qBittorrent, Prowlarr, FlareSolverr, and Jellyseerr.
4. **GPU support:** the host needs a working NVIDIA driver and Docker NVIDIA runtime configuration for Jellyfin's GPU settings to work.
5. **Health checks:** the configured commands depend on utilities being available inside each image. If a service repeatedly reports unhealthy, verify that its image includes the health-check utility and that the endpoint responds as expected.
