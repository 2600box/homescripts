# README.md

This is my own location that I used to store my scripts for my home Linux server. This is configured for my setup, but welcome any questions.

## Home Configuration

- Verizon Gigabit FIOS
- Google Drive with an Encrypted Media Folder
- Debian Stretch 9.5
- Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
- 32 GB of Memory
- 250 GB SSD Storage for my root
- 6TB mirrored for staging
- rTorrent, NZBGet, Sonarr, Radarr and Ombi all run locally on my mergerfs mount that allows hard linking of files.

## OpenVPN Configuration
For all my private traffic, I use [TorGuard](https://torguard.net/) as they support port forwarding and have very good support.

[Setup and Configuration](https://github.com/animosity22/homescripts/blob/master/OPENVPN.MD)

## Rclone configuration

I use a combination of mergerfs and rclone to keep a local mount that is always written to first and my second mount point is my rclone Google Drive.

        /data/local (local disk)
        /GD (rclone mount)
    /gmedia

[rclone.conf](https://github.com/animosity22/homescripts/blob/master/rclone.conf)

They all get mounted up via my systemd scripts for [gmedia-service](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.service).

My gmedia starts up items in order:
1) [rclone mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-rclone.service)
2) [mergerfs mount](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia.mount)
3) [find command](https://github.com/animosity22/homescripts/blob/master/rclone-systemd/gmedia-find.service) which justs caches the directory and file structure and provides me an output of the structure

## [mergerfs@github](https://github.com/trapexit/mergerfs)

I use mergerfs over unionfs as it provides me the ability to define a file system to always write first to.

I use the following options:

```bash
Options = use_ino,hard_remove,auto_cache,sync_read,allow_other,category.action=all,category.create=ff
```

Two important items:

- sync_read as rclone is default built with this and is required for proper streaming
- category.action=all,category.create=ff says to always create directories / files on the first listed mount point and for my configuration that is my /data/mounts/local

## Scheduled Nightly Uploads

I moved my files to my GD every ngiht via a cron job and an [upload cloud](https://github.com/animosity22/homescripts/blob/master/scripts/upload_cloud) script. This leverages an [excludes](https://github.com/animosity22/homescripts/blob/master/scripts/excludes) file which gets rid of partials and my torrent directory.

## Plex and Caddy Configuration

I use [Caddy](https://github.com/mholt/caddy) as a proxy server and route all my items through it. I build via this [script](https://github.com/animosity22/homescripts/blob/master/scripts/build_caddy).

My plex configuration in my CaddyFile as follows:

```bash
# Plex Server
plex.animosity.us {
gzip
timeouts 1h
log /opt/caddy/logs/plex.log
tls {
  dns cloudflare
}
proxy / 127.0.0.1:32400 {
 transparent
 websocket
 keepalive 12
 timeout 1h
    }
}
```

Plex also needs to have a few additional steps to be added so Caddy is used in all situations.

### Plex

Remote Access - Disable

```
Network - Custom server access URLs = https://<your-domain>:443,http://<your-domain>:80
```
Network - Secure connections = Preferred

<i>Note: You can force SSL by setting required and not adding the HTTP URL, however some players which do not support HTTPS (e.g: Roku, Playstations, some SmartTVs) will no longer function. I only use SSL is my config and as I do not open up HTTP traffic. </i>

Optional Requirements

### UFW or other firewall:

Deny port 32400 externally (Plex still pings over 32400, some clients may use 32400 by mistake despite 443 and 80 being set).

<i>Note adding allowLocalhostOnly="1" to your Preferences.xml, will make Plex only listen on the localhost, achieving the same thing as using a firewall and this is what I use in my configuration.</i>

## Known Issues

- Plex Playback
  - Apple TV (4th generation)
    - Direct Play Stuttering
      - This happens on both vfs-read-chunk-size and cache. Cache masks this more so since the chunks remain local. If you turn off "Allow Direct Play", this will fix the issue as it will Direct Stream instead. Using another player such as Infuse / Emby works as well as they do not exihibit the Direct Play issue. I will retest this once TVOS 12 hits to see if it has been fixed or not.
      - RClone debug log shows the files being rapidly opened and closed as the client seems to request part of the file and close it out.
      - Another work around that seems to be going well for me is using [Caddy](https://github.com/mholt/caddy) as a proxy server.
