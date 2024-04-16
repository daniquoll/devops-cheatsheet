Original article can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-prometheus-on-ubuntu-16-04).

# Table of contents

- [Prometheus installation](prometheus/install-prometheus-on-ubuntu.md)
- [Create service users](#create-service-users)
- [Download Node Exporter](#download-node-exporter)
- [Start Node Exporter](#start-node-exporter)
- [Secure Node Exporter](#secure-node-exporter)
- [Configure Prometheus to scrape Node Exporter](#configure-prometheus-to-scrape-node-exporter)

# Create service user

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

# Download Node Exporter

Download the current stable version of Node Exporter into your home directory. [Prometheus download page](https://prometheus.io/download/#node_exporter)

Let's download `node_exporter-1.7.0.linux-amd64.tar.gz`

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```

Unpack the downloaded archive.

```bash
tar xvf node_exporter-1.7.0.linux-amd64.tar.gz
```

Copy the binary to the `/usr/local/bin` directory and set the user and group ownership to the node_exporter user

```bash
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin
```

```bash
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Remove the leftover files from your home directory as they are no longer needed.

```bash
rm -rf node_exporter-1.7.0.linux-amd64.tar.gz node_exporter-1.7.0.linux-amd64
```

# Start Node Exporter

Open a new `systemd` service file.

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Copy the following content into the service file:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Collectors define which metrics Node Exporter will generate. You can see Node Exporter’s complete list of collectors — including which are enabled by default and which are deprecated — in the [Node Exporter README file](https://github.com/prometheus/node_exporter/blob/master/README.md#enabled-by-default).

If you ever need to override the default list of collectors, you can use the `--collectors.enabled` flag, like:

```ini
...
ExecStart=/usr/local/bin/node_exporter --collectors.enabled meminfo,loadavg,filesystem
...
```

Save the file and close your text editor.

Finally, reload systemd to use the newly created service.

```bash
sudo systemctl daemon-reload
```

Start Node Exporter:

```bash
sudo systemctl start node_exporter
```

Node Exporter status:

```bash
sudo systemctl status node_exporter
```

Enable the service to start on boot:

```bash
sudo systemctl enable node_exporter
```

# Secure Node Exporter

Generate a hashed password using `htpasswd` from `apache2-utils`:

![htpasswd output](https://miro.medium.com/v2/resize:fit:720/format:webp/1*5lHdexdJ4WP8Gr9QZOv2bA.png)

Create file `/etc/node_exporter/config.yml`:

```bash
sudo mkdir /etc/node_exporter
```

```bash
sudo nano /etc/node_exporter/config.yml
```

Copy the following content into the file:

```yml
basic_auth_users:
    YOUR_USERNAME: YOUR_HASHED_PASSWORD
    # OTHER_USERNAME: OTHER_HASHED_PASSWORD
    ...
```

Restart Node Exporter:

```bash
sudo systemctl restart node_exporter
```

# Configure Prometheus to scrape Node Exporter

Because Prometheus only scrapes exporters which are defined in the scrape_configs portion of its configuration file, we'll need to add an entry for Node Exporter, just like we did for Prometheus itself.

Open Prometheus configuration file.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

At the end of the `scrape_configs` block, add a new entry called `node_exporter`.

```yml
scrape_configs:
  ...
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['your_server_ip:9100']
  ...
```

If you configured Node Exporter to require password you need to add `basic_auth` to the `node_exporter` job:

```yml
scrape_configs:
  ...
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['your_server_ip:9100']
    basic_auth:
      username: 'YOUR_USERNAME'
      password: 'YOUR_PASSWORD'
  ...
```

Save the file and close your text editor.

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```