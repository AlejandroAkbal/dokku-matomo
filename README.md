# Matomo on Dokku

Deploy your own instance of [Matomo](https://matomo.org) on
[Dokku](https://github.com/dokku/dokku).

This guide makes use of the great [`docker-matomo`](https://github.com/crazy-max/docker-matomo) image by [CrazyMax](https://github.com/crazy-max), which does all the heavy-lifting to properly deploy Matomo without too much hassle.

## Preface

You will deploy [Matomo 4.12.3](https://github.com/matomo-org/matomo) onto your own
Dokku server.

Keep in mind that all the steps below are meant to be executed on the Dokku server.

### Limitations

Currently, the following features are not supported:

- Redis cache

Contributions are welcome!

### Requirements

- [Dokku](https://github.com/dokku/dokku)
- [Dokku "dokku-mariadb" plugin](https://github.com/dokku/dokku-mariadb)
- [Dokku "dokku-letsencrypt" plugin](https://github.com/dokku/dokku-letsencrypt)

## Setup

### App

First create a new Dokku app. We will call it `my-matomo`.

```sh
dokku apps:create my-matomo
```

### Database

Next create the MariaDB database required by Matomo and link it to the app.

```sh
# Optional, set `max_allowed_packet` to at least 64MB, as Matomo recommends
dokku mariadb:create my-matomo-db --config-options "--max-allowed-packet=67108864"

dokku mariadb:link my-matomo-db my-matomo
```

### Configuration

#### Environment variables

Configure Matomo with your preferences.
Here are some useful defaults:

```sh
dokku config:set my-matomo --no-restart matomo TZ=Europe/Berlin
dokku config:set my-matomo --no-restart matomo PUID=1000
dokku config:set my-matomo --no-restart matomo PGID=1000
dokku config:set my-matomo --no-restart matomo MEMORY_LIMIT=256M
dokku config:set my-matomo --no-restart matomo UPLOAD_MAX_SIZE=16M
dokku config:set my-matomo --no-restart matomo OPCACHE_MEM_SIZE=128
dokku config:set my-matomo --no-restart matomo REAL_IP_FROM=0.0.0.0/32
dokku config:set my-matomo --no-restart matomo REAL_IP_HEADER=X-Forwarded-For
dokku config:set my-matomo --no-restart matomo LOG_LEVEL=WARN
```

[View all configuration options](https://github.com/crazy-max/docker-matomo#environment-variables).

#### Persistent storage

You need to mount a volume to persist all the settings that you set in the Matomo interface.

```sh
mkdir /var/lib/dokku/data/storage/my-matomo

# UID:GUID are set to 1000. These are the values the docker-matomo image uses.
sudo chown 1000:1000 -R /var/lib/dokku/data/storage/my-matomo

dokku storage:mount my-matomo /var/lib/dokku/data/storage/my-matomo:/data
```

#### Domain setup

To get the routing working, we need to apply a few settings.
First, set the domain.

```sh
dokku domains:set my-matomo matomo.example.com
```

You will also need to update the ports set by Dokku.

```sh
dokku proxy:ports-add my-matomo http:80:8000
dokku proxy:ports-remove my-matomo http:80:5000
```

If Dokku proxy:report sentry shows more than one port mapping, remove all port mappings except the added above.

#### Email settings (optional)

Set the following settings if you want to receive emails from Matomo.

```sh
dokku config:set my-matomo --no-restart SSMTP_HOST=smtp.example.com
dokku config:set my-matomo --no-restart SSMTP_PORT=587
dokku config:set my-matomo --no-restart SSMTP_HOSTNAME=matomo.example.com
dokku config:set my-matomo --no-restart SSMTP_USER=user@example.com
dokku config:set my-matomo --no-restart SSMTP_PASSWORD=yoursmtppassword
dokku config:set my-matomo --no-restart SSMTP_TLS=YES
```

#### GeoIP2

The GeoIP2 plugin is already installed in the docker image, but needs to be configured.

[Follow the instructions in the crazymax/docker-matomo repository](https://github.com/crazy-max/docker-matomo#geoip2) to enable it.

### Advanced configuration

If needed, the Matomo configuration file is located at `/var/lib/dokku/data/storage/matomo/config/config.ini.php` and can be manually edited.

#### Cron jobs

You can enable automatic archiving by using the sidecar.
You will need to set up an external cron job to trigger the archiving.

```sh
sudo nano /etc/cron.d/dokku-my-matomo
```

Add the following lines to the crontab:

```sh
# This file should be located at /etc/cron.d/dokku-my-matomo

MAILTO="email@example.com"
PATH=/usr/local/bin:/usr/bin:/bin
SHELL=/bin/bash

# m   h   dom mon dow   username command
# *   *   *   *   *     dokku    command to be executed
# -   -   -   -   -
# |   |   |   |   |
# |   |   |   |   +----- day of week (0 - 6) (Sunday=0)
# |   |   |   +------- month (1 - 12)
# |   |   +--------- day of month (1 - 31)
# |   +----------- hour (0 - 23)
# +----------- min (0 - 59)

### HIGH TRAFFIC TIME IS B/W 00:00 - 04:00 AND 14:00 - 23:59
### RUN YOUR TASKS FROM 04:00 - 14:00
### KEEP SORTED IN TIME ORDER

### PLACE ALL CRON TASKS BELOW

# Run matomo archive
*/30 * * * * dokku dokku run --env "ARCHIVE_OPTIONS=--concurrent-requests-per-website=3" matomo-akbal-dev "/usr/local/bin/matomo_archive"

### PLACE ALL CRON TASKS ABOVE, DO NOT REMOVE THE WHITESPACE AFTER THIS LINE

```

Please read [CrazyMax's Cron documentation](https://github.com/crazy-max/docker-matomo#cron) for more information.

## Deploy

Now, all you need to do is use CrazyMax's docker image to deploy Matomo.

```sh
dokku git:from-image my-matomo crazymax/matomo:4.12.3
```

[Read more about the `git:from-image` command](https://dokku.com/docs~v0.26.6/deployment/methods/git/#initializing-an-app-repository-from-a-docker-image) in case of problems.

### Matomo web setup

You need to set up Matomo in the web interface and provide the database details.
You should be able to access the web via [`https://matomo.example.com`](https://matomo.example.com).

The database details can be accessed by running the command below to retrieve the DSN.

```sh
dokku mariadb:info my-matomo-db
```

An example DSN might look like this:
`mysql://mariadb:ffd4fc238ba8adb3@dokku-mariadb-my-matomo-db:3306/my_matomo_db`.
Copy and paste the details as follows:

```sh
Hostname: dokku-mariadb-my-matomo-db
Username: mariadb
Password: ffd4fc238ba8adb3
Database Name: my_matomo_db
```

After going through the setup, you should be able to use Matomo.

## Credit

This tutorial was made possible thanks to the following people:

- [CrazyMax](https://github.com/crazy-max)
- [Romain Clement](https://github.com/rclement/dokku-matomo)'s original tutorial
