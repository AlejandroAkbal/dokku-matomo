# Matomo on Dokku

Deploy your own instance of [Matomo](https://matomo.org) on
[Dokku](https://github.com/dokku/dokku).

This guide makes use of the great [`docker-matomo`](https://github.com/crazy-max/docker-matomo) image by [CrazyMax](https://github.com/crazy-max), which does all the heavy-lifting to properly deploy Matomo without too much hassle.

## Preface

You will deploy [Matomo 4.1.1\*](https://github.com/matomo-org/matomo/releases/tag/4.1.1) onto your own
Dokku server.

Keep in mind that all the steps below are meant to be executed on the Dokku server.

\*: Please read the [upgrading section](#upgrading) to see why this is the latest working version.

## Limitations

Currently, the following features are not supported:

- Archive via cron
- Redis cache

Contributions are welcome!

## Requirements

- [Dokku 0.26.6](https://github.com/dokku/dokku)
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
dokku mariadb:create my-matomo-db
dokku mariadb:link my-matomo-db my-matomo
```

### Configuration

#### Environment variables

Configure Matomo with your preferences.
Here are some useful defaults:

```sh
dokku config:set my-matomo --no-restart matomo TZ=Europe/Berlin
dokku config:set my-matomo --no-restart matomo MEMORY_LIMIT=256M
dokku config:set my-matomo --no-restart matomo UPLOAD_MAX_SIZE=16M
dokku config:set my-matomo --no-restart matomo OPCACHE_MEM_SIZE=128
dokku config:set my-matomo --no-restart matomo REAL_IP_FROM=0.0.0.0/32
dokku config:set my-matomo --no-restart matomo REAL_IP_HEADER=X-Forwarded-For
dokku config:set my-matomo --no-restart matomo LOG_LEVEL=WARN
```

#### Persistent storage

You need to mount a volume to persist all the settings that you set in the Matomo interface.

```sh
mkdir /var/lib/dokku/data/storage/my-matomo

# UID:GUID are set to 101. These are the values the nginx image uses,
# that is used by crazymax/matomo
chown 101:101 /var/lib/dokku/data/storage/my-matomo

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

If Dokku proxy:report sentry shows more than one port mapping, remove all port
mappings except the added above.

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

[Follow the instructions in the crazymax/docker-matomo repository](https://github.com/crazy-max/-matomo#geoip2) to enable it.

### Advanced configuration

If needed, the Matomo configuration file is located at `/var/lib/dokku/data/storage/matomo/config/config.ini.php` and can be manually edited.

## Deploy

Now, all you need to do is use CrazyMax's docker image to deploy Matomo.

```sh
dokku git:from-image my-matomo crazymax/matomo:4.1.1
```

[Read more about the `git:from-image` command](https://dokku.com/docs~v0.26.6/deployment/methods/git/#initializing-an-app-repository-from-a-docker-image) in case of problems.

### Setup Let's Encrypt SSL certificate (optional)

Set up an SSL certificate via Let's Encrypt.

```sh
dokku config:set --no-restart my-matomo DOKKU_LETSENCRYPT_EMAIL=letsencrypt@example.com
dokku letsencrypt my-matomo
dokku letsencrypt:auto-renew my-matomo
```

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

## Upgrading

I haven't had the chance to update successfully since Matomo 4.3.0 launched.
If you find a way, please make a PR!

## Credit

This tutorial was made possible thanks to the following people:

- [CrazyMax](https://github.com/crazy-max)
- [Romain Clement](https://github.com/rclement/dokku-matomo)'s original tutorial
