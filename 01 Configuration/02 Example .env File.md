# Overview
This file contains the environment variables used to configure the server's Docker Compose stack. It centralizes settings for user permissions, timezone, directory paths, Docker networking, VPN connectivity through Gluetun, application-specific options, and Cloudflare Tunnel authentication.

Before deploying the stack, copy the example configuration to a `.env` file, replace the placeholders with your own values, and adjust the directory paths and settings to match your system. Ensure that the relevant directories exist and that your user has the necessary read/write permissions.

```properties
# ============================================================
# Media Server - Environment Configuration
# ============================================================
#
# Instructions:
# 1. Copy this file to ".env".
# 2. Replace all <PLACEHOLDER> values with your own values.
# 3. Review the paths and settings to match your system.
# ============================================================


# ------------------------------------------------------------
# User / Permissions
# ------------------------------------------------------------

# User and group IDs used by containers to manage file ownership.
# On Linux, run "id" to find your UID and GID.
PUID=1000
PGID=1000


# ------------------------------------------------------------
# Timezone
# ------------------------------------------------------------

# Timezone used by containers.
# Examples: Asia/Kolkata, Europe/London, America/New_York
TZ=Asia/Kolkata


# ------------------------------------------------------------
# Docker Network
# ------------------------------------------------------------

# Name of the Docker network shared by media server services.
# Ensure the network exists or is created by your Compose setup.
MEDIA_NETWORK=media_net


# ------------------------------------------------------------
# Root Paths
# ------------------------------------------------------------

# Root directory used to store Docker-related files.
DOCKER_ROOT=/srv/docker

# Directory that will contain application configuration and settings.
CONFIG_ROOT=/srv/docker/configs

# Directory where downloaded files are going to be stored.
DOWNLOADS_ROOT=/mnt/media-hdd/downloads

# Root directory that will contain your media library.
MEDIA_ROOT=/mnt/media-hdd/media


# ------------------------------------------------------------
# Media Folders
# ------------------------------------------------------------

# Directory that will contain movies.
MOVIES_DIR=/mnt/media-hdd/media/movies

# Directory that will contain TV shows.
TV_DIR=/mnt/media-hdd/media/tv

# Update these paths to match your actual folder structure.
# Ensure the directories exist and are accessible to the
# relevant Docker containers. And {USER} has permission to 
# perform read/write.


# ------------------------------------------------------------
# VPN Configuration (Gluetun - WireGuard)
# ------------------------------------------------------------

# VPN provider supported by Gluetun.
# Example: protonvpn
VPN_SERVICE_PROVIDER=protonvpn

# VPN protocol/type.
VPN_TYPE=wireguard

# WireGuard MTU (maximum transmission unit).
# Adjust only if required by your VPN/network configuration.
WIREGUARD_MTU=1420

# VPN server country or countries, as supported by Gluetun.
# Examples depend on the provider and Gluetun's configuration.
# Consult Gluetun documentation for valid values.
SERVER_COUNTRIES=<YOUR_VPN_SERVER_COUNTRY>

# WireGuard private key supplied by your VPN provider.
# Treat this as a secret. Never publish or commit the real key.
WIREGUARD_PRIVATE_KEY=<YOUR_WIREGUARD_PRIVATE_KEY>

# WireGuard tunnel IP address(es), as supplied by your VPN provider.
WIREGUARD_ADDRESSES=10.2.0.2/32

# Enable or disable VPN port forwarding, if supported by your
# provider and required by your setup.
# Use the value expected by your Gluetun configuration.
VPN_PORT_FORWARDING=<on_OR_off>


# ------------------------------------------------------------
# VPN DNS / Firewall Options
# ------------------------------------------------------------

# Enable DNS-over-TLS when supported by your Gluetun setup.
DOT=off

# Control whether Gluetun preserves the configured nameserver.
DNS_KEEP_NAMESERVER=off

# Enable Gluetun's firewall.
FIREWALL=on

# Enable blocking of malicious destinations.
BLOCK_MALICIOUS=off

# Enable ad blocking.
BLOCK_ADS=off

# Verify supported values and behavior against your Gluetun
# version and VPN provider documentation.


# ------------------------------------------------------------
# Jellyfin
# ------------------------------------------------------------

# File creation permission mask for Jellyfin.
# 002 typically allows group write access, subject to the
# process's permissions and the directory's existing settings.
JELLYFIN_UMASK=002


# ------------------------------------------------------------
# qBittorrent
# ------------------------------------------------------------

# Port used to access the qBittorrent Web UI.
# Ensure it matches the port mapping in your Docker Compose file.
QBIT_WEBUI_PORT=8080

# File creation permission mask for qBittorrent.
QBIT_UMASK=022


# ------------------------------------------------------------
# Cloudflare Tunnel (Optional, only for Hosting)
# ------------------------------------------------------------

# Token used to authenticate your Cloudflare Tunnel.
# Obtain it from the Cloudflare Zero Trust dashboard when
# creating or configuring your tunnel.
# Keep this token secret and never commit the real value.
CLOUDFLARE_TUNNEL_TOKEN=<YOUR_CLOUDFLARE_TUNNEL_TOKEN>

# That's It!! :D
```
