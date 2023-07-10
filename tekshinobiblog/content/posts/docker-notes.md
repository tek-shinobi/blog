---
title: "Docker Notes"
date: 2021-05-10T08:15:54+03:00
draft: false 
categories: ["docker"]
tags: ["docker"]
---

To cleanup unused/dnagling docker images:  `sudo docker image prune` 

To cleanup ununsed volumes: `sudo docker volume prune`

To list current docker images: `sudo docker images`

To remove docker image by image_id: `sudo docker rmi <your_image_id1> <your_image_id2>`

(sometimes, `docker rmi` command will throw up errors like:
> Error response from daemon: conflict: unable to delete 78f7412b7923 (must be forced) - image is being used by stopped container ebfe07787e7c

This means there are some stopped docker containers using those images. Run this command to find and delete all stopped docker containers: 
```shell
sudo docker rm $(sudo docker ps -q -a)
```
## Best Practices
First run `sudo docker image prune`. This command also frees up any used volumes automatically and tells you how much disk space got freed.

Then run `sudo docker images` . This will show current images. If you see something like this
```
(i)23:46 $ sudo docker images
REPOSITORY                                            TAG           IMAGE ID       CREATED         SIZE
registry.heroku.com/herokudeemo/web                   latest        01c19839a389   2 days ago      91.5MB
rust                                                  latest        9333a9f0c0a6   2 days ago      1.23GB
<none>                                                <none>        78f7412b7923   4 days ago      14.4GB
<none>                                                <none>        b763fba2a7c4   4 days ago      14.4GB
<none>                                                <none>        300133543a17   6 days ago      14.4GB
<none>                                                <none>        a87b7bff727b   7 days ago      14.4GB
<none>                                                <none>        59d8fb4c96c9   7 days ago      14.4GB
postgres                                              12            e782ad565d74   4 weeks ago     314MB
postgres                                              latest        26c8bcd8b719   4 weeks ago     314MB
debian                                                buster-slim   48e774d3c4f5   4 weeks ago     69.3MB
rust                                                  1.49          00150f445191   3 months ago    1.25GB
hello-world                                           latest        bf756fb1ae65   16 months ago   13.3kB

```
Here there are still some images with `REPOSITORY: <none> and TAG:<none>`. These images are supposed to be pruned but could not because there are some stopped docker containers still referring them. So first remove all stopped docker containers: `sudo docker rm $(sudo docker ps -q -a)`

Now run the prune command again to remove dangling images: `sudo docker image prune`

If you run docker images now, you should see all dangling images gone.