Original article can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-prometheus-on-ubuntu-16-04).

# Table of contents

- [Create service users](#create-service-users)
- [Download Prometheus](#download-prometheus)
- [Configure Prometheus](#configure-prometheus)
- [Start Prometheus](#start-prometheus)
- [Secure Prometheus](#secure-prometheus)

# Create service user

Create these two users, and use the `--no-create-home` and `--shell /bin/false` options so that these users can't log into the server.

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```


```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Before we download the Prometheus binaries, create the necessary directories for storing Prometheus' files and data.

```bash
sudo mkdir /etc/prometheus
```

```bash
sudo mkdir /var/lib/prometheus
```

Now, set the user and group ownership on the new directories to the prometheus user.

```bash
sudo chown prometheus:prometheus /etc/prometheus
```

```bash
sudo chown prometheus:prometheus /var/lib/prometheus
```

# Download Prometheus

Download and unpack the current stable version of Prometheus into your home directory. [Prometheus download page](https://prometheus.io/download/#prometheus)

Let's download `prometheus-2.45.4.linux-amd64.tar.gz`

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.45.4/prometheus-2.45.4.linux-amd64.tar.gz
```

Unpack the downloaded archive.

```bash
tar xvf prometheus-2.45.4.linux-amd64.tar.gz
```

Copy the two binaries to the `/usr/local/bin` directory.

```bash
sudo cp prometheus-2.45.4.linux-amd64/prometheus /usr/local/bin/
```

```bash
sudo cp prometheus-2.45.4.linux-amd64/promtool /usr/local/bin/
```

Set the user and group ownership on the binaries to the prometheus user.

```bash
sudo chown prometheus:prometheus /usr/local/bin/prometheus
```

```bash
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Copy the `consoles` and `console_libraries` directories to `/etc/prometheus`.

```bash
sudo cp -r prometheus-2.45.4.linux-amd64/consoles /etc/prometheus
```

```bash
sudo cp -r prometheus-2.45.4.linux-amd64/console_libraries /etc/prometheus
```

Set the user and group ownership on the directories to the prometheus user. Using the `-R` flag will ensure that ownership is set on the files inside the directory as well.

```bash
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
```

```bash
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Remove the leftover files from your home directory as they are no longer needed.

```bash
rm -rf prometheus-2.45.4.linux-amd64.tar.gz prometheus-2.45.4.linux-amd64
```

# Configure Prometheus

```
sudo nano /etc/prometheus/prometheus.yml
```

Configuration file `prometheus.yml` will contain just enough information to run Prometheus for the first time.

Set the user and group ownership on the configuration file to the prometheus user.

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

# Start Prometheus

Open a new `systemd` service file.

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Copy the following content into the file:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address="127.0.0.1:9090"

[Install]
WantedBy=multi-user.target
```

Save the file and close your text editor.

To use the newly created service, reload systemd:

```bash
sudo systemctl daemon-reload
```

Start Prometheus:

```bash
sudo systemctl start prometheus
```

Prometheus status:

```bash
sudo systemctl status prometheus
```

Enable the service to start on boot:

```bash
sudo systemctl enable prometheus
```

# Secure Prometheus

Prometheus does not include built-in authentication or any other general purpose security mechanism.

For simplicity’s sake, we’ll use Nginx to add basic HTTP authentication to our installation.

Install `apache2-utils`, which will give you access to the `htpasswd` utility for generating password files.

```bash
sudo apt-get update
sudo apt-get install apache2-utils
```

Create a password file by telling `htpasswd` where you want to store the file and which username you'd like to use for authentication.

```bash
sudo htpasswd -c /etc/nginx/.htpasswd username
```

The result of this command is a newly-created file called `.htpasswd`, located in the `/etc/nginx` directory, containing the username and a hashed version of the password you entered.

Example of Nginx configuration file:

```apacheconf
server {
    listen 80;
    server_name your_domain;

    location / {
		auth_basic "Prometheus authentication";
		auth_basic_user_file /etc/nginx/.htpasswd;
		proxy_pass http://localhost:9090;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}

    location = /robots.txt { access_log off; log_not_found off; }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Check Nginx configuration for errors using the following command:

```bash
sudo nginx -t
```

Output:

```shell
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx to incorporate all of the changes.

```bash
sudo systemctl reload nginx
```

```bash
sudo systemctl status nginx
```

Point your web browser to `http://your_server_ip` or `http://your_domain`.
