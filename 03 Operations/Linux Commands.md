# Overview

A quick reference to commonly used Linux commands for file management, system administration, networking, process monitoring, and package management.

---
## 1. Directory & File Management

### `mkdir` | Create Directories

Creates new directories.

|Flag|Description|
|---|---|
|`-p`|Create parent directories as needed.|
|`-v`|Display each directory created.|

```bash
mkdir projects
mkdir -p projects/game/assets
mkdir -pv projects/game/assets
```

### `ls` | List Directory Contents

Lists files and directories.

|Flag|Description|
|---|---|
|`-l`|Long listing format.|
|`-a`|Include hidden files.|
|`-h`|Human-readable file sizes, typically with `-l`.|
|`-R`|List directories recursively.|
|`-t`|Sort by modification time, newest first.|

```bash
ls
ls -lah
ls -lt
ls -R
```

### `cd` | Change Directory

Changes the current working directory.

```bash
cd /srv/docker       # Navigate to a specific directory
cd ..                # Move to the parent directory
cd ~                 # Go to the home directory
cd -                 # Return to the previous directory
```

### `pwd` | Print Working Directory

Displays the absolute path of the current directory.

```bash
pwd
```

### `cp` | Copy Files and Directories

Copies files or directories.

|Flag|Description|
|---|---|
|`-r`|Copy directories recursively.|
|`-v`|Display copied files.|
|`-i`|Prompt before overwriting.|
|`-u`|Copy when the source is newer or the destination is missing.|

```bash
cp file.txt backup.txt
cp -rv folder/ backup/
cp -i file.txt destination.txt
cp -u source.txt destination.txt
```

### `mv` | Move or Rename Files

Moves files and directories or renames them.

|Flag|Description|
|---|---|
|`-v`|Display moved files.|
|`-i`|Prompt before overwriting.|
|`-u`|Move when the source is newer or the destination is missing.|

```bash
mv old.txt new.txt
mv file.txt /srv/docker/
mv -iv file.txt destination/
```

### `rm` | Remove Files and Directories

Deletes files and directories.

|Flag|Description|
|---|---|
|`-r`|Remove directories and their contents recursively.|
|`-f`|Force removal without prompting.|
|`-v`|Display removed files.|

```bash
rm file.txt
rm -r old_directory/
rm -rv temporary_files/
```

**Warning:** `rm -rf` can permanently delete large amounts of data. Double-check the target path before using it.

### `touch` | Create Files or Update Timestamps

Creates empty files if they do not exist, or updates timestamps if they do.

|Flag|Description|
|---|---|
|`-a`|Change the access time.|
|`-m`|Change the modification time.|

```bash
touch notes.txt
touch -a file.txt
touch -m file.txt
```

### `cat` | Display File Contents

Displays or combines file contents.

|Flag|Description|
|---|---|
|`-n`|Number all output lines.|
|`-b`|Number non-empty output lines.|

```bash
cat notes.txt
cat -n notes.txt
cat -b notes.txt
```

### `find` | Search for Files and Directories

Searches directory trees using names, types, sizes, and other conditions.

|Option|Description|
|---|---|
|`-name PATTERN`|Match names using a pattern.|
|`-type f`|Search for regular files.|
|`-type d`|Search for directories.|
|`-size SIZE`|Filter by file size.|
|`-exec COMMAND {} \;`|Execute a command for each result.|

```bash
find /srv/docker -name "*.yml"
find /srv/docker -type d
find /var/log -type f -size +100M
find . -type f -name "*.log" -exec ls -lh {} \;
```

---
## 2. File Permissions & Ownership

### `chmod` | Change Permissions

Changes the permissions assigned to files and directories.

|Option|Description|
|---|---|
|`-R`|Apply changes recursively.|

```bash
chmod 755 script.sh
chmod 644 config.txt
chmod u+x script.sh
chmod -R 755 directory/
```

**Common numeric permissions**

|Permission|Meaning|
|---|---|
|`755`|Owner can read, write, and execute; others can read and execute.|
|`644`|Owner can read and write; others can read.|
|`600`|Only the owner can read and write.|

### `chown` | Change Ownership

Changes the owner and optionally the group of a file or directory.

|Option|Description|
|---|---|
|`-R`|Apply changes recursively.|

```bash
chown user file.txt
chown user:group file.txt
chown -R user:group directory/
```

---
## 3. Process Management

### `ps` | List Processes

Displays information about running processes.

|Option|Description|
|---|---|
|`-e`|Show all processes.|
|`-f`|Display full-format process information.|
|`aux`|Common BSD-style combination showing processes for all users with detailed information.|

```bash
ps
ps -ef
ps aux
```

### `top` / `htop` | Monitor Processes

Displays processes and system resource usage in real time. `htop` provides an interactive interface and may need to be installed separately.

```bash
top
htop
```

### `kill` | Send a Signal to a Process

Sends a signal to a process identified by its PID.

