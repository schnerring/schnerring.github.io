---
title: "Install qBittorrent, Jackett, Lidarr, Radarr, Sonarr, and Plex inside TrueNAS Jails"
date: 2021-11-14T20:41:04+01:00
draft: false
comments: true
toc: true
cover:
  src: "cover.png"
tags:
  - FreeBSD
  - Homelab
  - Iocage
  - Jackett
  - Jail
  - Lidarr
  - Plex
  - qBittorrent
  - Radarr
  - Self-hosted
  - Sonarr
  - Torrent
  - TrueNAS
---

This post documents the steps required to install [qBittorrent](https://www.qbittorrent.org/), [Jackett](https://github.com/Jackett/Jackett), [Lidarr](https://lidarr.audio/), [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/), and [Plex](https://www.plex.tv/) in [TrueNAS jails](https://www.truenas.com/docs/core/applications/jails/) version `12.0-U6`.

<!--more-->

The [FN11.3 iocage jails - Plex, Tautulli, Sonarr, Radarr, Lidarr, Jackett, Transmission, Organizr](https://www.truenas.com/community/resources/fn11-3-iocage-jails-plex-tautulli-sonarr-radarr-lidarr-jackett-transmission-organizr.58/) guide inspired me to write this guide. I'll also briefly cover permissions.

## Disclaimer

[From Wikipedia on BitTorrent Legislation](https://en.wikipedia.org/wiki/BitTorrent#Legislation):

> Although the protocol itself is legal, problems stem from using the protocol to traffic copyright infringing works, since BitTorrent is often used to download otherwise paid content, such as movies and video games.

See also: [Legal issues with BitTorrent (Wikipedia)](https://en.wikipedia.org/wiki/Legal_issues_with_BitTorrent)

I'll only cover options that deviate from the defaults. I use DHCP reservations to manage my server IPs, so I use the `dhcp=1` option to create jails.

## Goals

The diagram above translates to the following requirements:

- Each service lives inside a separate jail
- Each service inside the jail runs as a different user
- Each jail owns a dataset for configuration data
- Jails share media datasets, but only one user has write permissions. E.g., Radarr can only read (`ro`) from the torrent dataset but write (`rw`) to the movies dataset.

## Groups

Navigate to {{< breadcrumb "Accounts" "Groups" >}} and add the following groups.

|                      |               |
| -------------------- | ------------- |
| GID                  | `850`         |
| Name                 | `qbittorrent` |
| Samba Authentication | `unchecked`   |

|                      |             |
| -------------------- | ----------- |
| GID                  | `354`       |
| Name                 | `jackett`   |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| GID                  | `356`       |
| Name                 | `lidarr`    |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| GID                  | `352`       |
| Name                 | `radarr`    |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| GID                  | `351`       |
| Name                 | `sonarr`    |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| GID                  | `972`       |
| Name                 | `plex`      |
| Samba Authentication | `unchecked` |

## Users

Navigate to {{< breadcrumb "Accounts" "Users" >}} and add the following users.

|                      |               |
| -------------------- | ------------- |
| Full Name            | `qBittorrent` |
| Username             | `qbittorrent` |
| User ID              | `850`         |
| New Primary Group    | `unchecked`   |
| Primary Group        | `qbittorrent` |
| Primary Group        | `qbittorrent` |
| Disable Password     | `Yes`         |
| Shell                | `nologin`     |
| Samba Authentication | `unchecked`   |

|                      |             |
| -------------------- | ----------- |
| Full Name            | `Jackett`   |
| Username             | `jackett`   |
| User ID              | `354`       |
| New Primary Group    | `unchecked` |
| Primary Group        | `jackett`   |
| Disable Password     | `Yes`       |
| Shell                | `nologin`   |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| Full Name            | `Lidarr`    |
| Username             | `lidarr`    |
| User ID              | `356`       |
| New Primary Group    | `unchecked` |
| Primary Group        | `lidarr`    |
| Disable Password     | `Yes`       |
| Shell                | `nologin`   |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| Full Name            | `Radarr`    |
| Username             | `radarr`    |
| User ID              | `352`       |
| New Primary Group    | `unchecked` |
| Primary Group        | `radarr`    |
| Disable Password     | `Yes`       |
| Shell                | `nologin`   |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| Full Name            | `Sonarr`    |
| Username             | `sonarr`    |
| User ID              | `351`       |
| New Primary Group    | `unchecked` |
| Primary Group        | `sonarr`    |
| Disable Password     | `Yes`       |
| Shell                | `nologin`   |
| Samba Authentication | `unchecked` |

|                      |             |
| -------------------- | ----------- |
| Full Name            | `Plex`      |
| Username             | `plex`      |
| User ID              | `972`       |
| New Primary Group    | `unchecked` |
| Primary Group        | `plex`      |
| Disable Password     | `Yes`       |
| Shell                | `nologin`   |
| Samba Authentication | `unchecked` |

## Datasets

Navigate to {{< breadcrumb "Storage" "Pools" >}} and add the datasets and permissions. We'll use default `755` permissions for all datasets.

### Jail Config Datasets

|             |                                |
| ----------- | ------------------------------ |
| Path        | `/mnt/vault0/apps/qbittorrent` |
| User        | `qbittorrent`                  |
| Apply User  | `checked`                      |
| Group       | `qbittorrent`                  |
| Apply Group | `checked`                      |

|             |                            |
| ----------- | -------------------------- |
| Path        | `/mnt/vault0/apps/jackett` |
| User        | `jackett`                  |
| Apply User  | `checked`                  |
| Group       | `jackett`                  |
| Apply Group | `checked`                  |

|             |                           |
| ----------- | ------------------------- |
| Path        | `/mnt/vault0/apps/lidarr` |
| User        | `lidarr`                  |
| Apply User  | `checked`                 |
| Group       | `lidarr`                  |
| Apply Group | `checked`                 |

|             |                           |
| ----------- | ------------------------- |
| Path        | `/mnt/vault0/apps/radarr` |
| User        | `radarr`                  |
| Apply User  | `checked`                 |
| Group       | `radarr`                  |
| Apply Group | `checked`                 |

|             |                           |
| ----------- | ------------------------- |
| Path        | `/mnt/vault0/apps/sonarr` |
| User        | `sonarr`                  |
| Apply User  | `checked`                 |
| Group       | `sonarr`                  |
| Apply Group | `checked`                 |

|             |                         |
| ----------- | ----------------------- |
| Path        | `/mnt/vault0/apps/plex` |
| User        | `plex`                  |
| Apply User  | `checked`               |
| Group       | `plex`                  |
| Apply Group | `checked`               |

### Media Datasets

|             |                              |
| ----------- | ---------------------------- |
| Path        | `/mnt/vault0/media/torrents` |
| User        | `qbittorrent`                |
| Apply User  | `checked`                    |
| Group       | `qbittorrent`                |
| Apply Group | `checked`                    |

|             |                           |
| ----------- | ------------------------- |
| Path        | `/mnt/vault0/media/music` |
| User        | `lidarr`                  |
| Apply User  | `checked`                 |
| Group       | `lidarr`                  |
| Apply Group | `checked`                 |

|             |                            |
| ----------- | -------------------------- |
| Path        | `/mnt/vault0/media/movies` |
| User        | `radarr`                   |
| Apply User  | `checked`                  |
| Group       | `radarr`                   |
| Apply Group | `checked`                  |

|             |                            |
| ----------- | -------------------------- |
| Path        | `/mnt/vault0/media/series` |
| User        | `sonarr`                   |
| Apply User  | `checked`                  |
| Group       | `sonarr`                   |
| Apply Group | `checked`                  |

## Jails

Connect to TrueNAS via SSH or similar.

### qBittorrent Jail

```bash
# Create jail
iocage create --name qbittorrent --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec qbittorrent mkdir /mnt/config
iocage fstab --add qbittorrent /mnt/vault0/apps/qbittorrent /mnt/config nullfs rw 0 0
# Mount media dataset
iocage exec qbittorrent mkdir /mnt/torrents
iocage fstab --add qbittorrent /mnt/vault0/media/torrents /mnt/torrents nullfs rw 0 0
# Update packages
iocage exec qbittorrent "pkg update && pkg upgrade"
# Install qBittorrent without GUI
iocage exec qbittorrent pkg install qbittorrent-nox
# Enable qBittorrent service
iocage exec qbittorrent sysrc qbittorrent_enable=YES
# Configure config directory
iocage exec qbittorrent sysrc qbittorrent_conf_dir=/mnt/config
# Start the service
iocage exec qbittorrent service qbittorrent start
```

Navigate to `http://<jail IP>:8080` in your browser and login with the default credentials.

| Username | Password     |
| -------- | ------------ |
| `admin`  | `adminadmin` |

Navigate to {{< breadcrumb "Tools" "Options..." >}} and change the **Default Save Path** to `/mnt/torrents`:

![Screenshot of qBittorrent options](qbittorrent-options.png)

### Jackett Jail

```bash
# Create jail
iocage create --name jackett --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec jackett mkdir /mnt/config
iocage fstab --add jackett /mnt/vault0/apps/jackett /mnt/config nullfs rw 0 0
# Change pkg repository set from `quarterly` to `latest`
# At the time of this writing, `jackett` from the quarterly repo wasn't working
iocage exec jackett sed -i '' 's/quarterly/latest/g' /etc/pkg/FreeBSD.conf
# Update packages
iocage exec jackett "pkg update && pkg upgrade"
# Install
iocage exec jackett pkg install jackett
# Enable service
iocage exec jackett sysrc jackett_enable=YES
# Configure config directory
iocage exec jackett sysrc jackett_data_dir=/mnt/config
# Start the service
iocage exec jackett service jackett start
```

Navigate to `http://<jail IP>:9117` in your browser to use Jackett.

### Lidarr Jail

```bash
# Create jail
iocage create --name lidarr --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec lidarr mkdir /mnt/config
iocage fstab --add lidarr /mnt/vault0/apps/lidarr /mnt/config nullfs rw 0 0
# Mount media datasets
iocage exec lidarr mkdir /mnt/music
iocage fstab --add lidarr /mnt/vault0/media/music /mnt/music nullfs rw 0 0
iocage exec lidarr mkdir /mnt/torrents
iocage fstab --add lidarr /mnt/vault0/media/torrents /mnt/torrents nullfs ro 0 0
# Change pkg repository set from `quarterly` to `latest`
iocage exec lidarr sed -i '' 's/quarterly/latest/g' /etc/pkg/FreeBSD.conf
# Update packages
iocage exec lidarr "pkg update && pkg upgrade"
# Install
iocage exec lidarr pkg install lidarr
# Enable service
iocage exec lidarr sysrc lidarr_enable=YES
# Configure config directory
iocage exec lidarr sysrc lidarr_data_dir=/mnt/config
# Start the service
iocage exec lidarr service lidarr start
```

Navigate to `http://<jail IP>:8686` in your browser to use Lidarr.

### Radarr Jail

```bash
# Create jail
iocage create --name radarr --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec radarr mkdir /mnt/config
iocage fstab --add radarr /mnt/vault0/apps/radarr /mnt/config nullfs rw 0 0
# Mount media datasets
iocage exec radarr mkdir /mnt/movies
iocage fstab --add radarr /mnt/vault0/media/movies /mnt/movies nullfs rw 0 0
iocage exec radarr mkdir /mnt/torrents
iocage fstab --add radarr /mnt/vault0/media/torrents /mnt/torrents nullfs ro 0 0
# Change pkg repository set from `quarterly` to `latest`
iocage exec radarr sed -i '' 's/quarterly/latest/g' /etc/pkg/FreeBSD.conf
# Update packages
iocage exec radarr "pkg update && pkg upgrade"
# Install
iocage exec radarr pkg install radarr
# Enable service
iocage exec radarr sysrc radarr_enable=YES
# Configure config directory
iocage exec radarr sysrc radarr_data_dir=/mnt/config
# Start the service
iocage exec radarr service radarr start
```

Navigate to `http://<jail IP>:7878` in your browser to use Radarr.

### Sonarr Jail

```bash
# Create jail
iocage create --name sonarr --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec sonarr mkdir /mnt/config
iocage fstab --add sonarr /mnt/vault0/apps/sonarr /mnt/config nullfs rw 0 0
# Mount media datasets
iocage exec sonarr mkdir /mnt/series
iocage fstab --add sonarr /mnt/vault0/media/series /mnt/series nullfs rw 0 0
iocage exec sonarr mkdir /mnt/torrents
iocage fstab --add sonarr /mnt/vault0/media/torrents /mnt/torrents nullfs ro 0 0
# Change pkg repository set from `quarterly` to `latest`
iocage exec sonarr sed -i '' 's/quarterly/latest/g' /etc/pkg/FreeBSD.conf
# Update packages
iocage exec sonarr "pkg update && pkg upgrade"
# Install
iocage exec sonarr pkg install sonarr
# Enable service
iocage exec sonarr sysrc sonarr_enable=YES
# Configure config directory
iocage exec sonarr sysrc sonarr_data_dir=/mnt/config
# Start the service
iocage exec sonarr service sonarr start
```

Navigate to `http://<jail IP>:8989` in your browser to use Sonarr.

### Plex Jail

```bash
# Create jail
iocage create --name plex --release 12.2-RELEASE dhcp=1 boot=1
# Mount jail config dataset
iocage exec plex mkdir /mnt/config
iocage fstab --add plex /mnt/vault0/apps/plex /mnt/config nullfs rw 0 0
# Mount media datasets
iocage exec plex mkdir /mnt/music
iocage fstab --add plex /mnt/vault0/media/music /mnt/music nullfs ro 0 0
iocage exec plex mkdir /mnt/series
iocage fstab --add plex /mnt/vault0/media/series /mnt/series nullfs ro 0 0
iocage exec plex mkdir /mnt/movies
iocage fstab --add plex /mnt/vault0/media/movies /mnt/movies nullfs ro 0 0
# Update packages
iocage exec plex "pkg update && pkg upgrade"
# Install
iocage exec plex pkg install plexmediaserver
# Enable service
iocage exec plex sysrc plexmediaserver_enable=YES
# Configure config directory
iocage exec plex sysrc plexmediaserver_support_path=/mnt/config
# Start the service
iocage exec plex service plexmediaserver start
```

Navigate to `http://<jail IP>:32400/web` in your browser to use Plex.
