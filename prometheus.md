# Prometheus install guide
Prometheus is a tool for collecting all sorts of performance metrics from a server and/or application over time. This can be very useful for monitoring the state of your server and diagnosing issues if they arise. This guide sets out the steps required to install Prometheus, along with the Prometheus node_exporter, in order to monitor an Ubuntu server.

## Acknowledgements and Disclaimer
The contents of this guide has been pulled together from a variety of sources. It has been tested on Ubuntu Server 18.04 and 20.04. Your mileage may vary. This guide is designed to be generic and as such, does not include direct monitoring for any applications. Please see my other guides for specific monitoring guides for applications.

## Prerequisites
This guide is not intended for absolute beginners. It assumes some knowledge of using a linux terminal. Before you get started you will need to have your Ubuntu server instance up and running. Your server will require an internet connection. This guide assumes that you are logged into the server using a non-root account with SUDO access. Security will not be covered in this guide  however it is _strongly_ recommended that you do not expose any of the Prometheus ports directly to the internet without configuring encryption and authentication, which are out of scope of this guide.

## Overview
This diagram below illustrates the two basic components to be installed on the server - Prometheus service and the Prometheus node_exporter service. The Grafana server, which is responsible for serving the dashboards that will display the server metrics, is covered in the Grafana guide in the wiki.

