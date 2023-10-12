---
title: "Grafana:Grafana Loki AWS S3 Integration"
date: 2023-10-12T21:14:50+03:00
draft: false 
categories: ["grafana"]
tags: ["loki", "s3", "aws"]
---

Some thoughts and experiences when setting up AWS S3 and single-store for Grafana-Loki. 
Some context: 
- using GLP stack (grafana, loki, promtail)
- Promtail collects logs from a Kafka stream and sends them to Loki for storage. It also indexes them by EC2 instance ids.


## Why this post?
1. Persisting loki logs somewhere is important otherwise, everytime loki service is restarted, all previous logs and corresponding visualizations on grafana dashboard will be lost.
1. The official docs from Grafana are at times confusing. Also, I tried doing it in 3 ways, with different weirdness. Documenting here for future reference.

## AWS S3 configuration
Create a IAM user (NOT a root user). Create a group with policy `AmazonS3FullAccess` and assign the user to this group (Grafana recommends setting specific policies, that is fine as well).

## Docker container approach
Here is how the promtail config looks (`promtail-config.docker.yaml`):
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: amskafka
    kafka:
      brokers:
        - 172.31.36.174:9092
      topics:
        - ams-instance-stats
      labels:
        job: amskafka
    pipeline_stages:
      - match:
          selector: '{job="amskafka"}'
          stages:
            - json:
                expressions:
                  instanceId: instanceId
            - labels:
                instanceId:

  - job_name: amskafkawebrtc
    kafka:
      brokers:
        - 172.31.36.174:9092
      topics:
        - ams-webrtc-stats
      labels:
        job: amskafkawebrtc
```

here, I am setting two kafka streams as sources for promtail. This being yaml, and an misdocumented feature of Grafana's promtail documentaion, pay special attention to spacing of these two line:
```shell
      - match:
          selector: '{job="amskafka"}'
```
here selector is child of match. At the time of writing this blog, grafana official examples show it without the two spaces (`selector` is two spaces to right of `match`). This is important!. If you copy pasted grafana official promtail example for pipeline_stages, you will get a very weird non-obvious error ` pipeline stage must contain only one key`. What this means is, pipeline_stages should have one "SINGLE ELEMENT" array as child, which is `match`. If the two spaces were missing, yaml would treat `selector` as sibling of `match` (no longer a single element array) and you will get this error.
Here is a stack overflow post with fellow strugglers... https://stackoverflow.com/questions/65032914/promtail-error-pipeline-stage-must-only-contain-one-key

pipeline_stage is needed to parse and index the log line. Here is an example log line that is being indexed by "instanceId":
```json
{"instanceId":"14524720-6c65-49a8-aca4-a9d14691a235","cpuUsage":{"processCPUTime":368560000,"systemCPULoad":0,"processCPULoad":0},"jvmMemoryUsage":{"maxMemory":973078528,"totalMemory":681574400,"freeMemory":464074192,"inUseMemory":217500208},"systemInfo":{"osName":"Linux","osArch":"amd64","javaVersion":"11","processorCount":2},"systemMemoryInfo":{"virtualMemory":3874603008,"totalMemory":3892285440,"freeMemory":1666928640,"inUseMemory":2225356800,"totalSwapSpace":0,"freeSwapSpace":0,"inUseSwapSpace":0,"availableMemory":2378375168},"fileSystemInfo":{"usableSpace":5310873600,"totalSpace":10213466112,"freeSpace":5327650816,"inUseSpace":4885815296},"jvmNativeMemoryUsage":{"inUseMemory":1024630784,"maxMemory":3892314112},"gpuUsageInfo":[],"ffmpegBuildInfo":"--prefix\u003d.. --disable-iconv --disable-opencl --disable-sdl2 --disable-bzlib --disable-lzma --disable-linux-perf --disable-xlib --disable-decoder\u003dh264_crystalhd --disable-decoder\u003dmpeg2_crystalhd --disable-decoder\u003dvc1_crystalhd --disable-decoder\u003dmpeg4_crystalhd --disable-decoder\u003dmsmpeg4_crystalhd --disable-decoder\u003dmsmpeg4_crystalhd --disable-decoder\u003dwmv3_crystalhd --enable-shared --enable-version3 --enable-runtime-cpudetect --enable-zlib --enable-libmp3lame --enable-libspeex --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libvo-amrwbenc --enable-openssl --enable-libopenh264 --enable-libvpx --enable-libfreetype --enable-libopus --enable-libxml2 --enable-libsrt --enable-libwebp --enable-libmfx --enable-libnpp --enable-nonfree --extra-cflags\u003d-I/usr/local/cuda/include --extra-ldflags\u003d-L/usr/local/cuda/lib64 --enable-cuda --enable-cuvid --enable-nvenc --enable-pthreads --enable-libxcb --cc\u003d\u0027gcc -m64\u0027 --extra-cflags\u003d\u0027-I../include/ -I../include/libxml2 \u0027 --extra-ldflags\u003d-L../lib/ --extra-libs\u003d\u0027-lstdc++ -lpthread -ldl -lz -lm -lva-drm -lva-x11 -lva\u0027","encoders-blocked":0,"encoders-not-opened":0,"publish-timeout-errors":0,"server-timing":{"up-time":42343897,"start-time":1697101393013},"time":"2023-10-12T20:48:56.528253Z","host-address":"198.44.37.257","ip-address":"4.345.51.44"}

