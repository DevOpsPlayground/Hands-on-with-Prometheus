# DevOps Playground #18: Hands on Prometheus

<img src='assets/prometheus_logo_orange.png'>

## Introduction

On this Playground we're going to install Prometheus on a server, alongside with a fake application that produces metrics, and a service that collects system metrics. We'll then explore Prometheus' UI and learn the basic query types.

**Name:** Daniel Meszaros
**Role:** DevOps And Continuous Delivery Consultant
**Email:** daniel.meszaros@ecs-digital.co.uk
**Linkedin:** https://www.linkedin.com/in/meszarosdaniel/

## Requirements

* Modern browser with JS enabled or Terminal or Putty
* Minimal knowledge of how an ssh terminal works.

## Setup

* Open the web terminal (web terminal: http://<your_hostname>.devopsplayground.com)
* Your Actual host name and user/pass are assigned at the start of the presentation.
* Log in to your instance.
* You are set.

## Deployment

### 1. Install prometheus

Prometheus is preinstalled on your machine, but you can easily install it with the following commands:

```bash
cd ~
mkdir prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.2.0-rc.0/prometheus-2.2.0-rc.0.linux-amd64.tar.gz
tar zxvf prometheus-2.2.0-rc.0.linux-amd64.tar.gz -C ./prometheus --strip-components 1
rm -f prometheus-2.2.0-rc.0.linux-amd64.tar.gz
```

### 2. First start of prometheus:

Let's explore prometheus on it's on first. Start the service, and check if the UI works.

```bash
cd prometheus
./prometheus
```

Once you've started the process, please in a new browser tab, open http://<your_hostname>.devopsplayground.com:9090

Explore the UI:

* **Status/Targets.** This is where you can see all your configured target services you're monitoring, and their state. At first you'll  only see one target, which is prometheus itself.
* **Graph.** This is where we'll send queries to our Prometheus instance, and can see the results in text and in graph formats. Note: This is mainly for exploring, not for building dashboards.
* **Alerts.** Where you'll see the triggered alerts.

### 3. `CTRL+C` the current process, and see the contents of the config file called `playground.yml`:

```bash
cat playground.yml
```

```text
global:
  scrape_interval:     15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'fake_app'
    static_configs:
      - targets: ['localhost:8081','localhost:8082']
        labels:
          env: prod
      - targets: ['localhost:8083']
        labels:
          env: dev

  - job_name: 'node_stats'
    static_configs:
      - targets: ['localhost:9100']
```

Explanation:
We'll use 4 targets in our playground. One will be a service that monitors the server's health, and 3 targets of the same fake application, created by Julius. These targets will allow us to explore the querying capabilities of Prometheus.

### 4. Start prometheus in the background with the config file

```bash
nohup ./prometheus --web.enable-lifecycle --config.file=playground.yml &
```

Once started, check out the UI again, and check out the targets.

### 5. Installing the fake app service

It's already installed on your instance, but if there's any issue, you can reinstall it with the following commands

```bash
cd ~
mkdir fake_app
wget https://github.com/juliusv/prometheus_demo_service/releases/download/0.0.4/prometheus_demo_service-0.0.4.linux.amd64.tar.gz
tar zxvf prometheus_demo_service-0.0.4.linux.amd64.tar.gz -C ./fake_app
rm -f prometheus_demo_service-0.0.4.linux.amd64.tar.gz
```

After installing, let's start the fake services:

```bash
fake_app/prometheus_demo_service -listen-address :8081 &
fake_app/prometheus_demo_service -listen-address :8082 &
fake_app/prometheus_demo_service -listen-address :8083 &
```

Once started check out the demo service in a browser, to see how an exporter works, and the data looks like. Open a new tab for user{1..100}.playground.com:8081/metrics

### 6. Installing `node_exporter`

Node Exporter is the official system metrics engine for Prometheus. It's already installed on your system, but in case any problems, here's the commands to install it:

```bash
cd ~
mkdir node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-amd64.tar.gz
tar zxvf node_exporter-0.15.2.linux-amd64.tar.gz -C node_exporter --strip-components 1
rm -f node_exporter-0.15.2.linux-amd64.tar.gz
```

Once installed, start the service:

```bash
./node_exporter &
```

### 7. Querying

* if it's not open, please go to `http://<your_user>.devopsplayground.com:9090`
* Please see the queries below for for your reference:
  * `http_requests_total`
  * `http_requests_total{job='fake_app'}`
  * `http_requests_total{job='fake_app',env='prod'}`
  * `rate(http_requests_total{job='fake_app',env='prod'}[5m])`
  * `node_memory_MemFree`
  * `node_memory_MemFree / 1024`
  * `(node_memory_MemFree / node_memory_MemTotal) * 100`
  * `demo_api_request_duration_seconds_count`
  * `demo_api_request_duration_seconds_count{path=~'.*bar',status!='200'}`
  * `sum(rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m]))`
  * `sum by(status) (rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m]))`
  * `sum by(status,path) (rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m]))`
  * `rate(demo_api_request_duration_seconds_sum{job="fake_app"}[30s])`
  * `sum by(status) (rate(demo_api_request_duration_seconds_sum{job="fake_app"}[5m]) / rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m]))`
  * `sum by(status) (rate(demo_api_request_duration_seconds_sum{job="fake_app"}[5m]) / rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m])) > 0.3`
  * `up`
  * `up == 0`

### 8. Alerting basics

Alerting has two components in prometheus.

1. The alerting rule files for the prometheus executable.
2. The alertmanager executable and it's config

In prometheus you create alerts based on rules, which is stored in a config yaml file like the main config file. Alertmanager then connects to the running prometheus instance to scrape the alerts.

* edit our main prometheus config file and add:

```bash
...
rule_files:
  - './alerts.d/*'
...
```

* `cat ~/prometheus/alerts.d/example.yml`

```bash
groups:
- name: example
  rules:
  # Alert for any instance that is unreachable for >1 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."

  # Alert for any instance that a request took more than 300 ms to complete.
  - alert: APIHighResponseTime
    expr: sum by(status) (rate(demo_api_request_duration_seconds_sum{job="fake_app"}[5m]) / rate(demo_api_request_duration_seconds_count{job="fake_app"}[5m])) > 0.3
    for: 1m
    annotations:
      summary: "High request response time on {{ $labels.instance }}"
      description: "{{ $labels.instance }} Request complete time is above 300ms (current value: {{ $value }}s)"
```

* Now issue this command on the terminal to reload the config files:

```bash
curl -X POST http://localhost:9090/-/reload
```

* And see the Alerts and the Status/Rules page in the Prometheus UI.


## Survey

Please help us improve by filling this survey out. Only takes about 2 minutes, and your help is greatly appreciated:
http://bit.ly/2CCqbUA


## Useful links:

* https://github.com/roaldnefs/awesome-prometheus
* https://www.slideshare.net/grobie/the-history-of-prometheus-at-soundcloud
* https://www.digitalocean.com/community/tutorial_series/how-to-query-prometheus
* Prometheus: https://github.com/prometheus/prometheus
* Node exporter: https://github.com/prometheus/node_exporter
* Functions: https://prometheus.io/docs/prometheus/latest/querying/functions/
