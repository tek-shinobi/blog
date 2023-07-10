---
title: "Using Postgres Inside Docker Container"
date: 2023-07-10T10:38:56+03:00
draft: false 
categories: ["docker"]
tags: ["docker", "postgres"]
---
These are my personal notes. Am running postgres docker image on a local linux box.

If port 5432 is already in use, run this to see what process is using it:
```terminal
sudo ss -lptn 'sport = :5432'
```


the above command will also display the pid using which you can kill the process. For example, if the pid is 1667, kill it by sudo kill 1667

Now sudo launch the shell script that runs the docker image.

## Connecting to the postgres database within the container

First get the pid of the docker process: `sudo docker ps`

Now login to the root of the docker instance: `sudo docker exec -it bdd95244ce47 bash`
(Here `bdd95244ce47` was the pid of the docker process from prev step, change it to whatever the pid is)

Then login to postgres root like so: `psql -U postgres`

then you can list databases like so: `\l`

connect to database: `\c whatever_db_name`

display all tables within connected database: `\dt`