```
Another point to notice is this line:
```shell
- url: http://loki:3100/loki/api/v1/push
```
notice the `loki:3100` instead of `localhost:3100`. This is important as in coming commands you will see that `loki` is the network name connecting the two containers (one running loki and another promtail).

Here is the `loki-config.yaml`
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 200

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
    shared_store: s3
  aws:
    s3: s3://ACCESS_KEY:SECRET_ACCESS_KEY@eu-central-1/bucket-name
    s3forcepathstyle: true

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: s3

chunk_store_config:
  max_look_back_period: 48h

limits_config:
  retention_period: 48h

table_manager:
  retention_deletes_enabled: true
  retention_period: 48h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
analytics:
 reporting_enabled: false
```
some grafana undocumented weirdness here. Note these lines:
```shell
  aws:
    s3: s3://ACCESS_KEY:SECRET_ACCESS_KEY@eu-central-1/bucket-name
    s3forcepathstyle: true
```
the uri style used here for specifying s3 bucket creddentials is called `s3forcepathstyle` which in recent versions of grafana is default set to `false`. Make sure it is set to `true`.

More grafana weirdness:. Note this line

```shell
common:
  instance_addr: 127.0.0.1
  path_prefix: loki
```

The path prefix is "loki" (NO preceding `/` like `/loki`). Then rest of the paths have `/loki` prefixed. I noticed that if I added a `/` to "loki" in path_prefix, nothing is exported to S3. Why.. don't know, grafana does not mention why. It will become weirder when I describe the systemctl way of launching the same loki-config. Then reverse is true.

Also, another undocumented feature... 
```shell
compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: s3
```
this part is necessary.

that's it. Now launch it via docker container like so. First lauch loki container and then promtail container.

launch loki:
```shell
docker run -d --name loki --restart unless-stopped -v $(pwd):/mnt/config -v lokilogs:/loki -p 3100:3100 grafana/loki:2.9.1 -config.file=/mnt/config/loki-config.yaml
```
note the volume mapping `-v lokilogs:/loki` . Here I am mapping a directory called `lokilogs` in the current directory to `/loki` within the container. This mapping is critical to enable loki to send docs to S3. If this is missing, no logs are going to be sent.

launch promtail:
```shell
docker run --name promtail -d --restart unless-stopped -v $(pwd):/mnt/config -v /var/log:/var/log --link loki grafana/promtail:2.9.1 -config.file=/mnt/config/promtail-config.docker.yaml
```

After some time, you will see two directories created in S3 bucket, `index` and `fake`. Here, approx every 2 hours logs are going to be stored.