|Signal|Purpose|
|---|---|
|`-15` (`SIGTERM`)|Request graceful termination.|
|`-9` (`SIGKILL`)|Force immediate termination.|

```bash
kill 1234
kill -15 1234
kill -9 1234
```

Use `SIGTERM` first when possible. `SIGKILL` prevents the process from cleaning up before it exits.

---
## 4. System Services & Logs

### `systemctl` | Manage Services

Controls services managed by systemd.

|Command|Description|
|---|---|
|`start SERVICE`|Start a service.|
|`stop SERVICE`|Stop a service.|
|`restart SERVICE`|Restart a service.|
|`status SERVICE`|Check service status.|
|`enable SERVICE`|Enable a service at boot.|
|`disable SERVICE`|Disable automatic startup.|

```bash
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
systemctl status docker
sudo systemctl enable docker
```

### `journalctl` | Read System Logs

Reads logs collected by the systemd journal.

|Option|Description|
|---|---|
|`-u SERVICE`|Show logs for a specific service.|
|`-f`|Follow new log entries.|
|`--since TIME`|Show entries since a specified time.|

```bash
journalctl -u docker
journalctl -u docker -f
journalctl --since "1 hour ago"
journalctl -u docker --since today
```

---
## 5. Downloads & Remote Access

### `wget` | Download Files

Downloads files from web servers.

|Option|Description|
|---|---|
|`-O FILE`|Save the output to a specified filename.|
|`-c`|Continue an incomplete download when supported by the server.|

```bash
wget https://example.com/file.zip
wget -O archive.zip https://example.com/file.zip
wget -c https://example.com/large-file.iso
```

### `curl` | Transfer Data

Transfers data to or from URLs. It is commonly used to download files and test HTTP endpoints.

|Option|Description|
|---|---|
|`-O`|Save using the remote filename.|
|`-o FILE`|Save to a specified filename.|
|`-L`|Follow HTTP redirects.|

```bash
curl -L https://example.com/file.zip -o file.zip
curl -O https://example.com/file.zip
curl -I https://example.com
```

### `ssh` | Connect to a Remote System

Opens a secure remote shell session.

|Option|Description|
|---|---|
|`-i KEYFILE`|Specify the private key file.|
|`-p PORT`|Specify the SSH port.|

```bash
ssh user@192.168.1.100
ssh -i ~/.ssh/id_ed25519 user@server
ssh -p 2222 user@server
```

### `scp` | Securely Copy Files

Copies files between local and remote systems over SSH.

|Option|Description|
|---|---|
|`-r`|Copy directories recursively.|
|`-P PORT`|Specify the SSH port.|

```bash
scp file.txt user@server:/home/user/
scp -r folder/ user@server:/home/user/
scp -P 2222 file.txt user@server:/home/user/
```

---
## 6. Archiving & Compression

### `tar` | Archive and Extract Files

Creates archives and extracts their contents.

|Option|Description|
|---|---|
|`-c`|Create an archive.|
|`-x`|Extract an archive.|
|`-v`|Display processed files.|
|`-f FILE`|Specify the archive filename.|
|`-z`|Use gzip compression.|
|`-j`|Use bzip2 compression.|

**Examples**

```bash
# Create a gzip-compressed archive.
tar -czvf backup.tar.gz folder/

# Extract a gzip-compressed archive.
tar -xzvf backup.tar.gz

# Create a bzip2-compressed archive.
tar -cjvf backup.tar.bz2 folder/

# List the contents of an archive.
tar -tzvf backup.tar.gz
```

---
## 7. Package Management

### `apt` / `apt-get` | Manage Packages

Used on Debian- and Ubuntu-based Linux distributions to manage software packages.

|Command|Description|
|---|---|
|`update`|Refresh package lists.|
|`upgrade`|Upgrade installed packages.|
|`install PACKAGE`|Install a package.|
|`remove PACKAGE`|Remove a package while generally retaining its configuration files.|

```bash
sudo apt update
sudo apt upgrade
sudo apt install htop
sudo apt remove htop
```

**Note:** `apt update` refreshes package information. It does not upgrade installed software by itself.

---
## 8. Command History

### `history` | View Previous Commands

Displays commands entered in the shell.

```bash
history
```

Run a command by its history number:

```bash
!42
```

Replace `42` with the desired command's history index. Review the command before executing it, especially if it contains destructive operations.

---
## 9. Networking

### `ping` | Test Connectivity

Sends ICMP echo requests to check whether a host is reachable.

|Option|Description|
|---|---|
|`-c COUNT`|Send a specified number of requests.|

```bash
ping 8.8.8.8
ping -c 4 google.com
```

### `ip` | Inspect Network Configuration

Displays and manages network interfaces, IP addresses, and routing information.

|Command|Description|
|---|---|
|`ip addr`|Display IP addresses.|
|`ip link`|Display network interfaces and their states.|
|`ip route`|Display the routing table.|

```bash
ip addr
ip link
ip route
```

---
