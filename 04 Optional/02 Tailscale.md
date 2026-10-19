# Overview

Tailscale creates a private network between your devices, allowing you to access your Ubuntu media server remotely without exposing SSH or other management ports directly to the public internet.

Once configured, you can connect to your server from your laptop, desktop, or other authorized device, even when you are away from home.

> [!NOTE] Overview
> - **Service:** Tailscale
> - **Server OS:** Ubuntu
> - **Primary use:** Private remote access and SSH
> - **Authentication:** Tailscale account
> - **Remote shell:** OpenSSH over the Tailscale network
> - **Public port forwarding:** Not required for normal Tailscale connectivity

## Table of Contents
- [1. Preparation](#1-preparation)
- [2. Install Tailscale on the Server](#2-install-tailscale-on-the-server)
- [3. Connect Your Server to Tailscale](#3-connect-your-server-to-tailscale)
- [4. Install Tailscale on Your Client Device](#4-install-tailscale-on-your-client-device)
- [5. Configure and Test SSH](#5-configure-and-test-ssh)
- [6. Optional Key Expiry Configuration](#6-optional-key-expiry-configuration)
- [7. Troubleshooting](#7-troubleshooting)
- [8. Final Checklist](#8-final-checklist)

---
## 1. Preparation

Before installing Tailscale, create an account and ensure that SSH is available on your Ubuntu server.

### 1.1 Create a Tailscale Account
1. Visit the [Tailscale website](https://tailscale.com/).
2. Create an account or sign in using a supported authentication provider, such as Google.
3. Complete the account setup.
4. Keep your account credentials secure.

You will use this account to authorize your server and the devices you want to connect from.

### 1.2 Install OpenSSH Server

Open a terminal on your Ubuntu server and run:

```bash
# Update package information
sudo apt update

# Install the OpenSSH server
sudo apt install openssh-server
```

OpenSSH allows you to connect to your Ubuntu server through a terminal. Tailscale provides the private network path used for that connection.

### 1.3 Check the SSH Service

Check whether the SSH service is running:

```bash
sudo systemctl status ssh
```

If it is not running, start it and enable it to start automatically during boot:

```bash
sudo systemctl enable --now ssh
```

Verify the status again:

```bash
sudo systemctl status ssh
```

Look for an active status.

### 1.4 Verify the SSH Listening Port

Run:

```bash
sudo ss -tlnp | grep ':22'
```

Example output:

```bash
LISTEN 0 128 0.0.0.0:22
```

The actual output may differ. You may see an IPv6 address, a specific interface address, or a different port if SSH has been configured differently.

> [!IMPORTANT] SSH Security
> Do not forward port `22` from your router to the public internet just to use Tailscale. Once Tailscale is connected, you can normally access SSH through the private Tailscale network.
> If you have already configured public SSH access, review whether it is still necessary.

---
## 2. Install Tailscale on the Server

Tailscale provides an installation script for supported Linux distributions.

### 2.1 Install the Package

Run the official installation command:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This downloads and executes the installation script. If you prefer to inspect scripts before executing them, download the script first and review its contents.

### 2.2 Verify the Installation

Check the installed version:

```bash
tailscale version
```

If the command returns version information, the Tailscale CLI is installed.

---
## 3. Connect Your Server to Tailscale

### 3.1 Authenticate the Server

Run:

```bash
sudo tailscale up
```

Tailscale may display an authentication URL in the terminal.

Example:

```bash
To authenticate, visit:

https://login.tailscale.com/a/EXAMPLE
```

Open the displayed URL, sign in to your Tailscale account, and authorize the server.

The actual URL will be unique to your authentication session.

### 3.2 Verify the Connection

After authentication, check the connection status:

```bash
tailscale status
```

To display the server's Tailscale IPv4 address:

```bash
tailscale ip -4
```

The server should now appear in your Tailscale device list.

### 3.3 Confirm the Device in the Dashboard

Open the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).

Find your server and verify that it appears online.

You should be able to identify the device by its machine name, operating system, and Tailscale IP address.

### 3.4 Rename the Server

A clear device name makes it easier to connect to the right machine later.
1. Open the Tailscale Admin Console.
2. Find your Ubuntu server.
3. Open the device's menu.
4. Select the rename or edit-name option.
5. Choose a short, recognizable name, such as `media-server`.

The exact menu labels may change with the dashboard interface.

> [!TIP] Use a Stable Name
> A name such as `media-server` is easier to remember than an automatically generated device name. If MagicDNS is enabled, you can often use the device's name instead of its Tailscale IP address.

---
## 4. Install Tailscale on Your Client Device

Install Tailscale on the device you intend to connect from, such as your Windows laptop or desktop.

### 4.1 Install and Sign In
1. Visit the [Tailscale Download Page](https://tailscale.com/download).
2. Download the version for your operating system.
3. Install the application.
4. Sign in using the same Tailscale account used on the Ubuntu server.
5. Confirm that both devices appear in the Admin Console.

Tailscale supports Windows, macOS, Linux, Android, iOS, and other platforms.

### 4.2 Test Connectivity

From your client device, try reaching the server using its Tailscale machine name:

```bash
ping media-server
```

Replace `media-server` with your actual device name.

This hostname-based example assumes MagicDNS is enabled and the name resolves correctly. Otherwise, use the server's Tailscale IP address:

```bash
ping YOUR_TAILSCALE_IP
```

You can also run:

```bash
tailscale ping media-server
```

`tailscale ping` is particularly useful for checking connectivity through Tailscale itself.

> [!NOTE] Ping Is Not a Definitive Test
> Some systems block ICMP traffic, so a failed regular `ping` does not necessarily mean Tailscale is broken. Use `tailscale ping` and test SSH as well.

---
## 5. Configure and Test SSH

With both devices connected to Tailscale, you can connect to the Ubuntu server privately.

### 5.1 Connect Through OpenSSH

From your client device, run:

```bash
ssh YOUR_USERNAME@media-server
```

Replace:
- `YOUR_USERNAME` with your Ubuntu account username.
- `media-server` with your actual Tailscale machine name.

If MagicDNS is unavailable, use the server's Tailscale IP address:

```bash
ssh YOUR_USERNAME@YOUR_TAILSCALE_IP
```

For example, the address will generally be in the private Tailscale IP range assigned to your device. Use the actual IP reported by `tailscale ip -4`.

### 5.2 Verify the SSH Host Key

The first time you connect, SSH may display a message similar to:

```bash
The authenticity of host 'media-server' can't be established.
The host key fingerprint is SHA256:EXAMPLE_FINGERPRINT.

Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is SSH asking whether you trust the server's host key.

**Do not blindly accept the prompt.** Verify the fingerprint through a trusted method, such as checking the host key fingerprint directly on your Ubuntu server:

```
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Compare the fingerprint displayed on the server with the fingerprint shown by the SSH client. If they match, accept the key and continue.

Once accepted, SSH normally stores the host key in your client's `known_hosts` file so it can detect unexpected changes in future connections.

> [!WARNING] Unexpected Host Key Changes
> If SSH later warns that the host identification has changed, investigate before continuing. A legitimate server rebuild can change its host key, but unexpected changes should not be ignored.

### 5.3 Optional: Use Tailscale SSH

Tailscale also has a feature called **Tailscale SSH**, which is separate from ordinary OpenSSH connections over the Tailscale network.
- **OpenSSH over Tailscale:** Uses your Ubuntu SSH service and its authentication configuration.
- **Tailscale SSH:** Uses Tailscale's SSH access controls and authentication mechanism for supported connections.

You do not need to enable Tailscale SSH just to use ordinary SSH over Tailscale. If you choose to enable it, review the relevant access policies and ensure that only authorized users can connect.

---
## 6. Optional Key Expiry Configuration

Tailscale devices can have key-expiry settings that require reauthentication after a period of time.

For an unattended home server, you may choose to disable key expiry for that device. This reduces the chance of losing remote access because the device's authentication expires while you are away.

### 6.1 How to Disable Key Expiry
1. Open the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).
2. Find your Ubuntu media server.
3. Open the device's menu.
4. Look for the **Disable key expiry** option.
5. Review the confirmation prompt before applying the change.

The exact labels and availability may vary by account and device configuration.

### 6.2 Security Trade-Off

Disabling key expiry means that the device may remain authenticated indefinitely unless you revoke or otherwise invalidate its access.

Consider the following:
- Keep the Ubuntu server updated.
- Protect your Tailscale account with strong authentication and MFA where available.
- Review the device list periodically.
- Remove devices you no longer use.
- Revoke access promptly if the server or account is compromised.

> [!WARNING] Optional, Not Mandatory
> Disabling key expiry is a convenience trade-off, not a requirement for Tailscale to work. Keeping key expiry enabled may be preferable if you want periodic reauthentication as part of your security policy.

---
## 7. Troubleshooting

### 7.1 The Server Does Not Appear in Tailscale

Check the service status:

```bash
sudo systemctl status tailscaled
```

Check the Tailscale connection:

```bash
tailscale status
```

If necessary, retry authentication:

```bash
sudo tailscale up
```

Follow the displayed authentication instructions and confirm that the server is authorized in the Admin Console.

### 7.2 The Server Name Does Not Resolve

Check whether MagicDNS is enabled in your Tailscale configuration.

Try connecting with the server's Tailscale IP address instead:

```bash
ssh YOUR_USERNAME@YOUR_TAILSCALE_IP
```

If the IP works but the hostname does not, investigate name resolution rather than assuming the tunnel itself is broken.

### 7.3 SSH Connection Is Refused

On the Ubuntu server, check:

```bash
sudo systemctl status ssh
sudo ss -tlnp | grep ':22'
```

Verify that OpenSSH is running and listening on the expected port.

Also confirm that the SSH service is configured to accept connections through the relevant interface and that the server firewall allows the intended traffic.

### 7.4 SSH Times Out

Check:
- Both devices are connected to Tailscale.
- The server appears online in the Admin Console.
- The Tailscale IP address is correct.
- The server firewall allows SSH from the Tailscale network.
- Tailscale access-control policies permit the connection.
- The SSH service is listening on the expected port.

If the regular `ping` command fails, try `tailscale ping` and then test SSH separately.

### 7.5 Authentication Fails

If the connection reaches the server but login fails, verify your Ubuntu username and SSH authentication method.

For better security, consider using SSH keys instead of relying solely on passwords. If you configure key-based authentication, store the private key securely and ensure its permissions are correct

---

> [!TIP] Remote Access Complete
> Your media server is now reachable from anywhere as long as that device is in your Tailscale's private network!

---

> [!TIP] ➜ Up Next
> You’ve completed  [02 Tailscale](#02-tailscale)
> **Previous:** [01 Backup](01%20Backup.md)  ------ **Next:** [03 Notifications](03%20Notifications.md)
