# Mastadon with Traefik and ElasticSearch

My notes on setting up a Mastadon instance on an Ubuntu server. End goal is to have:
* Mastadon running on custom domain
* Traefik for https and LetsEncrypt for certificate
* Optional ElasticSearch and Kibana for search

## Prerequisites
* Min. 2 vCPUs, 4 GB ram (8 GB with ElasticSearch), 20 GB storage
* Ubuntu 22 LTS
* Clone of this repository

## Traefik
We'll start by getting Traefik up and running. This is our load balancer and public https endpoint.
```

```

## Mastadon
```

```

## ElasticSearch
```
cd elastic

# Setup data directory
mkdir data
addgroup --gid 1000 elastic
adduser --uid 1000 --gid 1000 --gecos "" --disabled-password elastic
chown elastic:elastic data

# If needed, reset elastic user password
docker exec -ti elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

# If needed, reset kibana_system password
docker exec -ti elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system

docker compose exec -ti web /bin/bash
```

