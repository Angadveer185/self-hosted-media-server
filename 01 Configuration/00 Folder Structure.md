# Overview
This directory structure outlines how I organize the entire server. It separates Docker configuration, application configurations, backup scripts, server automation, monitoring, and Docker Compose stack definitions to keep everything organized and maintainable.

The `configs/` directory stores persistent application settings, `backups/` contains scripts for backing up server data, and `stacks/` holds the Docker Compose configuration used to deploy the media server.

This separation makes the server easier to manage, troubleshoot, back up, and reproduce on another system.


# Main Server
```text
/srv/
└── docker/                         # Root directory for Docker
    ├── backups/                    # Backup scripts and related files
    │   └── backup-media-server.sh  # Script to backup server data
    │
    ├── configs/                    # App configurations
    │   ├── bazarr/                 # Subtitle management
    │   ├── gluetun/                # VPN container configuration
    │   ├── jellyfin/               # Media streaming server
    │   ├── jellyseerr/             # Media request management
    │   ├── portainer/              # Docker management
    │   ├── prowlarr/               # Indexer management
    │   ├── qbittorrent/            # Torrent client
    │   ├── radarr/                 # Movie library management
    │   └── sonarr/                 # TV show library management
    │
    └── stacks/                     # Docker Compose project definitions
        └── media-stack/            # Media server stack
            ├── .env                # Store sensitive information
            └── docker-compose.yml  # Main YAML File
```

# Media

I used an external drive which I mounted onto my server. You may use internal storage if you want. In which case, your path would differ from mine.
```text
/mnt
└── media-hdd
    ├── downloads/                # Staging area for torrents
    └── media/
        ├── anime/                # Anime series and movies
        ├── movies/               # Anime series and movies
        └── tv/                   # Episodic television series
```
