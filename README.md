# TP 01

# Docker

---

## Lancement du container psql

### Container postgres

`docker pull postgres:11.6-alpine`

creation du Dockerfile

`docker build -t psql .`

`docker run -p 5432:5432 --network app-network --name postgres psql -d -it`

### Acces avec adminer

`docker network create app-network`

`docker pull adminer`

`docker run -p 8080:8080 --name adminer --network app-network adminer`

### Ajout des scripts sql

creation des scripts sql dans le répertoire du Dockerfile

ajouter les lignes suivantes au Dockerfile :

`COPY 01-CreateSchema.sql /docker-entrypoint-initdb.d/`

`COPY 02-InsertData.sql /docker-entrypoint-initdb.d/`

rebuild le container

#### persistance des données

`docker run -p 5432:5432 --network app-network -v ~/psqldata/:/var/lib/postgresql/data --name postgres psql -d -it`

## Lancement du container Java

#### Hello world

`docker run --name java-hello-world openjdk -d -it`

#### simple api

`docker run -p 8080:8080 --network app-network --name simple-api openjdk -d -it`

#### reverse proxy

`docker run -dit --network app-network --name my-apache-server -p 80:80 my-apache`

# Docker compose

---

```
version: '3.7'
services:
  backend:
    container_name:
      simple-api
    build:
      ./java-api/simple-api/
    networks:
      - my-app-network
    depends_on:
      - database
  database:
    container_name:
      postgres
    build:
      ./psql
    networks:
      - my-app-network
  httpd:
    container_name:
      my-apache-server
    build:
      ./apache
    ports:
      - "80:80"
    networks:
      - my-app-network
    depends_on:
      - backend
      - database
networks:
  my-app-network:
```
