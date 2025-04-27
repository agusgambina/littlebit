---
layout: post
title:  "Setting up MongoDB as a service in Docker Swarm"
date:   2018-09-10 16:00:00 -0300
categories: docker
comments: true
---

> Note: This Post is very similar to the last one "Setting up PostgreSQL database as a service in Docker Swarm". However, there are some differences, for example, how it is called the secret from the compose file.

## Running Services with Docker

This post explains how to set up *MongoDB* as a service in *Docker Swarm*.

This is my stack

* *OSX* *10.13.6*
* *Docker*  *18.06.0*

To enable *Docker Swarm* execute

```
$ docker swarm init
```

## Network

It is required a network where the database service would be connected. To create the network in *Docker* execute

```
$ docker network create --driver overlay mongodb_backend_network
```

## Persistence Layer

It is a good practice to split the database running process and the persistence storage. Because if the running process stops, and it is not recoverable, the data layer would be isolated. Docker allows this using volumes.

Create a folder on the disk for this volume.

```
$ mkdir -p ~/docker/volumes/mongodb
```

## Secrets

We need to create a password for the database user. *Docker Secret* are a great tool for this purpouse. Execute the following command to create the *Secret*

```
$ openssl rand -base64 12 | docker secret create mongodb_password -
```

You should check that the *Secret* is in the service

```
$ docker secret ls
```
## Docker compose file

Create a file `docker-compose.yml` with the following data. Try to understand what is each line for. You would probably need to change the `source` volume for the full path in your OS from the folder you created in the first step.

```
version: '3.7'

services:

  db:
    image: mongo
    secrets:
      - mongodb_password
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      resources:
        reservations:
          memory: 128M
        limits:
          memory: 256M
    ports:
      - 27017:27017
    networks:
      - mongodb_backend_network
    environment:
      MONGO_INITDB_ROOT_USERNAME: 'root'
      MONGO_INITDB_ROOT_PASSWORD_FILE: /run/secrets/mongodb_password
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - type: bind
        source: /Users/username/docker/volumes/mongodb
        target: /data/db

networks:
  mongodb_backend_network:

secrets:
  mongodb_password:
    external: true

```

## Start the service

You should execute the following command. You should be in the same location as the file created in the previous section

```
$ docker stack deploy -c docker-compose.yml mongodb
```

Wait some time until the service is started and all the configuration set up.

### Check service is running

```
$ docker service ls
```

### Check container is running

```
$ docker container ls
```

## Connect to the database

The secret should be on the service

### Check secrets are in the service

```
$ docker exec -it $(docker ps -f name=mongodb_db -q) ls /run/secrets/
```

I like the *Robo 3T* client for *MongoDB*. To connect to the database, as you set up in the file this is the data

```
User Name: root
```

The password you can obtain executing the following comnand

### Obtain the value of the secret
```
$ docker exec -it $(docker ps -f name=mongodb_db -q) cat /run/secrets/mongodb_password
```

## Resources

* [Robo 3T](https://robomongo.org/)

## References

* [Use Docker Secrets With MySQL on Docker Swarm](http://blog.ruanbekker.com/blog/2017/11/23/use-docker-secrets-with-mysql-on-docker-swarm/)
* [Docker Mastery: The Complete Toolset From a Docker Captain - Bret Fisher](https://www.udemy.com/share/1001eQA0UZc1laRQ==/)
