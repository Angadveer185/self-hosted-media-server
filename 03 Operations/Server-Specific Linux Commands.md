# Overview
A quick reference to commonly used commands for managing, monitoring, and troubleshooting my media server.

## 1. Navigate to the Media Stack

Navigate to the directory containing the Docker Compose configuration.

```bash
cd /srv/docker/stacks/media-stack
```

## 2. Restart the Media Server

Stop the running Compose stack and start it again in detached mode.

```bash
# Destroy the networks, links, and unattached resources, 
# then re-create them from scratch.
docker compose down && docker compose up -d

# Restart the processes inside the existing containers.
docker compose restart

# Replaces only the containers that need changes.
docker compose up -d --force-recreate
```

## 3. Check Container Health

Display the status of all running containers to quickly identify services that may not be running properly.

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.State}}"
```

## 4. Check Gluetun Logs

Inspect Gluetun's logs for VPN port-forwarding information.

```bash
docker logs --tail 200 gluetun | grep port
```

## 5. View the Server Directory Structure

Display the directory structure under `/srv`, up to four levels deep.

```bash
tree -L 4 /srv
```