![](https://i.imgur.com/1pIQXJa.png)

## Installation - Prometheus
### User creation
Create a user specifically for prometheus. This is a more secure way of running the new service.

```
sudo useradd --no-create-home --shell /bin/false prometheus
```


### Create Prometheus directories
Create the directories that Prometheus files will be installed to

```
sudo mkdir /etc/prometheus
```

```
sudo mkdir /var/lib/prometheus
```

Make the user "prometheus" the owner of the new folders

```
sudo chown -R prometheus:prometheus /etc/prometheus
```

```
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

### Download Prometheus software
Download the latest release of Prometheus from Github

```
mkdir -p /tmp/prometheus && cd /tmp/prometheus
curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest \
  | grep browser_download_url \
  | grep linux-amd64 \
  | cut -d '"' -f 4 \
  | wget -qi -
```    

Extract the downloaded archive and move the files to the install directories created earlier

```
cd /tmp/prometheus
tar xvf prometheus*.tar.gz
cd prometheus*/
sudo mv prometheus promtool /usr/local/bin/
sudo mv prometheus.yml  /etc/prometheus/prometheus.yml
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

Clean up the temp files
```
cd ~/
rm -rf /tmp/prometheus
```
### Create a systemd configuration file
We want to run Prometheus as a systemd service so that it will automatically restart itself if it crashes, and to make it easy to start and stop the Prometheus service.

    sudo tee /etc/systemd/system/prometheus.service<<EOF

    [Unit]
    Description=Prometheus
    Documentation=https://prometheus.io/docs/introduction/overview/
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Type=simple
    User=prometheus
    Group=prometheus
    ExecReload=/bin/kill -HUP $MAINPID
    ExecStart=/usr/local/bin/prometheus \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/var/lib/prometheus \
      --web.console.templates=/etc/prometheus/consoles \
      --web.console.libraries=/etc/prometheus/console_libraries \
      --web.listen-address=0.0.0.0:9090 \
      --web.external-url=
    
    SyslogIdentifier=prometheus
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    EOF


Reload the systemd daemon to include the new service configuration

```
sudo systemctl daemon-reload
```

Start the Prometheus service

```
sudo systemctl start prometheus.service
```

Check that Prometheus is running as expected

```
sudo systemctl status prometheus.service
```

The output should be similar to the screenshot below, as long as the service shows active rather than failed/stopped, it is likely to be running as required:

![](https://i.imgur.com/AYUysBa.png)

Enable the Prometheus service so it will restart if it fails
```
sudo systemctl enable prometheus.service
```

## Installation - Prometheus node_exporter
Create a user specifically for prometheus node_exporter. This is a more secure way of running the new service.

```
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### Download Prometheus node_exporter software
Download the latest release of Prometheus node_exporter from Github
```
mkdir -p /tmp/prometheus && cd /tmp/prometheus
curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest \
  | grep browser_download_url \
  | grep linux-amd64 \
  | cut -d '"' -f 4 \
  | wget -qi -
```
Extract the downloaded archive and move the files to the install directories created earlier
```
cd /tmp/prometheus
tar xvf node_exporter*.tar.gz
cd node_exporter*/
sudo mv node_exporter /usr/local/bin/
```

Change the user and group ownership to the node_exporter user

```
sudo chown -R node_exporter:node_exporter /usr/local/bin/node_exporter
```

Clean up the temp files
```
cd ~/
rm -rf /tmp/prometheus
```
### Add a Prometheus scrape job to the Prometheus configuration in /etc/prometheus/prometheus.yml

In order for the Prometheus service to pick up the metrics provided by the node_exporter service, we need to instruct Prometheus on where to scrape the metrics from. node_exporter listens on localhost:9100 so we will add this as a scrape job to prometheus.yml:

```
sudo nano /etc/prometheus/prometheus.yml
```

Use the cursor keys to move to the bottom of the file to the `scrape_configs:` section.

Add the following scrape job to the bottom of the file (take care to preserve the formatting, or Prometheus will throw an error when trying to read it)

      - job_name: 'node_exporter'
        static_configs:
          - targets: ['localhost:9100']

When the text is added, your prometheus.yml should look something like the picture below:

![](https://i.imgur.com/kAyjrJr.png)

press `ctrl+x`. Nano will ask if you want to write your changes. Press `y` and then press `enter` to write the file.

### Create a systemd configuration file
We want to run Prometheus node_exporter as a systemd service so that it will automatically restart itself if it crashes, and to make it easy to start and stop the Prometheus node_exporter service.

    sudo tee /etc/systemd/system/node_exporter.service<<EOF

    [Unit]
    Description=node_exporter
    Documentation=https://prometheus.io/docs/introduction/overview/
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Type=simple
    User=node_exporter
    Group=node_exporter
    ExecReload=/bin/kill -HUP $MAINPID
    ExecStart=/usr/local/bin/node_exporter
    
    SyslogIdentifier=node_exporter
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    EOF

Set the correct ownership for the new files

```
sudo chown -R node_exporter:node_exporter /usr/local/bin/node_exporter
```

Reload the systemd daemon to include the new service configuration
```
sudo systemctl daemon-reload
```
Restart the Prometheus service so it registers the changes to prometheus.yml
```
sudo systemctl restart prometheus.service
```
Start the Prometheus node_exporter service
```
sudo systemctl start node_exporter.service
```
Check that Prometheus node_exporter is running as expected
```
sudo systemctl status node_exporter.service
```

The output should be similar to the screenshot below, as long as the service shows active rather than failed/stopped, it is likely to be running as required:

![](https://i.imgur.com/W7lnMdm.png)

Enable the Prometheus node_exporter service so it will restart if it fails
```
sudo systemctl enable node_exporter.service
```
## Testing

Open a web browser on a device with access to your server. Navigate to http://<server_ip>:9090/targets

You should be presented with the Prometheus web interface and some information showing if the prometheus and node_exporter endpoints are working correctly or not:

![](https://i.imgur.com/M59yy5K.png)

If everything appears to be ok, you can navigate to http://<server_ip>:9090/metrics and test run some metrics by selecting a metric in the dropdown and clicking 'Execute.' If a value is returned, your Prometheus install is working as intended.

![](https://i.imgur.com/K2qflOh.png)

## Dashboards - Grafana

In order to build dashboards that use the data from your Prometheus install a Grafana deployment is required. Please see the other guides for how to install a Grafana instance and import node_exporter dashboards.
