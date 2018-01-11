# Prometheus Stack

This is a simple server monitoring stack using prometheus and Grafana for
servers hosted on AWS EC2.

## Components
There are several pieces that make the server monitoring work. They are outlined
below.

### Node Exporter
The prometheus stack follows the pull model where prometheus pulls the server
metrics from the servers it cares about looking at. Prometheus is currently
setup to look at all EC2 servers and gets the list of servers dynamically
from AWS. The node exporter should not be accessible from the internet.

Each box runs a small web-app called a Node Exporter that has the data for
Prometheus. Each box that is able should run the node exporter through a docker
container. Here is the code to run the Node Exporter on a box with docker.

```bash
docker run --restart always --name node-exporter -d -p 9100:9100 -v "/proc:/host/proc" -v "/sys:/host/sys" -v "/:/rootfs" --net="host" prom/node-exporter --path.procfs /host/proc --path.sysfs /host/proc --collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

The Node Monitor exposes these stats on port 9100, an example can be seen here:
http://internal.ip:9100/metrics

### Prometheus
Prometheus then visits each of these target Node Monitors and scrapes its status
every 5 seconds. You can see the list of targets on Prometheus here : http://internal.ip:9090/consoles/node.html
The prometheus service should not be accessible from the internet.

The data is stored on prometheus as time series data, but it is not easy to
consume from a user perspective.

Prometheus is configured in the prometheus/prometheus.yml file. It is setup to
look for the node exporter on port 9100 for all boxes on AWS EC2.

### Grafana
The third service running is Grafana and its sole role is to display the prometheus
data. Unlike the other services, Grafana is publicly exposed to the internet but
requires authentication: http://monitor.server.com:3000/

It is setup to use Prometheus as a data source and we have used a prebuilt
dashboard to show the server statistics.

The main dashboard was created using this Grafana template: https://grafana.com/dashboards/405

### Alertmanager
The last piece is Alertmanager that is responsible for alerting when alerts are
triggered. Currently alertmanager is setup to just notify the #network_operations
channel in slack. Alertmanager is configured according to the
alertmanater/config.yml file.

Alerts are triggered according to the prometheus/alert.rules file. Prometheus then
sends notifications to the alertmanager service which is responsible for sending
out the alerts.

Previously fired alerts can be seen at http://internal.ip:9093/api/v1/alerts
The Alertmanager service should not be accessible from the internet.

## More Documentation
Prometheus: https://prometheus.io/docs/introduction/overview/

Grafana: http://docs.grafana.org/

Alertmanager: https://prometheus.io/docs/alerting/alertmanager/

## Deploying and Updating
Deploying updates to prometheus/alertmanager configs is manual at this point.
We assume that the updating of rules will be seldom and automating the
deployments would be overengineering and likely cause more problems than it solves.

The config files are all reloaded with the SIGHUP command, so a simple restarting
of the containers will use the updated config and rules files.

```bash
docker-compose restart
```
