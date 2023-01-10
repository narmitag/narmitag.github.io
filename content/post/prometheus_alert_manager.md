---
title: "Prometheus Alert Manager" # Title of the blog post.
date: 2023-01-05T08:41:16Z # Date of post creation.
description: "Installing and using Alert Manager." # Description used for search engine.
featured: false # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
featureImage: "/images/prom/alertmanager.png" # Sets featured image on blog post.
thumbnail: "/images/prom/prometheus_small.png" # Sets thumbnail image appearing inside card on homepage.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Guides
tags:
  - prometheus
  - alert manager
# comment: false # Disable comment if false.
---

**The Prometheus [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) handles alerts sent by client applications such as the Prometheus server. It takes care of deduplicating, grouping, and routing them to the correct receiver integration such as email, PagerDuty, or OpsGenie. It also takes care of silencing and inhibition of alerts.**

To ensure that alert messages get delivered it's a good idea to run several alertmanager processes clustered together, so in this example we will deploy 2 on different machines

> These instructions assume you already have a running Prometheus server - see [here]({{< ref "/post/prometheus_install" >}}) for an example.


# Installing Alert Manager

Initially this will be done on the existing Prometheus server, these instructions can also be [downloaded](https://raw.githubusercontent.com/narmitag/prometheus/main/install_alert_manager.sh) as a install script


Set the required versions

```bash
export VERSION=0.25.0
export ARCH=arm64
```
and create a user to run the process under

```bash
useradd -M -r -s /bin/false alertmanager
```

Download the binaries and install it in ```/usr/local/bin```

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v$VERSION/alertmanager-$VERSION.linux-$ARCH.tar.gz
tar xvfz alertmanager-$VERSION.linux-$ARCH.tar.gz

cp alertmanager-$VERSION.linux-$ARCH/{alertmanager,amtool} /usr/local/bin/
chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
```

Copy the sample config files and create the config file for amtool
```bash
mkdir -p /etc/alertmanager
cp alertmanager-$VERSION.linux-$ARCH/alertmanager.yml /etc/alertmanager
chown -R alertmanager:alertmanager /etc/alertmanager
mkdir -p /var/lib/alertmanager
chown alertmanager:alertmanager /var/lib/alertmanager
mkdir -p /etc/amtool

cat > /etc/amtool/config.yml <<EOF
alertmanager.url: http://localhost:9093
EOF
```

Create and start the systemd process to run alertmanager
```bash

cat > /etc/systemd/system/alertmanager.service <<EOF
[Unit]
Description=Prometheus Alertmanager
Wants=network-online.target
After=network-online.target
[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
 --config.file /etc/alertmanager/alertmanager.yml \
 --storage.path /var/lib/alertmanager/
[Install]
WantedBy=multi-user.target
EOF

systemctl enable alertmanager
systemctl start alertmanager
```

Use amtool to check the configuration
```bash
neilarmitage@prom1:~/prometheus$ amtool config show
global:
  resolve_timeout: 5m
  http_config:
    follow_redirects: true
    enable_http2: true
  smtp_hello: localhost
  smtp_require_tls: true
  pagerduty_url: https://events.pagerduty.com/v2/enqueue
  opsgenie_api_url: https://api.opsgenie.com/
  wechat_api_url: https://qyapi.weixin.qq.com/cgi-bin/
  victorops_api_url: https://alert.victorops.com/integrations/generic/20131114/alert/
  telegram_api_url: https://api.telegram.org
  webex_api_url: https://webexapis.com/v1/messages
route:
  receiver: web.hook
  group_by:
  - alertname
  continue: false
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
inhibit_rules:
- source_match:
    severity: critical
  target_match:
    severity: warning
  equal:
  - alertname
  - dev
  - instance
receivers:
- name: web.hook
  webhook_configs:
  - send_resolved: true
    http_config:
      follow_redirects: true
      enable_http2: true
    url: http://127.0.0.1:5001/
    max_alerts: 0
templates: []
```

Now Prometheus needs to know about the alertmanager so edit ```vi /etc/prometheus/prometheus.yml``` and add

```yaml
alerting:
 alertmanagers:
 - static_configs:
 - targets: ["localhost:9093"]
```

and then restart Prometheus ```sudo  systemctl restart prometheus```

The new alertmanager should now show up under the status tab in Prometheus

![Alert Manager](/images/prom/alert_status1.png)

# Adding a second Alert Manager Instance

To ensure Alerts get delivered a second instance of Alert Manager is recommended. The setup of the Alert Manager on a separate host is the same as the initial setup with the exception of a addition command option on the startup of Alert Manager so it knows where the other host is ```--cluster.peer=```


```bash
[Unit]
Description=Prometheus Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
 --config.file /etc/alertmanager/alertmanager.yml \
 --storage.path /var/lib/alertmanager/ \
 --cluster.peer=<IP of first alertmanager>:9094

[Install]
WantedBy=multi-user.target
```

Once the second instance is running update the Systemd file on the first alert manager to add the ```--cluster.peer=<ip of 2nd host>:9094``` The reload the config and restart the service

```bash
sudo systemctl daemon-reload
sudo systemctl restart alertsudomanager
```

Now Prometheus needs to know about the second alertmanager so edit ```vi /etc/prometheus/prometheus.yml``` and add

```yaml
alerting:
 alertmanagers:
 - static_configs:
 - targets: ["localhost:9093","IP of 2nd alertmanager:9093]
```

and then restart Prometheus ```sudo  systemctl restart prometheus```

The new alertmanager should now show up under the status tab in Prometheus

![Alert Manager](/images/prom/alert_status2.png)

# Writing Alerts

First of all edit ```sudo vi /etc/prometheus/prometheus.yml``` and update the rules_files option to add a rules directory, this will allow us to create multiple files in this directory without having to further update the config.



```yaml
rule_files:
- "/etc/prometheus/rules/*.yml"
```

Create the rules file directory

```bash
sudo mkdir /etc/prometheus/rules/
sudo chown prometheus:prometheus /etc/prometheus/rules/
```

Now we can create a test rule ```sudo vi /etc/prometheus/rules/test.yml```

```yaml
groups:
- name: test-server
  rules:
  - alert: TestServerDown
    expr: up{job="test server"}==0
    labels:
      severity: critical
    annotations:
      summary: Test Server Down
```

Now restart Prometheus to pick up the change ```sudo systemctl restart prometheus```

The Alert should now show on the UI as Inactive

![Alert Inactive](/images/prom/alert_ok.png)

If you stop the node_exporter on the test server ```sudo systemctl stop node_exporter``` and wait a minute the alert should show as active


![Alert Active](/images/prom/alert_down.png)