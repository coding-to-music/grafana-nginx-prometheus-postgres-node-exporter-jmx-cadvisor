# grafana-nginx-prometheus-postgres-node-exporter-jmx-cadvisor

# 🚀 Monitoring solution with NGINX, Grafana, Prometheus and several Prometheus exporters, like cAdvisor, node-exporter, postgres_exporter and jmx_exporter. 🚀

https://github.com/coding-to-music/grafana-nginx-prometheus-postgres-node-exporter-jmx-cadvisor

From / By https://github.com/savvydatainsights

https://github.com/savvydatainsights/monitoring

## Environment variables:

```java

```

## GitHub

```java
git init
git add .
git remote remove origin
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:coding-to-music/grafana-nginx-prometheus-postgres-node-exporter-jmx-cadvisor.git
git push -u origin main
```

# SDI Monitoring

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Project for developing the SDI monitoring solution, consisted basically with the following components:

- [NGINX](https://www.nginx.com) - For [reverse proxying](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy) and [access restriction through HTTP basic authentication](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication);
- [Grafana](https://grafana.com) - The open platform for analytics and monitoring;
- [Prometheus](https://prometheus.io) - Monitoring system and time series database;
- [Prometheus exporters](https://prometheus.io/docs/instrumenting/exporters):
  - [cAvisor](https://github.com/google/cadvisor) - Analyzes resource usage and performance characteristics of running containers;
  - [node_exporter](https://github.com/prometheus/node_exporter) - Prometheus exporter for hardware and OS metrics exposed by \*NIX kernels;
  - [postgres_exporter](https://github.com/wrouesnel/postgres_exporter) - Prometheus exporter for PostgreSQL server metrics;
  - [jmx_exporter](https://github.com/prometheus/jmx_exporter) - JMX to Prometheus exporter: a collector that can configurably scrape and expose mBeans of a JMX target.

Table of Contents:

- [SDI Monitoring](#sdi-monitoring)
  - [Setup](#setup)
  - [Putting Prometheus exporters behind NGINX](#putting-prometheus-exporters-behind-nginx)
  - [Adding hosts](#adding-hosts)
  - [The dashboards](#the-dashboards)
  - [Deploying to Azure](#deploying-to-azure)

## Setup

In order to set the monitoring environment up, follow the steps below:

1. Create the file required for implementing basic authentication in NGINX, by executing the command: `htpasswd -c nginx/basic_auth/.htpasswd prometheus`;
2. Put in the file _prometheus/basic_auth_password_ the same password used previously. Prometheus will use this file to set the Authorization header during requests to exporters;
3. Finally, turn everything on through running: `docker-compose up -d`

Alternativelly to manually following the mentioned steps, you can just execute

```
ansible-playbook playbooks/setup.yml
```

You will be prompted to type the password, and then all the steps will be performed automatically.

## Putting Prometheus exporters behind NGINX

In our solution, all the Prometheus exporters have [NGINX](https://www.nginx.com) in front of them, as a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) and requiring [basic authentication](https://en.wikipedia.org/wiki/Basic_access_authentication). It's a good idea if you already have NGINX in your server, as a proxy server to other services. You restrict all the requests to a single port (80), avoiding every exporter from exposing its default port to the world.

The configuration below is an example of how you can configure NGINX. Use the same _.htpasswd_ file generated during the setup process, described earlier, for each Prometheus exporter. If you prefer, create specific files for different exporters, using [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html). Note: Bear in mind you will have to [configure Prometheus](prometheus/prometheus.yml) appropriately if you use either a different user than _prometheus_ or different passwords for different exporters.

```nginx
server {
    listen 80 default_server;

    location /docker-metrics {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/basic_auth/cadvisor.htpasswd;
        proxy_pass http://localhost:8080/metrics;
    }

    location /node-metrics {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/basic_auth/node-exporter.htpasswd;
        proxy_pass http://localhost:9100/metrics;
    }

    location /postgres-metrics {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/basic_auth/postgres_exporter.htpasswd;
        http://localhost:9187/metrics
    }

    location /jvms-metrics {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/basic_auth/jmx_exporters.htpasswd;
        http://localhost:9090/federate
    }
}
```

Notice in the configuration that Prometheus can be used to aggregate JVM metrics generated by jmx_exporter instances. The Prometheus server responsible for centralizing all the JVM metrics then is able to be scraped by the main Prometheus server, from a single endpoint. This is a feature of Prometheus called [federation](https://prometheus.io/docs/prometheus/latest/federation).

## Adding hosts

By default, the localhost is automatically monitored. However, you can add exporters of other hosts, by adding more Prometheus targets. Inside the [project's playbooks folder](playbooks), you will find Ansible playbooks which turn the task of adding targets to Prometheus much easier. To add a cAdvisor target, for example, execute:

`ansible-playbook playbooks/add-cadvisor.yml -e host=hostname -e target=ip:8080`

Replace _hostname_ and _ip_ with the appropriate values. If cAdvisor exposes the metrics through other port than 8080, change it too. Following the example, the metrics should be available by accessing <http://ip:8080/metrics>. Note: If cAdvisor is behind NGINX, the port is not important, once NGINX answers through the default HTTP port 80.

If your Prometheus server is in a remote host, you must set the _prometheus_host_ parameter, and a [inventory file](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) where previously you must have have put the SSH credentials required for Ansible connection:

`ansible-playbook playbooks/add-cadvisor.yml -i playbooks/inventory -e prometheus_host=production -e host=hostname -e target=ip:8080`

![Monitoring diagram](images/sdi-monitoring.png)

The diagram above shows you can add as many hosts as you want, each host with one or more instances of exporters from where Prometheus scrapes metrics.

## The dashboards

Grafana is available on port 3000. During its setup, the connection with Prometheus is made, and [dashboards](grafana/dashboards) are provisioned. They are all based on [dashboards shared by the community](https://grafana.com/dashboards). The table below shows the dashboards our Grafana has by default:

| Dashboard           | Original id                                 | Picture                                                         |
| ------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Docker monitoring   | [193](https://grafana.com/dashboards/193)   | ![Docker Monitoring dashboard](images/docker-dashboard.png)     |
| Host monitoring     | [6014](https://grafana.com/dashboards/6014) | ![Host Monitoring dashboard](images/host-dashboard.png)         |
| Postgres monitoring | [455](https://grafana.com/dashboards/455)   | ![Postgres Monitoring dashboard](images/postgres-dashboard.png) |
| JVM monitoring      | [3066](https://grafana.com/dashboards/3066) | ![JVM Monitoring dashboard](images/jvm-dashboard.png)           |

The dashboards were slightly changed from its originals for enabling the alternation between hosts.

## Deploying to Azure

With the Ansible playbook [deploy-to-azure.yml](playbooks/deploy-to-azure.yml) is possible to deploy the monitoring solution to a VM in [Azure](https://azure.microsoft.com). The playbook creates all the required resources and then runs the services in the new remote VM, created from a [baked Ubuntu image](https://github.com/savvydatainsights/ubuntu).

`ansible-playbook playbooks/deploy-to-azure.yml`
