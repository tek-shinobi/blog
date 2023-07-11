---
title: "Using Postgres Inside Docker Container"
date: 2023-04-10T10:38:56+03:00
draft: false 
categories: ["docker"]
tags: ["docker", "postgres", "golang"]
---
## Cleanup if default postgres port not free
If port 5432 is already in use, run this to see what process is using it:
```terminal
sudo ss -lptn 'sport = :5432'
```

the above command will also display the pid using which you can kill the process. For example, if the pid is 1667, kill it by sudo kill 1667

(to connect to an existing running docker container via its container id, scroll to end)

## Connecting to postgres from scratch
First launch the postgres container in interactive mode:
```terminal
docker run \
--name testing-dev \
--rm \
-e POSTGRES_PASSWORD=password \
-p 5432:5432 postgres
```
The above command will download the latest postgres image if not found locally.

Here we `testing-dev` is the container name, `--rm` so as to remove any stopped container with same name before lauching this one. We are launching this in interactive mode, so it will occupy the current terminal.

From another terminal, login to this container's bash shell like so:
```shell
docker exec -it testing-dev bash
```
You will be logged into bash shell as root. You will see something like this:
```terminal
root@e78cf93c2c4e:/# 
```

 Now type this:
```shell
psql -h localhost -U postgres
```
The above will launch you in as default `postgres` super admin user. This is a special default user and you are able to login as this one, without a password, because you are accessing this postgres instance from the same host machine. The prmopt will change to `postgres`
```terminal
root@e78cf93c2c4e:/# psql -h localhost -U postgres
psql (15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

postgres=#
```
In Postgres, `\` indicates a ***metacommand***. You can run postgres meta commands to do some useful things. 

Like:
* `\?` : this meta command will list all available meta commands 
* `\l` : this meta command will list all databases
* `\dt` : this meta command will list all tables
* `\du` : this meta command will list all users

create a database named `null_vals` like so:
```shell
 create database null_vals;
```

in the shell, when you will do `\l`, you will now see something like this:
```terminal
postgres=# create database null_vals;
CREATE DATABASE
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider 
|   Access privileges   
-----------+----------+----------+------------+------------+------------+-----------------
+-----------------------
 null_vals | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            
| 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            
| 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            
| =c/postgres          +
           |          |          |            |            |            |                 
| postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            
| =c/postgres          +
           |          |          |            |            |            |                 
| postgres=CTc/postgres
(4 rows)
```

now switch to `null_vals` database like so:
```shell
\c null_vals
```
in terminal, you will see something like this:
```terminal
postgres=# \c null_vals
You are now connected to database "null_vals" as user "postgres".
null_vals=#
```
The prompt has now changed to your database name `null_vals`

Now we need to create a user and password, so we can start interacting from Golang.

So we will setup user `anshu` with password `password` like so:
```shell
CREATE ROLE anshu WITH LOGIN PASSWORD 'password';
```
You will see something like so in terminal:
```terminal
null_vals=# CREATE ROLE anshu WITH LOGIN PASSWORD 'password';
CREATE ROLE
null_vals=#
```
Now you need to also grant permissions to user `anshu` before you are able to use it. This step was optional till postgres version 14. From postgres 15 onwards, this step is mandatory.

Grant permissions like so:
```shell
grant all on schema public to anshu;
```
You will see on terminal like so:
```terminal
null_vals=# grant all on schema public to anshu;
GRANT
```

Now you can connect to `null_vals` database as user `anshu` like so:
```shell
\c null_vals anshu
``` 
in terminal, it will look like so:
```terminal
null_vals=# \c null_vals anshu
You are now connected to database "null_vals" as user "anshu".
```

Above, you didn't exit from the existing shell. You just switched from default superuser `postgres` to user `anshu`

If you want, you can exit the current postgres shell and login with username and password creadentials like so:
```terminal
null_vals=# exit
root@e78cf93c2c4e:/#
```

Now from root, login to database `null_vals` with your creds:
```shell
psql --host localhost --dbname=null_vals --username=anshu
```
it might ask you for password too.
```terminal
root@e78cf93c2c4e:/# psql --host localhost --dbname=null_vals --username=anshu
psql (15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

null_vals=>
```

That's it. You are all set.

Let's create a database table while we are at it.
```shell
create table if not exists mytable (
id bigserial primary key,
title text null,
version integer null 
);
```
it will look like so in terminal
```terminal
null_vals=> create table if not exists mytable (
id bigserial primary key,
title text null,
version integer null default 1
);
CREATE TABLE
```
and now list tables
```terminal
null_vals=> \dt
        List of relations
 Schema |  Name   | Type  | Owner 
--------+---------+-------+-------
 public | mytable | table | anshu
(1 row)

null_vals=>
```

Now we are all set. Creating table was not necessary from postgres shell. It can be done via migrations from within Golang.

Here is how you connect to this database from Golang:
```go
package main

import (
	"context"
	"database/sql"
	"flag"
	"log"
	"os"
	"time"

	_ "github.com/lib/pq"
)

type config struct {
	db struct {
		dsn string
	}
}

func main() {
	var cfg config
	dsn := "postgres://anshu:password@localhost:5432/null_vals?sslmode=disable"
	flag.StringVar(&cfg.db.dsn, "db-dsn", dsn, "PostgreSQL DSN")
	flag.Parse()
	logger := log.New(os.Stdout, "", log.Ldate|log.Ltime)
	db, err := openDB(cfg)
	if err != nil {
		logger.Fatal(err)
	}

	defer db.Close()

	logger.Printf("database connection pool established\n")
}

func openDB(cfg config) (*sql.DB, error) {
	db, err := sql.Open("postgres", cfg.db.dsn)
	if err != nil {
		return nil, err
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	err = db.PingContext(ctx)
	if err != nil {
		return nil, err
	}

	return db, nil
}
```

## Connecting to the postgres database within the container, via container instance id

First get the pid of the docker process: `sudo docker ps`

Now login to the root of the docker instance: `sudo docker exec -it bdd95244ce47 bash`
(Here `bdd95244ce47` was the pid of the docker process from prev step, change it to whatever the pid is)

Then login to postgres root like so: `psql -U postgres`

then you can list databases like so: `\l`

connect to database: `\c whatever_db_name`

display all tables within connected database: `\dt`