# cloud-monitoring
Fluentd, Prometheus, Loki, Grafana, and more.

```bash
# batch.sh will hold a bunch of commands to run in one go.
touch ~/batch.sh && chmod +x ~/batch.sh
```

## Docker
```bash
# install docker
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo apt install -y curl gnupg software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli
sudo usermod -a -G docker $(whoami)
EOF
~/batch.sh

# configure docker
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo tee /etc/docker/daemon.json <<EOFF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOFF
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
EOF
~/batch.sh
```
## Ubuntu
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/ubuntu/build
tee ~/docker/ubuntu/build/Dockerfile <<EOFF
FROM ubuntu:20.04
RUN addgroup --gid $(id -g ubuntu) ubuntu \
&& addgroup --gid $(ls -ldn /var/run/docker.sock | awk '{print $4}') docker \
&& adduser --disabled-password --gecos "" --gid $(id -g ubuntu) --uid $(id -u ubuntu) ubuntu \
&& adduser ubuntu docker \
&& apt update \
&& apt install -y sqlite3 \
&& ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
USER ubuntu:ubuntu
WORKDIR /home/ubuntu/nifty
EOFF
cd ~/docker/ubuntu/build
docker build -t ubuntu:latest .
cd ~
docker run -it --name focal --network=host --rm -v ~:/home/ubuntu ubuntu:latest sqlite3 --version
EOF
```
Sample log message:
```bash
docker  run \
        --log-driver=fluentd \
        --log-opt fluentd-address=127.0.0.1:24224 \
        --log-opt tag="{{.Name}}" \
        --log-opt mode=non-blocking \
        --log-opt max-buffer-size=1k \
        --name focal \
        --network="host" \
        --rm \
        -v ~:/home/ubuntu ubuntu echo 'Hello World!' $(date)
```

## Fluentd
See
- [Docker Hub](https://hub.docker.com/r/fluent/fluentd/)
- [Fluentd Configuration](https://docs.fluentd.org/configuration/config-file)
- [Fluentd-Loki](https://grafana.com/docs/loki/latest/clients/fluentd/)
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/fluentd/build
mkdir -p ~/docker/fluentd/config
mkdir -p ~/docker/fluentd/log
tee ~/docker/fluentd/build/Dockerfile <<EOFF
FROM fluent/fluentd:v1.15.0-1.0
USER root
RUN fluent-gem install fluent-plugin-grafana-loki && fluent-gem install fluent-plugin-prometheus
USER fluent
EOFF
cd ~/docker/fluentd/build
docker build -t fluentd:latest .
cd ~
tee ~/docker/fluentd/config/fluentd.config <<EOFF
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<source>
  @type tail
  path /var/log/fluentd.log
  tag *
  <parse>
    @type none
  </parse>
</source>
<match **>
  @type copy
  <store>
    @type loki
    url "http://127.0.0.1:3100"
    extra_labels {"logger":"fluentd"}
    flush_interval 2s
    flush_at_shutdown true
    buffer_chunk_limit 1k
    insecure_tls true
  </store>
  <store>
    @type stdout
  </store>
</match>
EOFF
tee ~/docker/fluentd/run.sh <<EOFF
docker  run \\
        -d \\
        --name fluentd \\
        --network=host \\
        --rm \\
        --user $(id -u):$(id -g) \\
        -v ~/docker/fluentd/config:/fluentd/etc \\
        -v ~/nifty/console.log:/var/log/fluentd.log \\
        -v ~/docker/fluentd/log:/fluentd/log \\
        fluentd:latest -c /fluentd/etc/fluentd.config
EOFF
chmod +x ~/docker/fluentd/run.sh
~/docker/fluentd/run.sh
EOF
~/batch.sh
```

