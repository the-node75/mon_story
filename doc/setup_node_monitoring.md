# Installation monitoring tools on nodes

## Setup variables

```
MONITORING_HOSTNAME=<server name under which the metrics will be displayed> 
MON_SERVER_URL="http://YOUR_MONITORING_SERVER_IP:8086"
ORG_NAME=<your organization name>
BUCKET_NAME=story
INFLUXDB_NODE_TOKEN=<your node token for InfluxDB>
```

> `MONITORING_HOSTNAME` # for each server must be unique

> `MON_SERVER_URL` url of server with InfluxDB server, see [setup_monitoring_server.md](./setup_monitoring_server.md)

> `ORG_NAME` the name of your organization or any random name, such as Umbrella corp

> see generation guide for `INFLUXDB_NODE_TOKEN` in [setup_monitoring_server.md](./setup_monitoring_server.md) under *Create token for nodes*


## Enable monitoring on the Story consensus node (CometBFT node)

We use data from the built-in monitoring port of the CometBFT node

To enable it, you need to make changes to the configuration file, default location *~/.story/story/config/config.toml*

You need to find the `[instrumentation]` section and turn on prometeus, by set `prometheus = true`

Make sure that nothing on your system is using port `26660`, if not, change the `prometheus_listen_addr` configuration.
```
#######################################################
###       Instrumentation Configuration Options     ###
#######################################################
[instrumentation]

# When true, Prometheus metrics are served under /metrics on
# PrometheusListenAddr.
# Check out the documentation for the list of available metrics.
prometheus = true

# Address to listen for Prometheus collector(s) connections
prometheus_listen_addr = ":26660"

# Maximum number of simultaneous connections.
# If you want to accept a larger number than the default, make sure
# you increase your OS limits.
# 0 - unlimited.
max_open_connections = 3

# Instrumentation namespace
namespace = "cometbft"
```

This changes will take effect after the node is restarted. 

```
sudo systemctl restart storyd.service 
```

Check monitoring port:
```
COMETBFT_MON_PORT=26660 # or anoter port
curl -s localhost:$COMETBFT_MON_PORT
```




## Enable monitoring on the Story Geth node

Geth provides the most detailed metrics possible when passing data directly to InfluxDB
So for Geth we do not use the monitoring-port, but configure the work with the database at once.

To configure sending metrics, add the following flags to the geth node run command

```
--metrics \
--metrics.influxdbv2 \
--metrics.influxdb.tags "host=$MONITORING_HOSTNAME" \
--metrics.influxdb.endpoint "$MON_SERVER_URL" \
--metrics.influxdb.token "$INFLUXDB_NODE_TOKEN" \
--metrics.influxdb.organization "$ORG_NAME" \
--metrics.influxdb.bucket "$BUCKET_NAME" \
```
Restart the node, if you made changes to the service-file, don't forget to first `sudo systemctl daemon-reload`

We can check the success of this step if the monitoring server and Grafana are fully up to date. Refresh the dashboard, find the corresponding `MONITORING_HOSTNAME` at the top of the host page. Metrics related to the work of Geth node should be displayed.

## Install Telegraf

```
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

sudo apt update && sudo apt install telegraf

sudo systemctl enable --now telegraf
sudo systemctl is-enabled telegraf

# make the telegraf user sudo and adm to be able to execute scripts as umee user
sudo adduser telegraf sudo
sudo adduser telegraf adm
sudo -- bash -c 'echo "telegraf ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'

# Backup configuration
sudo mv /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.orig
```

### Create new configuration for Telegraf


```
tee $HOME/telegraf.conf > /dev/null <<EOF
# Global Agent Configuration
[agent]
  hostname = "${MONITORING_HOSTNAME}"
  flush_interval = "15s"
  interval = "15s"
# Input Plugins
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["devtmpfs", "devfs"]
[[inputs.diskio]]
[[inputs.mem]]
[[inputs.net]]
[[inputs.nstat]]
[[inputs.netstat]]
[[inputs.linux_sysctl_fs]]
[[inputs.system]]
[[inputs.swap]]
[[inputs.processes]]
[[inputs.interrupts]]
[[inputs.kernel]]

[[outputs.influxdb_v2]]
  ## The URLs of the InfluxDB cluster nodes.
  ##
  ## Multiple URLs can be specified for a single cluster, only ONE of the
  ## urls will be written to each interval.
  ##   ex: urls = ["https://us-west-2-1.aws.cloud2.influxdata.com"]
  urls = ["$MON_SERVER_URL"]
  
  ## Local address to bind when connecting to the server
  ## If empty or not set, the local address is automatically chosen.
  # local_address = ""

  ## Token for authentication.
  token = "${INFLUXDB_NODE_TOKEN}"
  
  ## Organization is the name of the organization you wish to write to.
  organization = "${ORG_NAME}"

  ## Destination bucket to write into.
  bucket = "${BUCKET_NAME}"


# cometbft node
[[inputs.prometheus]]
  ## An array of urls to scrape metrics from.
  urls = ["http://localhost:${COMETBFT_MON_PORT}"]

  
EOF
```

Setup config

```
sudo mv /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.orig # backup original config file
sudo cp $HOME/telegraf.conf /etc/telegraf/telegraf.conf
sudo systemctl restart telegraf
```

Check telegraf status
```
sudo systemctl status telegraf
```

Now you can see the result in the Grafana dashboard!!

