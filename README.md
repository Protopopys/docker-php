# docker-php

Deploys very small php-fpm images based on alpine 3.7 to your machine using docker-compose.

One Dockerfile for all.

## Pre-requisites

* `docker`
* `docker-compose`

## How to build and run

1. `$ cd compose/`

1. Modify environment variables at `.env` to feed your needs

1. Build images:

    ###### php 5.3 only

    `$ docker-compose build php5.3-fpm`
    
    ###### php 5.6 only

    `$ docker-compose build php5.6-fpm`
    
    ###### php 7.0 only

    `$ docker-compose build php7.0-fpm`
    
    ###### php 7.1 only

    `$ docker-compose build php7.1-fpm`
    
    ###### php 7.2 only

    `$ docker-compose build php7.2-fpm`
    
    ###### ALL

    `$ docker-compose build`
  
Available on [Docker Hub](https://hub.docker.com/r/protopopys/php/)