## Prometheus
- [Loki Configuration](https://grafana.com/docs/loki/latest/configuration/)
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/prometheus/config
tee ~/docker/prometheus/config/prometheus.yml <<EOFF
EOFF
docker pull prom/prometheus:v2.38.0
docker tag prom/prometheus:v2.38.0 prometheus:latest
docker run --it --name prometheus --network=host --rm prometheus:latest
```
Open `localhost:9090`.

## Loki
See
- [Loki Configuration](https://grafana.com/docs/loki/latest/configuration/)
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/loki/config
mkdir -p ~/docker/loki/data
tee ~/docker/loki/config/local-config.yaml <<EOFF
auth_enabled: false
server:
  http_listen_port: 3100
common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory
schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
ruler:
  alertmanager_url: http://localhost:9093
EOFF
docker pull grafana/loki:latest
docker tag grafana/loki:latest loki:latest
tee ~/docker/loki/run.sh <<EOFF
docker  run \\
        -d \\
        --name=loki \\
        --network=host \\
        --rm \\
        --user $(id -u):$(id -g) \\
        -v ~/docker/loki/config/local-config.yaml:/etc/loki/local-config.yaml \\
        -v ~/docker/loki/data:/loki \\
        loki:latest
EOFF
chmod +x ~/docker/loki/run.sh
~/docker/loki/run.sh
sleep 10s
curl http://localhost:3100/ready
EOF
~/batch.sh
```

```bash
# Log using HTTP API
EPOCHNANO=$(date +%s)000000000
echo $EPOCHNANO
curl  -v \
      -H "Content-Type: application/json" \
      -X POST -s "http://localhost:3100/loki/api/v1/push" \
      --data-raw '{"streams": [{ "stream": { "logger": "shell" }, "values": [ [ "1662950083000000000", "This is a test" ] ] }]}'
```

## cAdvisor
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
docker pull gcr.io/cadvisor/cadvisor:v0.45.0
docker tag gcr.io/cadvisor/cadvisor:v0.45.0 cadvisor:latest
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  cadvisor:latest
EOF
```

## Grafana

See, (Nginx Reverse Proxy)[https://grafana.com/tutorials/run-grafana-behind-a-proxy/]
```bash
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/grafana/build
mkdir -p ~/docker/grafana/lib
mkdir -p ~/docker/grafana/log
mkdir -p ~/docker/grafana/provisioning
mkdir -p ~/docker/grafana/provisioning/datasources
mkdir -p ~/docker/grafana/provisioning/plugins
mkdir -p ~/docker/grafana/provisioning/notifiers
mkdir -p ~/docker/grafana/provisioning/alerting
mkdir -p ~/docker/grafana/provisioning/dashboards
docker pull grafana/grafana:latest
docker tag grafana/grafana:latest grafana:latest
tee ~/docker/grafana/run.sh <<EOFF
docker  run \\
        -d \\
        --name=grafana \\
        --network=host \\
        --rm \\
        --user $(id -u):$(id -g) \\
        -v ~/docker/grafana/lib:/var/lib/grafana \\
        -v ~/docker/grafana/log:/var/log/grafana \\
        -v ~/docker/grafana/provisioning:/etc/grafana/provisioning \\
        -v ~/docker/grafana/provisioning/datasources:/etc/grafana/provisioning/datasources \\
        -v ~/docker/grafana/provisioning/plugins:/etc/grafana/provisioning/plugins \\
        -v ~/docker/grafana/provisioning/notifiers:/etc/grafana/provisioning/notifiers \\
        -v ~/docker/grafana/provisioning/alerting:/etc/grafana/provisioning/alerting \\
        -v ~/docker/grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards \\
        grafana:latest
EOFF
chmod +x ~/docker/grafana/run.sh
~/docker/grafana/run.sh
EOF
~/batch.sh
```
```bash
GF_PATHS_DATA=/var/lib/grafana
GF_PATHS_LOGS=/var/log/grafana
GF_PATHS_PLUGINS=/var/lib/grafana/plugins
GF_PATHS_PROVISIONING=/etc/grafana/provisioning
```

## Note(s)
```bash
tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
mkdir -p ~/docker/portainer
tee ~/docker/portainer/run.sh<<EOFF
#!/bin/bash
docker run \
-d \
--name portainer \
--network host \
--rm \
-v /var/run/docker.sock:/var/run/docker.sock \
portainer:latest
EOFF
chmod +x ~/docker/portainer/run.sh
EOF
# https://eduroll.eu/nginx-reverse-proxy-for-docker-containers-and-for-portainer/
# 
```