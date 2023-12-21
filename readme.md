# Zapdos

UnownHash metrics guidelines and dashboards collection for Prometheus integration. Provided dashboards are comaptible with VictoriaMetrics (not Prometheus) TSDB. You will find them under project repositories, named as `{repo}-vm-dashboard.json`.

This document assumes that you have basic Linux, Docker and networking knowledge.

Tools Used:
- [Grafana](https://grafana.com/) - interactive visualization web application.
- [VmAgent](https://docs.victoriametrics.com/vmagent.html) - tiny agent which helps you collect metrics from various sources.
- [VictoriaMetrics](https://victoriametrics.com/) - TSDB, _backward compatible with Prometheus_, more memory/storage efficient and has extended queries syntax. Was picked instead Prometheus as I couldn't successfully rewrite queries to limited PromQL.
- [VmCtl (Optional)](https://docs.victoriametrics.com/vmctl.html) - data migration tool which for example allows to import Prometheus metrics.

## Pre-Configuration

For installation process, connect to server SSH and create tunnel to Grafana. Thanks to that you'll be able to access Grafana WebUI locally through a browser.

> VPN or nginx proxy is recommended but out of scope of this guide.

```bash
ssh user@host -L 3001:127.0.0.1:3001
```

Make sure you have installed Docker and Docker Compose.

## Docker Setup

This section explains how we can spin all things using docker compose.

Assuming:
- `~/docker` - docker-compose.yml and data directory.
- `1000:1000` - current user.


### 1. Create Docker-Compose

Create `~/docker/docker-compose.yml` file. Edit it to match your needs.

> It's adviced to enable [auth](https://docs.victoriametrics.com/vmagent.html?highlight=basicauth#kafka-broker-authorization-and-authentication) if you plan to open ports. To do that, simply uncomment auth lines in both services. Setting [data retention](https://docs.victoriametrics.com/#retention) is good idea as well.

```yml
# docker-compose.yml
services:
  # Visualization
  grafana:
    image: grafana/grafana-oss
    container_name: grafana
    user: 1000:1000
    restart: unless-stopped
    ports:
      - 3001:3000
    volumes:
      - ./grafana:/var/lib/grafana

  # Metrics agent
  vmagent:
    image: victoriametrics/vmagent
    container_name: vmagent
    user: 1000:1000
    restart: unless-stopped
    working_dir: /vmagentdata
    depends_on:
      - victoriametrics
    ports:
      - 8429:8429
    volumes:
      - ./vmagent/data:/vmagentdata
      - ./vmagent/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - "--promscrape.config=/etc/prometheus/prometheus.yml"
      - "--remoteWrite.url=http://victoriametrics:8428/api/v1/write"
      # - "--remoteWrite.basicAuth.username=CHANGEME"
      # - "--remoteWrite.basicAuth.password=CHANGEME"

  # TSDB Engine
  victoriametrics:
    image: victoriametrics/victoria-metrics
    container_name: victoriametrics
    user: 1000:1000
    restart: unless-stopped
    ports:
      - 8428:8428
    volumes:
      - ./victoriametrics/data:/storage
    command:
      - "--storageDataPath=/storage"
      - "--httpListenAddr=:8428"
      - "--maxLabelsPerTimeseries=30"
      # - "--remoteWrite.basicAuth.username=CHANGEME"
      # - "--remoteWrite.basicAuth.password=CHANGEME"
      # - "--retentionPeriod=365d"
```

### 2. Create `prometheus.yml` scrape configuration

Inside `~/docker/vmagent/prometheus.yml` file create scrapping configuration.

- Replace targets with matching ones (example assumes you run labelled services through docker)
- Remember to enable prometheus integration in corresponding configurations
- If you added auth in VM configuration, be sure to fill them in there as well.

```yml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: 'unown'
    # basic_auth:
    #   username: CHANGEME
    #   password: CHANGEME
    static_configs:
      - targets: ['golbat:9271']
        labels:
          instance: 'golbat'

      - targets: ['dragonite:7272']
        labels:
          instance: 'dragonite'

      - targets: ['rotom:7072']
        labels:
          instance: 'rotom'
```

### 3. Import Prometheus snapshots (Optional)

In case you've been using Prometheus before (or other supported [TSDB](https://docs.victoriametrics.com/vmctl.html)), you can simply move your metrics data to VictoriaMetrics. 

#### 1. Start victoriametrics TSDB
```bash
docker compose up -d victoriametrics
```

#### 2. Start migration process

- Consider running it inside screen/tmux as this process might take a while!
- It might be wise to stop prometheus before running migration.

```bash
docker run --rm -it \
  --user 1000:1000 \
  --network docker_default \
  -v ~/docker/prometheus/data:/prometheus \
  victoriametrics/vmctl \
  prometheus --prom-snapshot=/prometheus --vm-addr http://victoriametrics:8428
```

### 4. Start VmAgent & Grafana

```bash
docker compose up -d grafana vmagent
```

After executing those commands, you should have running Grafana interface under [127.0.0.1:3001](http://127.0.0.1:3001)

#### 1. Process with initial user configuration

Set a new password for your user and if you want edit your user details under [/profile](http://127.0.0.1:3001/profile)

#### 2. Add new datasource

Under [/connections/datasources/new](http://127.0.0.1:3001/connections/datasources/new) you have to add a new source pointing to VictoriaMetrics TSDB. If you added auth in VM configuration, be sure to fill them in there as well.

From a list of sources pick Prometheus, use `http://victoriametrics:8428` and click `Save & Test`

### 5. Import dashboards

> Dasboards are made for VictoriaMetrics. Loading them in Prometheus will result in not working graphs or incorrect data. Feel free to port them from VM to Prometheus and share.

Go to the [/dashboard/import](http://127.0.0.1:3001/dashboard/import) and load dashboard files.
