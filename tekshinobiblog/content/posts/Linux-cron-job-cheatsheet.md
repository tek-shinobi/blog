---
title: "Linux Cron Job Cheatsheet"
date: 2020-02-23T21:09:35+02:00
draft: false 
---

the list of all cron jobs for a user:
```sh
crontab -l
```

to setup a new cronjob for a user:
```sh
crontab -e
```
choose the editor and add the cron job at the end of the file.

to setup a cron job for a specific user:
```sh
crontab -u username -e
```

to remove all cron jobs for a user:
```sh
crontab -r
```

cronjob has 6 fields:
```
* * * * * command
```
the `command` is the command to be executed. 

The 5 `*`s are the time fields. They are:
- minute (0-59)
- hour (0-23)
- day of month (1-31)
- month (1-12)
- day of week (0-6) (0 is Sunday)
- command (the command to be executed)

to run a command every minute:
```sh
* * * * * command
```

for example:
```sh
* * * * * /usr/bin/echo "hello"
```

to run a command hourly:
```sh
@hourly command
```

to run a command every time the system boots:
```sh
@reboot command
```

