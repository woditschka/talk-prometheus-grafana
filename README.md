# Monitoring with Prometheus and Grafana

Talk @ SoCraTes Canaries 2018

## What is Prometheus

[https://prometheus.io](https://prometheus.io/)

Prometheus is a open source project that provides monitoring and alerting. 
It's main features are:

* multi dimensional time series database  based on metric name and key value pairs
* flexible query language providing aggregation
* collect data via http pull
* static targes and automatic service discovery
* large set of monitoring targets
* allerting


## What is Grafana

[https://grafana.com](https://grafana.com)

Grafana is an open source project that provides metrics visualisation and analysis.
It's main features are:

* powerful metric visualisations
* multiple datasource backends
* allerting

## Monitoring targets demonstrated

* [Node Exporter](https://github.com/prometheus/node_exporter)
* [cadvisor](https://github.com/google/cadvisor)

# Installation on Ubuntu 16.04

On a fresh instance of Ubuntu (e.g. on [https://digitalocean.com](https://digitalocean.com))

## System Preparation

### Update system
```
apt-get -y update
apt-get -y upgrade
```

### Install Firewall
```
ufw status
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80/tcp
ufw --force enable
```

### Increase vm.max_map_count

```
sysctl -w vm.max_map_count=262144
```

### Install Docker

[https://docs.docker.com/installation/ubuntulinux/](https://docs.docker.com/installation/ubuntulinux/)

```
wget -qO- https://get.docker.com/ | sh
```

### Create isolated network for monitoring components
```
docker network create --driver bridge isolated_monitoring
```

## Appliction instalation


### Install application folder structure on host
```
mkdir -p /var/docker/prometheus/config
mkdir -p /var/docker/prometheus/data
chmod -R uga+rwX /var/docker
```

### create prometheus config
```
cat >/var/docker/prometheus/config/prometheus.yml <<'EOL'
global:
  scrape_interval: 10s
  scrape_timeout: 10s
  evaluation_interval: 1m

scrape_configs:
- job_name: prometheus
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - prometheus:9090

- job_name: node
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - node-exporter:9100  
    
- job_name: cadvisor
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - cadvisor:8080    
EOL
```

### Install Prometheus stack on docker
```
docker run -d --restart=always --name node-exporter \
  -p 127.0.0.1:9100:9100 \
  --net="host" \
  --pid="host" \
  --net=isolated_monitoring \
  quay.io/prometheus/node-exporter
  
docker run -d --restart=always --name cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  -p 127.0.0.1:8080:8080 \
  --net=isolated_monitoring \
  --detach=true \
  google/cadvisor:latest  
  
docker run -d --restart=always --name prometheus \
  -v /var/docker/prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /var/docker/prometheus/data:/prometheus \
  -p 127.0.0.1:9090:9090 \
  --net=isolated_monitoring \
  quay.io/prometheus/prometheus

docker run -d --restart=always --name grafana \
  -p 127.0.0.1:3000:3000 \
  --network=isolated_monitoring \
  grafana/grafana
```

```
curl localhost:9100/metrics
```

## Access to UI

Access to the UI must be secures, so for the moment we rely on ssh prot forwarding to secure the connection:

```
ssh -L 9090:localhost:9090 -L 3000:localhost:3000 root@<metrics-host>
``` 

### Prometheus UI

The Prometheus UI can be accessed on [http://localhost:9090](http://localhost:9090). 
It provides simple visualisation of scrape targets, service discovery and metrics data.


### Grafana UI

The Grafana UI can be accessed on [http://localhost:3000](http://localhost:3000), the default login credentials are admin/admin.

#### Datasource

Grafana supports multipe datasources, to access Prometheus data create a datasource with the type 'Prometheus' on the url 'http://prometheus:9090'.

```
Configuration > Data Sources > Add data source

type: Prometheus, url: http://prometheus:9090
``` 

#### Dashbords

You can create your own dashboards or use one of the comunity dasbords from [https://grafana.com/dashboards](https://grafana.com/dashboards).

For The comutity Dasbords select Import Dashbord and use the number of the dashboard form the website:
```
Create > Import Dashboard > enter number, select datasource
```

* Node Exporter Full - 1860  
* Prometheus 2.0 Overview - 3662
* Docker Host & Container Overview - 395
 
