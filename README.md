# slinky-monitoring

A monitoring solution for node runners and validators utilizing docker containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [NodeExporter](https://github.com/prometheus/node_exporter), and alerting with [AlertManager](https://github.com/prometheus/alertmanager). 

This is intended to be a single-stop solution for monitoring your Slinky Side Car needs.

## TL;DR: Steps

Clone this repository on your Docker host, cd into slinky-monitoring directory and run compose up:

```bash
git clone https://github.com/LavenderFive/slinky-monitoring
cd slinky-monitoring
cp .env.sample .env
export NODE_URL=localhost:1317 # Enter your own node url here
docker run -it --rm --entrypoint sh -v $(pwd)/slinky:/slinky ghcr.io/skip-mev/slinky-sidecar:latest -c "slinky-config --chain dydx --node-http-url $NODE_URL --oracle-config-path /slinky/oracle.json"
docker-compose up -d
```

## Generate Slinky Config
This command will create the Slinky oracle.json config file under `~/slinky/`. Unless you are running this repo
on the same server as the node, you will want to change the `NODE_URL` from localhost.
```sh
export NODE_URL=localhost:1317 # Enter your own node url here
docker run -it --rm --entrypoint sh -v $(pwd)/slinky:/slinky ghcr.io/skip-mev/slinky-sidecar:latest -c "slinky-config --chain dydx --node-http-url $NODE_URL --oracle-config-path /slinky/oracle.json"
```

## Setup Grafana

### Grafana  Dashboard
This monitoring solution comes built in with a Slinky Monitoring dashboard, 
which works out of the box. Grafana, Prometheus, and Infinity are installed 
automatically.

---

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up. The config file can be added directly in grafana part like this

```yaml
grafana:
  image: grafana/grafana:7.2.0
  env_file:
    - .env
```

and the config file format should have this content

```yaml
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```

If you want to change the password, you have to remove this entry, otherwise the change will not take effect

```yaml
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: [http://prometheus:9090](http://prometheus:9090)
* Access: proxy

***Monitor Services Dashboard***

![Monitor Services](https://raw.githubusercontent.com/LavenderFive/slinky-monitoring/master/screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

## Define alerts

Two alert groups have been setup within the [alert.rules](https://github.com/LavenderFive/slinky-monitoring/blob/master/prometheus/alert.rules) configuration file:

* Monitoring services alerts [targets](https://github.com/LavenderFive/slinky-monitoring/blob/master/prometheus/alert.rules#L13-L22)
* Peggo alerts [peggo](https://github.com/LavenderFive/slinky-monitoring/blob/master/prometheus/alert.rules#L2-L11)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```bash
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```


## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/LavenderFive/slinky-monitoring/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](https://raw.githubusercontent.com/LavenderFive/slinky-monitoring/master/screens/Slack_Notifications.png)
