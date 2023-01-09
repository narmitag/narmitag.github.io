---
title: "Monitoring NGINX with Prometheus" # Title of the blog post.
date: 2023-01-03T08:38:26Z # Date of post creation.
description: "Using Prometheus to monitor NGINX." # Description used for search engine.
featured: false # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
featureImage: "/images/prom/prom_nginx.png" # Sets featured image on blog post.
featureImageAlt: 'Description of image' # Alternative text for featured image.
thumbnail: "/images/prom/nginx_small.png" # Sets thumbnail image appearing inside card on homepage.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Guides
tags:
  - prometheus
# comment: false # Disable comment if false.
---

**Using the [NGINX Prometheus exporter](https://github.com/nginxinc/nginx-prometheus-exporter) to add MGINX metrics into Prometheus.**

NGINX Prometheus exporter fetches the metrics from a single NGINX or NGINX Plus, converts the metrics into appropriate Prometheus metrics types and finally exposes them via an HTTP server to be collected by Prometheus.

> These instructions assume you already have a running Prometheus server - see [here]({{< ref "/post/prometheus_install" >}}) for an example.

# Installing NGINX and exposing metrics

Using a Ubuntu host

```bash
sudo apt install nginx
```

Update the configuration to exposed the Basic Status page ```sudo vi /etc/nginx/sites-enabled/default```

and add
``` 
      location /nginx_status {
        stub_status;
        allow 127.0.0.1;        #only allow requests from localhost
        deny all;               #deny all other hosts
      }
```

and restart nginx ```sudo systemctl restart nginx```

The metrics should now be available with curl

```bash
root@nginx:/etc/nginx/sites-enabled# curl localhost/nginx_status
Active connections: 1
server accepts handled requests
 2 2 2
Reading: 0 Writing: 1 Waiting: 0
```

# Running the Node exporter

Set the required version and architecture (arm64 or amd64)
```Bash
export VERSION=0.11.0
export ARCH=arm64
```

Create a user
```Bash
useradd -M -r -s /bin/false nginx_exporter
```

Download and install the binary

```Bash
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v$VERSION/nginx-prometheus-exporter_${VERSION}_linux_$ARCH.tar.gz
tar xvfz nginx-prometheus-exporter_${VERSION}_linux_${ARCH}.tar.gz

cp nginx-prometheus-exporter /usr/local/bin/
chown nginx_exporter:nginx_exporter /usr/local/bin/nginx-prometheus-exporter
```

Create a SystemD service for the nginx_exporter

```Bash
cat > /etc/systemd/system/nginx_exporter.service <<EOF
[Unit]
Description=Prometheus Nginx Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=nginx_exporter
Group=nginx_exporter
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter -nginx.scrape-uri=http://localhost/nginx_status

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start nginx_exporter
systemctl enable nginx_exporter
```

The exporter should now be exposing metrics

```bash
root@nginx:/etc/nginx/sites-enabled# curl localhost:9113/metrics
# HELP nginx_connections_accepted Accepted client connections
# TYPE nginx_connections_accepted counter
nginx_connections_accepted 3
# HELP nginx_connections_active Active client connections
# TYPE nginx_connections_active gauge
nginx_connections_active 1
# HELP nginx_connections_handled Handled client connections
# TYPE nginx_connections_handled counter
nginx_connections_handled 3
# HELP nginx_connections_reading Connections where NGINX is reading the request header
# TYPE nginx_connections_reading gauge
nginx_connections_reading 0
# HELP nginx_connections_waiting Idle client connections
# TYPE nginx_connections_waiting gauge
nginx_connections_waiting 0
# HELP nginx_connections_writing Connections where NGINX is writing the response back to the client
# TYPE nginx_connections_writing gauge
nginx_connections_writing 1
# HELP nginx_http_requests_total Total http requests
# TYPE nginx_http_requests_total counter
nginx_http_requests_total 4
# HELP nginx_up Status of the last metric scrape
# TYPE nginx_up gauge
nginx_up 1
# HELP nginxexporter_build_info Exporter build information
# TYPE nginxexporter_build_info gauge
nginxexporter_build_info{arch="linux/arm64",commit="e4a6810d4f0b776f7fde37fea1d84e4c7284b72a",date="2022-09-07T21:09:51Z",dirty="false",go="go1.19",version="0.11.0"} 1
```


On the Prometheus host edit the file ```/etc/prometheus/prometheus.yml``` and add a new job in the required section

```Bash
 - job_name: 'NGINX Server'
  static_configs:
  - targets: ['<IP_ADDRESS_OF_NGINX_SERVER>:9113']
```

and restart Prometheus ```sudo systemctl restart prometheus```

The NGINX metrics should now start to appear in Prometheus

