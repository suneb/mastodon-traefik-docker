# Docker Compose: Mastodon with Traefik and ElasticSearch

Notes on setting up a Mastodon instance on an Ubuntu server using Docker Compose. End goal was to have:
* Mastodon running on custom domain
* Traefik for https and LetsEncrypt for automatic certificates
* Optional ElasticSearch for search

Disclaimer: Might've missed a few details here and there, let me know, if you're having problems.

## Prerequisites
* Min. 2 vCPUs, 4 GB ram (8 GB with ElasticSearch), 20 GB storage
* Ubuntu 22 LTS w. Docker [see https://docs.docker.com/engine/install/ubuntu/]
* Open firewall for http (port 80) and https (port 443)
* Domain name (eg. my.domain) where you control DNS records
* Credentials for an outgoing e-mail service, such as SendGrid or MailJet
* Clone of this repository

Please note: Running all commands shown below as `root`

## DNS
Setup for domain name with the following A-records:
* Get public IP: `wget -qO- http://ipecho.net/plain | xargs echo`
* Configure following domains to point to the public ip:
  * `my.domain` 
  * `traefik.my.domain` 
  * `kibana.my.domain` 

Strictly speaking you only need to setup `my.domain` as the others are for debugging and admin purposes. However the following steps assume that all dns records above are set up.

## Traefik
We'll start by getting Traefik up and running. This is our load balancer and public https endpoint.
```
cd traefik
cp .env.template to .env

# install htpasswd for generating hashed auth. data
sudo apt-get install apache2-utils

# generate admin login, copy the output of this command
echo $(htpasswd -nbB admin "mypass") | sed -e s/\\$/\\$\\$/g 

nano .env
```
Change `mypass` above to your own password. Change settings in `.env` to fit your setup:
```
TRAEFIK_DASHBOARD_HOST=traefik.my.domain
TRAEFIK_ADMIN_PASSWORD=(paste admin login here)
TRAEFIK_IP_WHITELIST=127.0.0.1 (ips allowed to access traefik dashboard)
TRAEFIK_LETSENCRYPT_EMAIL=email@my.domain
```
Hit ctrl-s to save and ctrl-x to exit. Then spin up docker:
```
docker compose up -d && docker compose logs -f
```
At this point Traefik should generate `certs/acme.json` containing the first https certificate. You should be able to access the Traefik dashboard at `https://traefik.my.domain/`.

If you have your own security around Traefik (eg. only local access or Wireguard), you can disable the security features by removing the relevant middleware on this line in `docker-compose.yml`:
```
traefik.http.routers.traefik.middlewares=dashboardauth,homeonly@docker
```

## Mastodon
We'll start out by creating a few local users/groups that the docker containers will use to access the local file system:
```
addgroup --gid 991 mastodon
adduser --uid 991 --gid 991 --gecos "" --disabled-password mastodon

cd mastodon
mkdir public
chown mastodon:mastodon public

touch .env.production
chown mastodon:mastodon .env.production
```
Now let's prepare the `.env` that'll be used by `docker-compose.yml`:
```
cp .env.template .env
nano .env
```
Adjust the settings in `.env`:
```
COMPOSE_PROJECT_NAME=mastodon
POSTGRES_PASSWORD=(make a secure password for database)
TRAEFIK_MASTODON_DOMAIN=my.domain (change to your domain)
```
Save and quit the editor.

Okay, let's setup mastodon itself:
```
docker compose run --rm -v $(pwd)/.env.production:/opt/mastodon/.env.production web bundle exec rake mastodon:setup
```
This command will start an interactive setup guide:
```
Your instance is identified by its domain name. Changing it afterward will break things.
Domain name: my.domain

Single user mode disables registrations and redirects the landing page to your public profile.
Do you want to enable single user mode? No

Are you using Docker to run Mastodon? Yes

PostgreSQL host: db
PostgreSQL port: 5432
Name of PostgreSQL database: postgres
Name of PostgreSQL user: postgres
Password of PostgreSQL user: (insert password from .env)
Database configuration works! 🎆

Redis host: redis
Redis port: 6379
Redis password:
Redis configuration works! 🎆

Do you want to store uploaded files on the cloud? No

Do you want to send e-mails from localhost? No
SMTP server: mail.example.com
SMTP port: 587
SMTP username: smtp@my.domain
SMTP password:
SMTP authentication: plain
SMTP OpenSSL verify mode: none
E-mail address to send e-mails "from": Mastodon <notifications@my.domain>
Send a test e-mail with this configuration right now? No

This configuration will be written to .env.production
Save configuration? Yes

Prepare the database now? Yes
Running `RAILS_ENV=production rails db:setup` ...

Database 'postgres' already exists
Done!

All done! You can now power on the Mastodon server 🐘
Do you want to create an admin user straight away? No
```
You can review the settings by opening `.env.production` with nano. Hopefully we're all set, let's start docker:
```
docker compose up -d && docker compose logs -f
```
At some point `mastodon-web-1` wil log that it's listening on `http://0.0.0.0:3000`.

Just need to create an admin user. You can do this in the command line by accessing the container and using Matodons cli:
```
docker compose exec -ti web /bin/bash

# Inside the container:
bin/tootctl accounts create \
  admin \
  --email admin@my.domain \
  --confirmed \
  --role Owner
```
Mastodon will print out your temporary password.

You can check if everything is running by going to `https://my.domain/`. If it doesn't, there's a few ways to debug:
* Run `curl -v http://localhost:3000/health` and see if it returns HTTP 200 OK. If it does, the problem is probably with the connection to Traefik.
  * If not, check the docker logs with `docker compose logs -f`
* Access the Traefik dashboard at `https://traefik.my.domain/` and check if the `mastodon-web@docker` router is linked to both the `:443` entrypoint and `mastodon-web` service

## ElasticSearch
ElasticSearch is (optionally) used by Mastodon to do full-text search, which allows logged in users to find results from their own statuses, their mentions, their favourites, and their bookmarks.

Let's start by getting ElasticSearch set up:
```
# Start fom the root directory of the repo
cd elastic

# Setup user and data directory
mkdir data
addgroup --gid 1000 elastic
adduser --uid 1000 --gid 1000 --gecos "" --disabled-password elastic
chown elastic:elastic data

# Set up .env file
cp .env.template to .env
nano .env
```
Change the settings to your needs:
```
ELASTIC_VERSION=8.5.2
COMPOSE_PROJECT_NAME=elastic
KIBANA_SYSTEM_USER=kibana_system
KIBANA_SYSTEM_PASSWORD=(reset password to get)
KIBANA_ENCRYPTION_KEY=(use any text string that is 32 characters or longer)
KIBANA_PUBLIC_URL=https://kibana.my.domain (change to your domain)
ES_HEALTH_USER=health_user (create user in kibana)
ES_HEALTH_PASSWORD=(generate a password)
TRAEFIK_KIBANA_HOST=kibana.my.domain (change to your domain)
```
Don't worry about the `KIBANA_SYSTEM_PASSWORD` and `ES_HEALTH_PASSWORD`, they can be fixed at a later time.
For now, it should be possible to start ES:
```
docker compose up -d && docker compose logs -f
```
When ElasticSearch is running, reset the passwords for `elastic` and `kibana_system` using these commands:
```
# Reset elastic user password
docker exec -ti elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

# Reset kibana_system password
docker exec -ti elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```
Put the `kibana_system` password into the `KIBANA_SYSTEM_PASSWORD` variable. Re-up the containers:
```
docker compose down
docker compose up && docker compose logs -f
```
When ElasticSearch has started, you should be able to access Kibana on `https://kibana.my.domain`. From here you can create users and roles in Stack Management.

At this point we can enable ElasticSearch support on Mastodon. Go to the `mastodon` directory and edit `docker-compose.yml`. Enable all lines related to `elastic-net`:
```
Remove comments in front of lines, that mention elastic-net: eg.:
- elastic-net
... and:
elastic-net:
  external: true
```
This will allow the Mastodon services to communicate with the ElasticSearch services. Now add the following lines to `.env.production`:
```
ES_ENABLED=true
ES_HOST=elasticsearch
ES_PORT=9200
ES_USER=elastic
ES_PASS=(elastic password we reset earlier)
ES_PREFIX=mastodon
```
I use the `elastic` user the first time, later this can be changed to a less priviliged user. Re-up the mastodon services:
```
docker compose down
docker compose up && docker compose logs -f
```
The only thing left is to initialize the ElasticSearch support in the mastodon service:
```
docker compose exec -ti web /bin/bash

# Inside the container:
bin/tootctl search deploy
```
In Kibana you can check, if the indexes have been created under Stack Management.
You can also disable Kibana to save ressources.

To setup the `ES_HEALTH_USER` in Elastic `.env` go to Kibana, Stack Management to create a new role with the following cluster priviliges: `monitor`. Create a user `health_user` with the new role and configure password in `.env` in `ES_HEALTH_PASSWORD` variable.

## Resources
All journeys start with a search. For me, this led me to the following exellent ressources, which were all very helpful:
*  https://gist.github.com/smashnet/38cf7c30cb06427bab78ae5ab0fd2ae3
* https://github.com/peterrus/docker-mastodon
* https://gitlab.com/dweipert.de/devops/mastodon-docker-traefik

Thanks to these guys for the head start.