## `systemctl` way:

You can download loki and promtail linux binaries from Grafana. There are slight variations in loki and promtail configs:

`promtail-config.yaml`
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: amskafka
    kafka:
      brokers:
        - 172.31.36.174:9092
      topics:
        - ams-instance-stats
      labels:
        job: amskafka
    pipeline_stages:
      - match:
          selector: '{job="amskafka"}'
          stages:
            - json:
                expressions:
                  instanceId: instanceId
            - labels:
                instanceId:

  - job_name: amskafkawebrtc
    kafka:
      brokers:
        - 172.31.36.174:9092
      topics:
        - ams-webrtc-stats
      labels:
        job: amskafkawebrtc
```

As mentioned before, the difference is in this line:
```shell
clients:
  - url: http://localhost:3100/loki/api/v1/push
```
it has `localhost`. This is because that refers to `loki` application which is listening on localhost in this case.

Here is the `loki-config.yaml`:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 200

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
    shared_store: s3
  aws:
    s3: s3://ACCESS_KEY:SECRET_ACCESS_KEY@eu-central-1/bucket-name
    s3forcepathstyle: true

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: s3

chunk_store_config:
  max_look_back_period: 48h

limits_config:
  retention_period: 48h

table_manager:
  retention_deletes_enabled: true
  retention_period: 48h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
analytics:
 reporting_enabled: false
```

again, note the `/loki` in path prefix. You need it that way. `loki` will not work here. Of course, no errors will happen and you will see everything fine on grafana dashboard. Just that nothing will be sent to s3.

You can first try locally like so:
First launch loki:
```shell
sudo ./loki-linux-amd64 -config.file=loki-config.yaml
```
notice the `sudo` there. `/loki` is a sudo path. Since this is where this config yaml tells loki to store its logs on the filesystem, you will need sudo access.

Now launch promtail:
```shell
sudo ./promtail-linux-amd64 -config.file=promtail-config.yaml
```
Since we are storing `positionas.yaml` (an internally created promtail file) in `/tmp`, make it sudo access.

Once you can see everything working fine, you can easily convert them to systemctl service like so:

`loki.service`
```shell
[Unit]
Description=Loki Service
After=network.target

[Service]
Environment="CFGFILE=/home/ubuntu/downloads/loki-config.yaml"
ExecStart=/home/ubuntu/downloads/loki-linux-amd64 -config.file=${CFGFILE}
StandardOutput=file:/var/log/loki-systemd.log

[Install]
WantedBy=multi-user.target
```

`promtail.service`
```shell
[Unit]
Description=Promtail Service
After=network.target loki.service

[Service]
Environment="CFGFILE=/home/ubuntu/downloads/promtail-config.yaml"
ExecStart=/home/ubuntu/downloads/promtail-linux-amd64 -config.file=${CFGFILE}
StandardOutput=file:/var/log/promtail-systemd.log

[Install]
WantedBy=multi-user.target
```

save both `loki.service` and `promtail.service` in `/etc/systemd/system`

and launch via
`sudo systemctl start loki` and `sudo systemctl start promtail`

you can tail their logs at files pointed to in `StandardOutput` setting

## Only persist loki logs on local filesystem
Not really recommended, since if running on an EC2 instance and that instance dies for some reason, all locally stored logs will be lost. Also, makes sense to centralize the log storage when multiple instances are involved, only possible with something like S3.

Note, we will be taking the docker container approach, launching both loki and promatail in respective docker containers and connection them via network.
so, here is the `loki-config.yaml`:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb:
    directory: /loki/index

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  retention_enabled: true

chunk_store_config:
  max_look_back_period: 48h

limits_config:
  retention_period: 48h

table_manager:
  retention_deletes_enabled: true
  retention_period: 48h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
analytics:
 reporting_enabled: false
                                
```

Use `promtail-config.docker.yaml` from above for promtail config.

Launch commands are exactly same as the one shown for docker case. Just that, in this case, no export to S3 will happen and all records will be stored into the mounted directory, `lokilogs`.