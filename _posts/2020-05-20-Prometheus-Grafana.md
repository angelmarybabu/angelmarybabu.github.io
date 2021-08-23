---
layout: post
title: How to install and configure Prometheus as well as Grafana for an EC2 instance?
author: Angel Mary Babu
date: 2020-05-20 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops, ]
math: true
mermaid: true
description: Install and configure Prometheus

---



Prometheus is one of the widely accepted open source monitoring tools and it takes metrics from the client machine using the node exporter. It displays them using Grafana which gives more visibility and customization..



## Installation



Now we have a basic understanding of Prometheus and Grafana. So let’s start to set up in an EC2 instance. Here I am taking an Ubuntu 20 server as a log server. 

##### Node Exporter 

We have to install node exporter on our host machines and it can be download using the following steps;
```sh
wget https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-amd64.tar.gz
```

Extract the downloaded archive

```sh
tar -xf node_exporter-0.15.2.linux-amd64.tar.gz
```

Move the node_exporter binary to /usr/local/bin:

```sh
sudo mv node_exporter-0.15.2.linux-amd64/node_exporter /usr/local/bin
```

Remove the archive file

```sh
rm -r node_exporter-0.15.2.linux-amd64.tar.gz
```
We will create users and service files for node_exporter.
```sh
sudo useradd -rs /bin/false node_exporter
```

Then, we will create a systemd unit file so that node_exporter can be started at boot. 
```sh
vi /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Now we can restart and reload the service,
```sh
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

##### Prometheus in the log server,

Download in a directory of your server with the following command,
```sh
wget https://github.com/prometheus/prometheus/releases/download/v2.1.0/prometheus-2.1.0.linux-amd64.tar.gz
```
Extract it.
```sh
tar -xf prometheus-2.1.0.linux-amd64.tar.gz
```

Let’s move the binaries to /usr/local/bin:

```sh
sudo mv prometheus-2.1.0.linux-amd64/prometheus prometheus-2.1.0.linux-amd64/promtool /usr/local/bin
```
Now, we are going to create directories for configuration files and other prometheus data.

```sh
sudo mkdir /etc/prometheus /var/lib/prometheus
```
Then, we have to move the configuration files to the directory from the present directory to the created directories

```sh
sudo mv prometheus-2.1.0.linux-amd64/consoles prometheus-2.1.0.linux-amd64/console_libraries /etc/prometheus
```

Now, we can remove the archive file which we no longer needed. 
```sh
rm -r prometheus-2.1.0.linux-amd64.tar.gz
```
##### Configuring Prometheus


Once we have installed Prometheus, we have to configure Prometheus to let it know about the HTTP endpoints it should monitor. Now we are going to add our client machine IPs in `/etc/hosts` using the following format;

```sh
x.x.x.x host-machine-1
x.x.x.x host-machine-2
```

Now we will configure the configuration file `/etc/prometheus/prometheus.yml` as following,

```sh
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100','host-machine-1:9100','host-machine-2:9100']
```

Here we have to create the necessary users and permission for the Prometheus user,
```
sudo useradd -rs /bin/false prometheus
sudo chown -R prometheus: /etc/prometheus /var/lib/prometheus
```

Also, we will create a systemd unit file in `/etc/systemd/system/prometheus.service` with the following contents :
```
[Unit]
Description=Prometheus
After=network.target

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
```

Last, we have to reload and start the services to make the changes in effect,
```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```


Now the prometheus  web UI can be accessed using the following URLt `http://<your_server_IP>:9090/`

##### Grafana

First, Install Grafana on our log instance.
```
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.0.4_amd64.deb
sudo apt-get install -y adduser libfontconfig
sudo dpkg -i grafana_5.0.4_amd64.deb
```
Now, enable and automatic restart on boot,

```
sudo systemctl daemon-reload && sudo systemctl enable grafana-server && sudo systemctl start grafana-server.service
```

Grafana is running now, and we can connect to it at `http://your.server.ip:3000`. The default user and password is `admin / admin`.

Now you have to create a Prometheus data source:

- Click on the Grafana logo to open the sidebar.
- Click on “Data Sources” in the sidebar.
- Choose “Add New”.
- Select “Prometheus” as the data source.
- Set the Prometheus server URL (in our case: http://localhost:9090/)
- Click “Add” to test the connection and to save the new data source.


You can change the password of the Grafana from the Dashboard,

- Click on the user icon at the bottom of the dashboard
- Click on Preference
- Enter Old password and New password
- Click change password


To import the Grafana dashboard to the solution:

- On the Create tab, select Import.
 [Sample ](https://grafana.com/grafana/dashboards/405 )
- Paste the ID (`405` ) of the dashboard you want to import and click Load.
- Select the Data Source as Prometheus and click Import.

Prometheus uses port `9090` and node_exporter uses port `9100` whereas Grafana uses port `3000`. ( We should whitelist the IP address of the Prometheus server with port 9100 in the `Security Group` ) 


For better visibility, we can set up an Nginx proxy.


Install Nginx.

```
sudo apt install nginx
cd /etc/nginx/sites-enabled
```


Create a new Nginx configuration for Prometheus

```
vim prometheus.conf
```

And copy/paste the example below

```
server {
    listen 80;
    listen [::]:80;
    server_name  domain-name.com;

    auth_basic           	"Restricted Access!";
    auth_basic_user_file 	/etc/nginx/conf.d/.htpasswd; 


    location /grafana/ {
        auth_basic off;
        proxy_pass           http://localhost:3000/;
    }
  
    location /prometheus/ {
        proxy_pass http://127.0.0.1:9090/;
        }
}
```
Save and test the new configuration has no errors

```
nginx -t
```

Restart Nginx

```
sudo service nginx restart
sudo service nginx status
```
##### To configure basic authentication:


Install apache2-utils or httpd-tools
```
yum install httpd-tools		[RHEL/CentOS]
sudo apt install apache2-utils	[Debian/Ubuntu]
```
Next, run htpasswd command below to create the password file with the first user. The -c option is used to specify the passwd file, once you hit [Enter], you will be asked to enter the user password.
```
htpasswd -c /etc/nginx/conf.d/.htpasswd admin
```

Now we have to make the following changes in the Grafana configuration file and in the Prometheus configuration file to work the Nginx proxy.


Add the following in the Grafan ini file

```
vim ​​/etc/grafana/grafana.ini


​​root_url = http://localhost:3000/grafana/ 
```

Add as following in the prometheus file
```
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.external-url=http://domain-name.com/prometheus/ \
    --web.route-prefix="/"

[Install]
WantedBy=multi-user.target
```
Restart the services to make the changes in effect.

##### To install SSL

Install Certbot and it’s Nginx plugin with apt:
```
sudo apt install certbot python3-certbot-nginx
```
to obtain SSL,

```
sudo certbot --nginx -d example.com -d www.example.com
```

Choose the option [2]
Sample:
```
Output
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

That’s it!!!
