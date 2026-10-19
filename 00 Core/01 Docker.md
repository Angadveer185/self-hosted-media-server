# Overview
This document covers the installation and configuration of Docker, the preparation of the server's directory structure, and the deployment of the media stack using Docker Compose.

It is written as a step-by-step guide for anyone setting up a similar media server from scratch.

> [!NOTE] Overview
> Docker runs applications inside isolated environments called **containers**. Instead of installing every application directly onto the host operating system, we can run services independently while managing their configurations, networking, and persistent data.
> This guide uses Ubuntu as the host operating system.

## Table of Contents
- [1. Understanding the Components](#1-understanding-the-components)
- [2. Installing Docker Engine](#2-installing-docker-engine)
    - [2.1. Prepare the System](#21-prepare-the-system)
    - [2.2. Add the Official Docker Repository](#22-add-the-official-docker-repository)
    - [2.3. Install Docker Engine](#23-install-docker-engine)
    - [2.4. Enable and Start Docker](#24-enable-and-start-docker)
    - [2.5. Configure User Permissions](#25-configure-user-permissions)
    - [2.6. Verify the Installation](#26-verify-the-installation)
- [3. Optional: Install Docker Desktop](#3-optional-install-docker-desktop)
- [4. Install Portainer](#4-install-portainer)
- [5. Prepare the Media Server Environment](#5-prepare-the-media-server-environment)
    - [5.1. Create the Directory Structure](#51-create-the-directory-structure)
    - [5.2. Configure Permissions](#52-configure-permissions)
    - [5.3. Create the Docker Network](#53-create-the-docker-network)
    - [5.4. Verify Docker Compose](#54-verify-docker-compose)
- [6. Optional: Improve the CLI Experience](#6-optional-improve-the-cli-experience)
- [7. Configure Docker Desktop](#7-configure-docker-desktop)
- [8. Configure Portainer](#8-configure-portainer)
- [9. Deploy the Media Stack](#9-deploy-the-media-stack)
    - [9.1. Prepare Environment Variables and Permissions](#91-prepare-environment-variables-and-permissions)
    - [9.2. Create the Compose File](#92-create-the-compose-file)
    - [9.3. Deploy Using the CLI](#93-deploy-using-the-cli)
    - [9.4. Deploy Using Portainer](#94-deploy-using-portainer)
    - [9.5. Deploy Using Docker Desktop](#95-deploy-using-docker-desktop)
- [10. Verify the Deployment](#10-verify-the-deployment)
- [11. Troubleshooting and Maintenance](#11-troubleshooting-and-maintenance)

---
## 1. Understanding the Components

Before installing anything, it helps to understand the tools used throughout this setup.

|Component|Purpose|
|---|---|
|**Docker Engine**|Runs and manages containers on the server.|
|**Docker CLI**|Allows us to control Docker through terminal commands.|
|**Docker Compose**|Defines and manages multiple related containers using a YAML file.|
|**Docker Desktop**|Provides a graphical interface for managing Docker.|
|**Portainer CE**|Provides a browser-based interface for managing Docker environments, containers, networks, volumes, and stacks.|

### Which interfaces should I use?

Docker Engine and Docker Compose are the foundation of this setup. Docker Desktop and Portainer are optional management interfaces.
- **CLI:** Best for scripting, automation, troubleshooting, and following the commands in this documentation.
- **Docker Desktop:** Useful if you prefer a desktop application with a graphical interface.
- **Portainer:** Useful for managing the server remotely through a web browser.

> [!TIP] Recommended approach
> Install Docker Engine and Docker Compose first. You can then use the CLI, Portainer, or Docker Desktop to manage your containers.
> You do not need both Docker Desktop and Portainer. For a headless media server, Docker Engine with Portainer is often sufficient.

## 2. Installing Docker Engine

Docker Engine is the core component that runs containers. The following instructions install Docker Engine and its associated tools using Docker's official Ubuntu package repository.

### 2.1. Prepare the System

Update the package lists and install the utilities required to download packages and verify repository signatures.

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y ca-certificates curl gnupg lsb-release
```

If an older or conflicting Docker installation exists, remove the legacy packages before proceeding.

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc || true
```

> [!WARNING] Existing Docker installations
> Removing packages is not the same as deleting Docker's data directory, but changing or replacing an existing installation can still affect your environment. If this is an existing server, back up important container configurations and data first.
> Skip the removal command if you already have a working installation that you intend to keep.

### 2.2. Add the Official Docker Repository

Docker's official repository provides current Docker Engine packages and their associated components.

**Step 1: Create the keyring directory.**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

**Step 2: Download and install Docker's GPG signing key.**

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**Step 3: Add the official Docker repository.**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Step 4: Refresh the package lists.**

```bash
sudo apt update
```

> [!NOTE] Operating system compatibility
> These commands target Ubuntu. If you are using Debian or another Linux distribution, follow the corresponding instructions in the [official Docker Engine installation documentation](https://docs.docker.com/engine/install/).

### 2.3. Install Docker Engine

Install Docker Engine, the CLI, the container runtime, Buildx, and the Docker Compose plugin.

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

The packages provide the following functionality:

|Package|Purpose|
|---|---|
|`docker-ce`|Docker Engine.|
|`docker-ce-cli`|The `docker` command-line interface.|
|`containerd.io`|The container runtime used by Docker.|
|`docker-buildx-plugin`|Extended image-building functionality.|
|`docker-compose-plugin`|Docker Compose V2, invoked using `docker compose`.|

### 2.4. Enable and Start Docker

Enable Docker to start automatically at boot and start it immediately.

```bash
sudo systemctl enable --now docker
```

Check the service status:

```bash
sudo systemctl status docker
```

Look for `active (running)` in the output.

If Docker is not running, start it manually:

```bash
sudo systemctl start docker
```

### 2.5. Configure User Permissions

By default, administrative privileges are generally required to communicate with the Docker daemon through its Unix socket.

To run Docker commands without repeatedly using `sudo`, add your user to the `docker` group.

```bash
sudo usermod -aG docker "$USER"
```

Apply the group membership to your current terminal session:

```bash
newgrp docker
```

Alternatively, log out and log back in.

> [!WARNING] Security consideration
> Membership in the `docker` group effectively grants root-level privileges on the host. Only add trusted users to this group.

### 2.6. Verify the Installation

Run the following commands to confirm that Docker is installed and functioning correctly.

**Check the client and server versions:**

```bash
docker version
```

**Inspect the Docker environment:**

```bash
docker info
```

**Run a test container:**

```bash
docker run --rm hello-world
```

The test image should download if necessary, run, display a success message, and exit. The `--rm` flag automatically removes the stopped test container.

> [!TIP] Installation complete
> Docker Engine is ready. Continue to the next section to install an optional graphical interface or proceed directly to preparing the media server environment.

---
## 3. Optional: Install Docker Desktop

Docker Desktop provides a graphical interface for managing containers, images, volumes, and other Docker resources.

It is optional and is primarily useful when the server has a desktop environment.

### 3.1. Download Docker Desktop
1. Open the [official Docker Desktop installation guide for Ubuntu](https://docs.docker.com/desktop/setup/install/linux/ubuntu/).
2. Download the latest appropriate Debian package (`.deb`).
3. Save the downloaded file, typically in your `Downloads` directory.

### 3.2. Install Docker Desktop

Open a terminal in the directory containing the downloaded package.

For example:

```bash
cd ~/Downloads
```

Install the package, replacing the filename if necessary.

```bash
sudo apt update
sudo apt install -y ./docker-desktop-amd64.deb
```

The downloaded filename may differ depending on the release. Use the exact filename shown in your directory.

If APT displays an unsandboxed-download warning because the package is inaccessible to the `_apt` user, check that the package was installed successfully before proceeding. Do not assume that every installation warning can be ignored.

### 3.3. Start Docker Desktop

Launch Docker Desktop from the application menu, or use its user-level systemd service:

```bash
systemctl --user enable --now docker-desktop
```

If necessary, check its status:

```bash
systemctl --user status docker-desktop
```

Complete the first-run prompts in the application.

Docker Desktop on Linux uses a separate Docker context and VM-based environment by default. Consequently, its containers and resources may be separate from those managed by the host's system-wide Docker Engine.

> [!IMPORTANT] Choose your Docker environment
> If you installed Docker Engine and Docker Desktop, make sure you know which Docker context you are using. Do not assume that containers created by one environment will automatically appear in the other.
> For this media server, the simplest approach is to use the system-wide Docker Engine consistently, whether managing it through the CLI or a compatible web interface.

---
## 4. Optional: Install Portainer

Portainer Community Edition (CE) provides a browser-based interface for managing Docker.

It allows you to inspect containers, view logs, manage volumes and networks, and deploy Docker Compose stacks without relying entirely on the terminal.

### 4.1. Create a Persistent Volume

Create a Docker volume to store Portainer's configuration and application data.

```bash
docker volume create portainer_data
```

Docker volumes persist independently of a container's lifecycle. This means removing and recreating the Portainer container does not automatically erase the data stored in this volume.

### 4.2. Deploy Portainer CE

Run the following command:

```bash
docker run -d \
  --name portainer \
  --restart=always \
  -p 9443:9443 \
  -p 8001:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

> [!NOTE] Image version
> This command uses the `lts` image tag. For reproducible deployments, consider pinning a specific Portainer release after checking the [official Portainer installation instructions](https://docs.portainer.io/start/install-ce/server/docker/linux).

### 4.3. Understand the Command

|Option|Explanation|
|---|---|
|`docker run -d`|Creates and starts the container in the background.|
|`--name portainer`|Assigns the container the name `portainer`.|
|`--restart=always`|Restarts the container automatically when Docker restarts or the container stops.|
|`-p 9443:9443`|Exposes Portainer's HTTPS web interface on host port `9443`.|
|`-p 8001:8000`|Exposes the Portainer Edge Agent tunnel port on host port `8001`. It is not required for a basic local Docker environment.|
|`-v /var/run/docker.sock:/var/run/docker.sock`|Allows Portainer to communicate with the host Docker Engine.|
|`-v portainer_data:/data`|Stores Portainer's persistent application data.|
|`portainer/portainer-ce:lts`|Specifies the Portainer Community Edition image.|

The port mapping format is:

```bash
host_port:container_port
```

For example, `9443:9443` makes the container's HTTPS service accessible through port `9443` on the host.

### 4.4. Access Portainer

Open the following URL in a browser on the server:

```bash
https://localhost:9443
```

If accessing Portainer from another computer, replace `localhost` with the server's IP address:

```bash
https://SERVER_IP:9443
```

For example, if the server's LAN address is `192.168.1.100`, use:

```bash
https://192.168.1.100:9443
```

The browser may display a certificate warning because Portainer's default HTTPS certificate is not trusted by your browser. Verify that you are connecting to your own server before proceeding.

Create an administrator account and configure the local Docker environment.

> [!WARNING] Network security
> Keep Portainer's management interface restricted to trusted devices or a secure VPN. Do not expose it directly to the public internet without appropriate security controls.

To check whether Portainer is running:

```bash
docker ps
```

---
## 5. Prepare the Media Server Environment

Before deploying the media applications, create a consistent directory structure for configuration files, downloads, and the media library.

The examples below uses the directory structure described in [00 Folder Structure](../01%20Configuration/00%20Folder%20Structure.md).

### 5.1. Create the Directory Structure

> [!WARNING] Directory Structure Difference
> The commands below create `/downloads` and `/media` **inside** `/srv/docker`. This differs from the structure used in my setup.
> In my configuration, the media directories are stored on an external hard drive mounted at `/mnt/media-hdd`. As a result, `/downloads` and `/media` are located outside `/srv/docker`.
> **Both approaches are valid.** Choose the structure that matches your storage setup, and ensure your Docker Compose volume mappings and environment variables point to the correct host paths.
> **If you follow the commands below:**
> - Your downloads will be stored in `/srv/docker/downloads`.
> - Your media library will be stored in `/srv/docker/media`.
> - If you use the backup script from [01 Backup](../04%20Optional/01%20Backup.md), update its exclusions to prevent the potentially large `downloads` and `media` directories from being included in your configuration backups.

Run the following commands to create the example directory structure:

```bash
sudo mkdir -p /srv/docker/{configs,downloads,media,stacks}

sudo mkdir -p /srv/docker/configs/{prowlarr,sonarr,radarr,bazarr,qbittorrent,gluetun,portainer}

sudo mkdir -p /srv/docker/downloads/{movies,tv,anime}

sudo mkdir -p /srv/docker/media/{movies,tv,anime}
```

The resulting directory structure will look like this:

```
/srv/docker/
├── configs/
│   ├── prowlarr/
│   ├── sonarr/
│   ├── radarr/
│   ├── bazarr/
│   ├── qbittorrent/
│   ├── gluetun/
│   └── portainer/
├── downloads/
│   ├── movies/
│   ├── tv/
│   └── anime/
├── media/
│   ├── movies/
│   ├── tv/
│   └── anime/
└── stacks/
```

> [!TIP] Before Continuing
> If you are using an external drive, replace the relevant paths with your actual mount location, such as `/mnt/media-hdd/downloads` and `/mnt/media-hdd/media`. Keep your `.env` file, Compose volume mappings, and backup script consistent with whichever structure you choose.

|Directory|Purpose|
|---|---|
|`/srv/docker/configs/`|Stores application configurations and persistent service data.|
|`/srv/docker/downloads/`|Stores files downloaded by the download client before they are imported into the media library.|
|`/srv/docker/media/`|Stores the organized media library.|
|`/srv/docker/stacks/`|Stores Docker Compose files used to deploy and manage groups of services.|


> [!TIP] Keep downloads and media organized
> Separating downloads from the final media library makes the setup easier to maintain. Keeping both under `/srv/docker` also makes it possible to configure consistent container paths and, where appropriate, support hardlinks for efficient imports.

### 5.2. Configure Permissions

Make your current user the owner of the Docker directory tree:

```bash
sudo chown -R "$USER":"$(id -gn)" /srv/docker
```

For a straightforward single-user setup, you can grant the owner and group read, write, and directory traversal permissions:

```bash
sudo chmod -R 775 /srv/docker
```

These commands have different purposes:
- `chown` changes the owner and group associated with files and directories.
- `chmod` changes the permissions that determine who can read, write, or access them.

The `775` permission grants read, write, and execute permissions to the owner and group, and read and execute permissions to others.

> [!WARNING] Permission management
> `chmod -R 775` is a broad permission change. It does not automatically make files owned by different users writable, and it is not suitable for every security model. In particular, configuration directories may contain credentials, API keys, or other sensitive data.
> Adjust permissions according to the users and groups that need access rather than applying increasingly broad permissions to fix every problem.

### 5.3. Create the Docker Network

Create a dedicated Docker bridge network for the media stack:

```bash
docker network create media_net
```

A custom network allows containers to communicate with one another using their Docker service or container names, provided they are attached to that network.

For example, services on `media_net` can generally communicate using names such as `sonarr` or `radarr`, depending on how the Compose services are defined.

If the network already exists, Docker will report that it cannot create another network with the same name. That is expected.

You can verify the network with:

```bash
docker network ls
```

> [!IMPORTANT] Network naming
> This guide consistently uses `media_net`. The name must match the network referenced in the Docker Compose file.
> If your Compose file uses a different network name, update either the file or the commands accordingly.

### 5.4. Verify Docker Compose

Docker Compose is included in the installation performed earlier.

Check its version:

```bash
docker compose version
```

If the command is unavailable, install the plugin:

```bash
sudo apt update
sudo apt install -y docker-compose-plugin
```

The command used throughout this documentation is:

```bash
docker compose
```

This is Docker Compose V2, which is different from the older standalone `docker-compose` command.

> [!TIP] Environment prepared
> Docker Engine, Docker Compose, the base directory structure, and the dedicated network are now ready for the media stack.

---
## 6. Optional: Improve the CLI Experience

These changes are optional quality-of-life improvements. They are not required for Docker to function.

### 6.1. Enable Bash Autocompletion

Install Bash's completion package:

```bash
sudo apt install -y bash-completion
```

Add the following to `~/.bashrc` if it is not already present:

```bash
if [ -f /usr/share/bash-completion/bash_completion ]; then
    source /usr/share/bash-completion/bash_completion
fi
```

Reload the configuration:

```bash
source ~/.bashrc
```

This enables shell completion support for compatible commands. Docker-specific completions may also depend on the shell configuration and installed completion scripts.

### 6.2. Add Docker Aliases

Aliases shorten frequently used commands.

Add the following lines to `~/.bashrc`:

```bash
alias dk='docker'
alias dkc='docker compose'
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

Reload the configuration:

```bash
source ~/.bashrc
```

You can now use the following shortcuts:

|Alias|Equivalent command|
|---|---|
|`dk ps`|`docker ps`|
|`dkc up -d`|`docker compose up -d`|
|`dps`|Lists running containers with their names, status, and ports.|

These aliases only affect your shell. They do not change Docker's configuration or behavior.

---
## 7. Configure Docker Desktop

Skip this section if you are managing Docker exclusively through the CLI and Portainer.

### 7.1. Complete the Initial Setup
1. Open Docker Desktop.
2. Complete any first-run prompts.
3. Confirm that the Docker environment is running.
4. Open the Containers dashboard and inspect the available containers.

### 7.2. Configure Resources

If the Docker Desktop environment is the one you intend to use, open its settings and review the available resource options.

Depending on the version, you may be able to configure:
- CPU allocation.
- Memory allocation.
- Disk usage or disk image settings.

The available controls depend on the platform and Docker Desktop release.

> [!IMPORTANT] Docker Engine versus Docker Desktop
> Docker Desktop on Linux normally operates through its own VM and Docker context. It does not automatically manage containers running under the host's system-wide Docker Engine.
> For this media server, do not switch between environments unintentionally. Keep the containers, volumes, networks, and Compose deployments in the same Docker environment.

---
## 8. Configure Portainer

If you installed Portainer, complete its initial setup before deploying the media stack.

### 8.1. Create the Administrator Account

Open:

```bash
https://localhost:9443
```

Create an administrator username and a strong password.

Use your own credentials. Do not reuse example usernames or passwords from documentation.

### 8.2. Connect to the Local Docker Environment

During the initial setup, select the local Docker environment if it is available.

If the environment has not been configured yet:
1. Open **Environments** in Portainer.
2. Add or select the local Docker environment.
3. Choose the appropriate Docker connection method.
4. Confirm that the environment is reachable.

For the local Docker installation used in this guide, Portainer communicates with Docker through the mounted Docker socket.

> [!NOTE] Existing Portainer installation
> If Portainer is already running and connected to the correct Docker Engine, you do not need to repeat this setup.

---
## 9. Deploy the Media Stack

This section covers the deployment of the media stack using Docker Compose.

The Compose file defines the services, container settings, network configuration, environment variables, ports, and persistent storage.

Only **one deployment method is needed**. Choose the CLI method if you want a repeatable terminal-based workflow, or use Portainer if you prefer a browser-based interface.

### 9.1. Prepare Environment Variables and Permissions

Before creating the Compose file, identify the user and group IDs that the media applications should use.

Run:

```bash
id -u "$USER"
id -g "$USER"
```

The results are commonly `1000` and `1000`, respectively, on a typical single-user Ubuntu installation, but you should use your actual values.

These values are commonly passed to containers through the `PUID` and `PGID` environment variables when the relevant image supports them.

Set the timezone using the appropriate IANA timezone identifier. For example, India uses:

```bash
Asia/Kolkata
```

The exact environment variables depend on the images selected for the stack. Not every image uses `PUID`, `PGID`, or `TZ`.

> [!WARNING] Environment variables are image-specific
> Do not assume that every container recognizes the same environment variables. Follow the documentation for each application image.

### 9.2. Create the Compose File

Create the directory that will hold the media stack's Compose file:

```bash
mkdir -p /srv/docker/stacks/media-stack
```

Ensure your user can manage the directory:

```bash
sudo chown -R "$USER":"$(id -gn)" /srv/docker/stacks
```

Create or open the Compose file:

```bash
nano /srv/docker/stacks/media-stack/docker-compose.yml
```

Paste your media stack's Docker Compose configuration into the file. The compose file used for this server is: [01 Docker Compose](../01%20Configuration/01%20Docker%20Compose.md).

Before saving, verify that the configuration uses the correct values for:
- `PUID` and `PGID`, where supported.
- `TZ`, where supported.
- The `/srv/docker/configs/` paths for application configurations.
- The `/srv/docker/downloads/` paths for downloads.
- The `/srv/docker/media/` paths for the final media library.
- The `media_net` network, if the stack is designed to use it.
- Any required port mappings and environment variables.

In Nano, save using `Ctrl + O`, press `Enter`, and exit using `Ctrl + X`.

> [!IMPORTANT] The Compose file is the source of truth
> Ensure that your configuration file matches the actual directory structure, image documentation, and networking requirements. The deployment commands below assume that the file exists at the path specified above.

### 9.3. Deploy Using the CLI

This is the recommended workflow for a repeatable deployment.

#### Step 1: Confirm the network exists

```bash
docker network inspect media_net >/dev/null 2>&1 \
  || docker network create media_net
```

This creates the network only if it does not already exist.

#### Step 2: Validate the Compose configuration

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  config
```

This checks the Compose configuration and renders the resolved configuration. Review any reported errors before proceeding.

#### Step 3: Pull the container images

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  pull
```

Docker downloads the images required by the Compose file.

#### Step 4: Start the stack

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  up -d
```

The `-d` option runs the containers in the background.

Compose creates or updates the containers and networks declared in the configuration as needed.

#### Step 5: Inspect the containers

```bash
docker ps
```

To include stopped containers:

```bash
docker ps -a
```

To view the logs for a service:

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  logs --tail=100
```

To follow logs continuously:

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  logs -f
```

Press `Ctrl + C` to stop following the logs. This does not stop the containers.

### 9.4. Deploy Using Portainer

Portainer can deploy the same Compose configuration through its Stacks interface.

#### Step 1: Open Portainer

Navigate to:

```
https://localhost:9443
```

Log in and select the Docker environment that hosts the media stack.

#### Step 2: Create a stack
1. Open **Stacks**.
2. Select **Add stack**.
3. Enter a name, such as `media-stack`.
4. Select the Web editor if you are pasting the Compose configuration manually.
5. Paste the entire contents of `docker-compose.yml`.

#### Step 3: Review the configuration

Before deploying, check that:
- All environment variables are correct.
- The bind-mount paths match the directories created earlier.
- The network configuration matches the intended Docker network.
- Required ports are configured correctly.
- The Compose configuration is complete.

If your configuration uses an externally created network, ensure that the network exists and is declared appropriately in the Compose file.

#### Step 4: Deploy the stack

Click **Deploy the stack**.

Portainer will process the Compose configuration and create or update the required resources.

After deployment, open the stack to inspect its services, container status, and logs.

> [!NOTE] Managing the stack after deployment
> If you deploy the stack through Portainer, use Portainer as your primary interface for subsequent stack edits where practical. You can also manage it through the CLI, but avoid making conflicting configuration changes through different interfaces.

### 9.5. Deploy Using Docker Desktop

Skip this section if you are using the host's Docker Engine and Portainer.

If your version of Docker Desktop provides a compatible Compose or stack workflow, use the following general process:
1. Open Docker Desktop.
2. Confirm that the intended Docker context is active.
3. Open the appropriate Compose or container-management interface.
4. Load the Compose configuration, if supported by that version.
5. Review the configuration and deploy the services.
6. Inspect the resulting containers and logs.

The exact interface varies by Docker Desktop version. Docker Desktop does not universally provide the same Stacks workflow as Portainer.

If the graphical interface does not support the desired deployment workflow, use the CLI commands in [9.3. Deploy Using the CLI](#93-deploy-using-the-cli).

> [!WARNING] Do not deploy the same stack twice
> Deploying the same Compose configuration through multiple interfaces can cause conflicts or unexpected changes, especially if the interfaces are connected to different Docker environments. Choose one primary deployment workflow.

---
## 10. Verify the Deployment

Once the stack has been deployed, perform the following checks.

### 10.1. Confirm the Containers Are Running

```bash
docker ps
```

Review the container names, status, and exposed ports.

To include containers that have exited:

```bash
docker ps -a
```

A container that repeatedly exits or restarts requires further investigation.

### 10.2. Check the Compose Stack

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  ps
```

This displays the status of the services declared in the Compose file.

### 10.3. Inspect Logs

If a service fails to start, inspect its logs:

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  logs --tail=100 SERVICE_NAME
```

Replace `SERVICE_NAME` with the relevant Compose service name.

Common issues include:
- Incorrect environment variables.
- Invalid bind-mount paths.
- Permission errors.
- Port conflicts.
- Missing networks.
- Incorrect VPN or application configuration.

### 10.4. Verify the Filesystem

Confirm that the directory structure exists:

```bash
ls -la /srv/docker
ls -la /srv/docker/configs
ls -la /srv/docker/downloads
ls -la /srv/docker/media
ls -la /srv/docker/stacks
```

### 10.5. Access the Applications

Use the ports defined in your actual Compose configuration to access each application's web interface.

Do not assume that every application uses a particular port simply because it appears in an example configuration. Confirm the port mappings in the Compose file or run:

```bash
docker ps
```

> [!TIP] Deployment complete
> Docker is installed, the media server's directory structure is prepared, and the Compose stack has been deployed. The next step is to configure each application and verify that the services can communicate correctly.

---
## 11. Troubleshooting and Maintenance

### Docker commands require `sudo`

If you have added your user to the `docker` group but commands still require `sudo`, log out and log back in, or start a new shell session.

Check your group membership:

```bash
groups
```

Remember that access to the Docker socket grants substantial privileges.

### Docker is not running

Check the service:

```bash
sudo systemctl status docker
```

Start it if necessary:

```bash
sudo systemctl start docker
```

Enable automatic startup:

```bash
sudo systemctl enable docker
```

### A container is not starting

Inspect the container's status and logs:

```bash
docker ps -a
docker logs --tail=100 CONTAINER_NAME
```

Replace `CONTAINER_NAME` with the actual container name.

For Compose-managed services, prefer the Compose logs command shown in [10.3. Inspect Logs](#103-inspect-logs).

### A port is already in use

Check the port mappings:

```bash
docker ps
```

If another service already occupies a host port, change the host-side port mapping in the Compose file or stop the conflicting service if appropriate.

For example:

```bash
ports:
  - "8090:8080"
```

This maps host port `8090` to container port `8080`. The container application must actually listen on the specified container port.

### A container cannot access a directory

Inspect ownership and permissions:

```bash
ls -ld /srv/docker
ls -ld /srv/docker/configs
ls -ld /srv/docker/downloads
ls -ld /srv/docker/media
```

Confirm that the container's effective user has the necessary access and that the bind-mount paths match the Compose configuration.

Do not immediately solve permission issues by granting unrestricted access to every directory.

### Update the media stack

To pull updated images and recreate containers when necessary:

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  pull

docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  up -d
```

Review application release notes and back up important configuration data before making major upgrades.

### Stop the media stack

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  down
```

This stops and removes the stack's containers and its Compose-managed networks, while normally preserving named volumes and host directories.

External networks and other resources not owned by the stack are not automatically removed.

### Restart the media stack

```bash
docker compose \
  -f /srv/docker/stacks/media-stack/docker-compose.yml \
  restart
```

This restarts the services without rebuilding the configuration.

> [!CAUTION] Protect persistent data
> Avoid deleting Docker volumes or configuration directories unless you understand their contents. Removing a container does not necessarily delete its data, but deleting the volume or host directory that stores the data can permanently erase application settings and other important files.

---

> [!TIP] ➜ Up Next
> You’ve completed  [01 Docker](#01-docker)
> **Previous:** [00 Overview](00%20Overview.md)  ------ **Next:** [02 Gluetun](02%20Gluetun.md)
