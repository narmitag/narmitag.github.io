---
title: "Installing Prometheus" # Title of the blog post.
date: 2023-01-02T08:40:51Z # Date of post creation.
description: "Installing Prometheus from scratch." # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
featureImage: "/images/prom/prom_1.png" # Sets featured image on blog post.
featureImageAlt: 'prometheus log' # Alternative text for featured image.
thumbnail: "/images/prom/prometheus_small.png" # Sets thumbnail image appearing inside card on homepage.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Guides
tags:
  - prometheus
# comment: false # Disable comment if false.
---

**[Prometheus](https://github.com/prometheus) is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. It has been used by many companies for over 10 years and forms the foundation of a good and flexiable monitoring system**

In this the first of a series of posts we will go back to the basics installing Prometheus and node exporter on either VM's or Bare-metal hosts. In later posts the installation on Kubernetes will be covered.

> Requirements: To complete this you will need at least one, preferably  two virtual machines running Ubuntu 22.04. The instructions will work on other operating systems but it has not been tested.

# Installation of the Prometheus Server

These instructions can be downloaded as a single [script](https://raw.githubusercontent.com/narmitag/prometheus/main/install_prom_vm.sh)

Set the required version and architecture (arm64 or amd64)
```Bash
export VERSION=2.18.0
export ARCH=arm64
```

Create a user to run Prometheus and create the required directories

```Bash
useradd -M -r -s /bin/false prometheus
mkdir /etc/prometheus /var/lib/prometheus
```

Download and install the binaries
```Bash
wget https://github.com/prometheus/prometheus/releases/download/v$VERSION/prometheus-$VERSION.linux-$ARCH.tar.gz
tar xzf prometheus-$VERSION.linux-$ARCH.tar.gz prometheus-$VERSION.linux-$ARCH/

cp prometheus-$VERSION.linux-$ARCH/{prometheus,promtool} /usr/local/bin/
chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
```

Copy the default setup files from the download
```Bash
cp -r prometheus-$VERSION.linux-$ARCH/{consoles,console_libraries} /etc/prometheus/
cp prometheus-$VERSION.linux-$ARCH/prometheus.yml /etc/prometheus/prometheus.yml

chown -R prometheus:prometheus /etc/prometheus
chown prometheus:prometheus /var/lib/prometheus
```

Create a SystemD service file for Prometheus and Start the process

```Bash
cat > /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
 --config.file /etc/prometheus/prometheus.yml \
 --storage.tsdb.path /var/lib/prometheus/ \
 --web.console.templates=/etc/prometheus/consoles \
 --web.console.libraries=/etc/prometheus/console_libraries
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
```

If everything has worked ok you should now be able to curl Prometheus and get a response

```Bash
curl localhost:9090
```

The UI should also be available on ```http:<server-ip>:9090```

If the above fail check the output of ```systemctl status prometheus``` and ```journalctl -u prometheus```

# Installation of the Node Exporter

This can be installed on the same host as Prometheus but if possible on a separate host.

These instructions can be downloaded as a single [script](https://raw.githubusercontent.com/narmitag/prometheus/main/install_node_exporter.sh)

Set the required version and architecture (arm64 or amd64)
```Bash
export VERSION=1.5.0
export ARCH=arm64
```

Create a user
```Bash
useradd -M -r -s /bin/false node_exporter
```

Download and install the binary

```Bash
wget https://github.com/prometheus/node_exporter/releases/download/v$VERSION/node_exporter-$VERSION.linux-$ARCH.tar.gz
tar xvfz node_exporter-$VERSION.linux-$ARCH.tar.gz

cp node_exporter-$VERSION.linux-$ARCH/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Create a SystemD service for the node_exporter

```Bash
cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

The node exporter should respond on port 9100
```Bash
curl localhost:9100/metrics
```

Now the Prometheus configuration needs to be updated to scrape the node exporter configuration. Prometheus works on a pull model so it will pull all the metrics from end-points rather than having the metrics pushed to it.

On the Prometheus host edit the file ```/etc/prometheus/prometheus.yml``` and add a new job in the required section

```Bash
 - job_name: 'Test Server'
  static_configs:
  - targets: ['<IP_ADDRESS_OF_TEST_SERVER>:9100']
```

To allow Prometheus to pick up the new config either by restarting the service ```systemctl restart prometheus``` or by sending a HUP to the process ```killall -HUP prometheus```

Prometheus will now be picking up metrics from the node exporter. To test enter the expression ```up``` in the UI and press execute. It should return 2 jobs.

![Node exporter](/images/prom/up_command.png)