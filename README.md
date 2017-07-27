[![License: Apache 2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)
[![Build Status](https://travis-ci.org/LasLabs/docker-runbot.svg?branch=master)](https://travis-ci.org/LasLabs/docker-runbot)

[![](https://images.microbadger.com/badges/image/laslabs/docker-runbot.svg)](https://microbadger.com/images/laslabs/docker-runbot "Get your own image badge on microbadger.com")
[![](https://images.microbadger.com/badges/version/laslabs/docker-runbot.svg)](https://microbadger.com/images/laslabs/docker-runbot "Get your own version badge on microbadger.com")

Docker Runbot
=============

This image provides a fully Dockerized Runbot environment.

Usage
=====

The easiest way to deploy this image is by using Docker Compose.

A simple compose file would look something like the below. This is a great one for testing Runbot locally:

```yml
version: '2'

volumes:
  odoo-db-data:
    driver: local
  odoo-web-data:
    driver: local

services:

  web:
    image: laslabs/runbot:latest
    restart: unless-stopped
    links:
      - postgresql:db
    volumes:
      - odoo-web-data:/var/lib/odoo
      - /var/run/docker.sock:/var/run/docker.sock
    tty: true
    privileged: true
    ports:
      - 10080:8069
      - 1800-2000:1800-2000

    environment:
      PGPASSWORD: 'odoo'
      PGUSER: 'odoo'
      ADMIN_PASSWORD: 'admin'

  postgresql:
    image: postgres:9.6-alpine
    restart: unless-stopped
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_PASSWORD: 'odoo'
      POSTGRES_USER: 'odoo'
    volumes:
      - odoo-db-data:/var/lib/postgresql/data/pgdata
```

In the above example, Runbot is being exposed on port 10080 with no load balancer.
The default Runbot port ranges are also exposed (1800-2000).

The following compose file can be used in order to launch Runbot behind a load balancer
with a host rule for `localhost`. You will need to change it to a proper DNS name if not
testing locally:

```yml
version: '2'

volumes:
  odoo-db-data:
    driver: local
  odoo-web-data:
    driver: local

services:

  web:
    image: laslabs/runbot:latest
    restart: unless-stopped
    links:
      - postgresql:db
    volumes:
      - odoo-web-data:/var/lib/odoo
      - /var/run/docker.sock:/var/run/docker.sock
      - /Users/dlasley/Documents/Repos/oca-runbot-addons/runbot_travis2docker:/opt/odoo/auto/addons/runbot_travis2docker
      - /Users/dlasley/Documents/Repos/oca-runbot-addons/runbot_traefik:/opt/odoo/auto/addons/runbot_traefik
    tty: true
    privileged: true
    environment:
      PGPASSWORD: odoo
      PGUSER: odoo
      ADMIN_PASSWORD: admin
    labels:
      traefik.enable: 'true'
      traefik.port: '8069'
      traefik.frontend.rule: 'Host: localhost;'

  postgresql:
    image: postgres:9.6-alpine
    restart: unless-stopped
    environment:
      PGDATA: '/var/lib/postgresql/data/pgdata'
      POSTGRES_PASSWORD: 'odoo'
      POSTGRES_USER: 'odoo'
    volumes:
      - odoo-db-data:/var/lib/postgresql/data/pgdata

  traefik:
    image: laslabs/runbot-traefik:latest
    stdin_open: true
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    tty: true
    ports:
    - 80:80
    - 8080:8080
    command:
    - --web
```

In the above example, the load balancer is exposing Runbot and all of its builds on port 80.
Port 8080 is the Traefik Web UI, which can be helpful for diagnosing issues.

The following is what a production deploy would look like. Make sure to adjust the `traefik.frontend.rule`
for your environment, as well as change all passwords:

```yml
version: '2'
volumes:
  odoo-web-data:
    driver: rancher-nfs
  odoo-db-data:
    driver: rancher-nfs
  runbot-builds:
    driver: rancher-nfs
  runbot-ssh:
    driver: rancher-nfs
services:
  install:
    image: laslabs/runbot:latest
    command: 'install-addons'
    links:
    - postgresql:db
    volumes:
    - odoo-web-data:/var/lib/odoo
    - /var/run/docker.sock:/var/run/docker.sock
    - runbot-builds:/opt/odoo/custom/src/odoo-extra/runbot/static
    - runbot-ssh:/home/odoo/.ssh
    tty: true
    environment:
      PGPASSWORD: odoo
      PGUSER: odoo
  cron:
    privileged: true
    image: laslabs/runbot:latest
    environment:
      PGPASSWORD: odoo
      PGUSER: odoo
      WAIT_NOHOST: install
    volumes:
    - odoo-web-data:/var/lib/odoo
    - /var/run/docker.sock:/var/run/docker.sock
    - runbot-builds:/opt/odoo/custom/src/odoo-extra/runbot/static
    - runbot-ssh:/home/odoo/.ssh
    tty: true
    links:
    - postgresql:db
    command:
    - /usr/local/bin/odoo
    - --max-cron-threads=1
    - --workers=1
    - --limit-time-real=600
    - --limit-time-cpu=300
  postgresql:
    image: postgres:9.6-alpine
    hostname: db
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_PASSWORD: odoo
      POSTGRES_USER: odoo
    volumes:
    - odoo-db-data:/var/lib/postgresql/data/pgdata
  web:
    image: laslabs/runbot:latest
    environment:
      ADMIN_PASSWORD: admin
      PGPASSWORD: odoo
      PGUSER: odoo
      PROXY_MODE: 'true'
      WAIT_NOHOST: install
    volumes:
    - odoo-web-data:/var/lib/odoo
    - /var/run/docker.sock:/var/run/docker.sock
    - runbot-builds:/opt/odoo/custom/src/odoo-extra/runbot/static
    tty: true
    links:
    - postgresql:db
    ports:
    - 8069
    command:
    - /usr/local/bin/odoo
    - --max-cron-threads=0
    - --workers=4
    - --no-database-list
    - --db-filter=prod
    labels:
      traefik.enable: 'true'
      traefik.port: '8069'
      traefik.frontend.rule: 'Host: runbot.example.com;'
      traefik.frontend.passHostHeader: 'true'
  traefik:
    image: laslabs/runbot-traefik:latest
    stdin_open: true
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    tty: true
    ports:
    - 80:80
    - 8080:8080
    command:
    - --web
  longpolling:
    image: laslabs/runbot:latest
    environment:
      PGPASSWORD: odoo
      PGUSER: odoo
      PROXY_MODE: 'true'
      WAIT_NOHOST: install
    volumes:
    - odoo-web-data:/var/lib/odoo
    - /var/run/docker.sock:/var/run/docker.sock
    - runbot-builds:/opt/odoo/custom/src/odoo-extra/runbot/static
    tty: true
    links:
    - postgresql:db
    ports:
    - 8072
    command:
    - /usr/local/bin/odoo
    - --max-cron-threads=0
    - --workers=2
    labels:
      traefik.enable: 'true'
      traefik.port: '8072'
      traefik.frontend.rule: 'Host: localhost;PathPrefix:/longpolling'
      traefik.frontend.passHostHeader: 'true'
```

The above compose is very similar to the production LasLabs one. It creates the
following services:

* `postgresql` - Database for the Runbot master
* `traefik` - The almighty load balancer
* `install` - Odoo instance that spawns before the other ones.
  Creates the initial database & installs initial addons, then shuts down.
  Can be discarded after initial use.
* `web` - Web facing Runbot container
* `longpolling` - Web facing Runbot longpolling container
* `cron` - This is the Runbot cron worker, and actually performs the builds.
  Scale this for more build capacity.

Environment Variables
=====================

The following environment variables are available for configuration of the
Runbot container:

| Name | Default | Description |
|------|---------|-------------|
| `ADMIN_PASSWORD` | admin | Password for the Runbot database manager |
| `UNACCENT` | true | Search without accented characters |
| `PGUSER` | odoo | Username to database |
| `PGPASSWORD` | odoopassword | Password for the database |
| `PGHOST` | db | Hostname for the database server |
| `PGDATABASE` | prod | Database name to use for Runbot|
| `PROXY_MODE` | false | Set to `true` if Runbot is behind a load balancer |
| `WITHOUT_DEMO` | all | Demo data setting for Runbot |


Known Issues / Roadmap
======================

*

Bug Tracker
===========

Bugs are tracked on [GitHub Issues](https://github.com/LasLabs/docker-runbot/issues).
In case of trouble, please check there to see if your issue has already been reported.
If you spotted it first, help us smash it by providing detailed and welcomed feedback.

Credits
=======

Contributors
------------

* Dave Lasley <dave@laslabs.com>

Maintainer
----------

[![LasLabs Inc.](https://laslabs.com/logo.png)](https://laslabs.com)

This module is maintained by [LasLabs Inc.](https://laslabs.com)

* https://github.com/LasLabs/docker-runbot
