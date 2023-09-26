---
title: "Golang:NewProjSteps"
date: 2018-09-24T17:35:09+03:00
draft: false 
categories: ["golang"]
tags: ["tools"]
---

## Database

create and run a docker postgres container in detached mode. Container name is `modelapp-postgres`. The postgres image used is `postgres:latest`:
```shell
docker run --name modelapp-postgres -d --rm -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:latest
```

check logs of running docker container:
```shell
docker logs modelapp-postgres | bat -l man   
```
inspect docker container:
```shell
docker inspect modelapp-postgres | bat -l man
```
login to container as root, to see if postgres is running:
```shell
docker exec -it modelapp-postgres psql -U root
```
further check if postgres running ok and in a ready state by running the `select now();` at root console:
```shell
root=# select now();
              now              
-------------------------------
 2023-09-24 13:14:13.216822+00
(1 row)
```
to create new database, exit out of prev step and then enter postgres container shell like so. Since this is a postgres container's shell, we have access to some postgres cli commands like one for creating a database:
```shell
docker exec -it modelapp-postgres /bin/sh
```

create database named `modelapp`. With owner as root and username as root
```shell
# createdb --username=root --owner=root modelapp
# 
```

connect to the database console like so:
```shell
# psql modelapp
```
the prompt will change to one of database name:
```shell
# psql modelapp
psql (15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

modelapp=# 
```
to quit the database console:
```shell
modelapp=# \q
```

to drop the database (from within the bash shell of container):
```shell
# dropdb modelapp
```

a shortcut to directly goto the database console without going through the container shell is (the database has to exist for this to work):
```shell
docker exec -it modelapp-postgres psql -U root modelapp
```

another shortcut way to directly create a database and drop database using docker command is:
```shell
# create database
docker exec -it modelapp-postgres createdb --username=root --owner=root modelapp
# delete database
docker exec -it modelapp-postgres dropdb --username=root --owner=root modelapp
```
## Golang migrate
create migration schema file:
```shell
./migrate create -ext sql -dir internal/repository/migrations/postgres -seq create_users_table
```