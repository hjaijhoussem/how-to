# Prometheus Installation

1. [Setup Prometheus](#setup-prometheus)
   1. [Install Binary](#install-binary)
   2. [Configure Prometheus as Systemd](#configure-prometheus-as-systemd)
2. [Secure Connection between Prometheus and Node Exporter](#secure-connection-between-prometheus-and-node-exporter)
    1. [Authentication](#authentication)
    2. [Encryption (Setup TLS)](#encryption-setup-tls)

## Setup Prometheus
### Install Binary
```shell 
wget https://github.com/prometheus/prometheus/releases/download/v2.40.1/prometheus-2.40.1.linux-amd64.tar.gz
tar xvf prometheus-2.40.1.linux-amd64.tar.gz
cd prometheus-2.40.1.linux-amd64
```
**Note** by default, Prometheus doesn't come with systemd file, so `systemctl` and `service` commands does not work on it, so to execute prometheus :
````shell
  ./prometheus > /dev/null 2>&1 & 
````

### Configure Prometheus as Systemd:
1. Add Prometheus user as below
    ````shell
    useradd --no-create-home --shell /bin/false prometheus
    ````
2. Create Directories for storing prometheus config file and data
    ````shell
    mkdir /etc/prometheus
    mkdir /var/lib/prometheus
    ````
3. Change the permissions
    ````shell
    chown prometheus:prometheus /etc/prometheus
    chown prometheus:prometheus /var/lib/prometheus
    ````
4. Copy the binaries
    ````shell
    cp /root/prometheus-2.40.1.linux-amd64/prometheus /usr/local/bin/
    cp /root/prometheus-2.40.1.linux-amd64/promtool /usr/local/bin/
    ````
5. Change the ownership of binaries
    ````shell
    chown prometheus:prometheus /usr/local/bin/prometheus
    chown prometheus:prometheus /usr/local/bin/promtool
    ````
6. Copy the directories consoles and console_libraries
    ````shell
    cp -r /root/prometheus-2.40.1.linux-amd64/consoles /etc/prometheus
    cp -r /root/prometheus-2.40.1.linux-amd64/console_libraries /etc/prometheus
    ````
7. Change the ownership of directories consoles and console_libraries
    ````shell
    chown -R prometheus:prometheus /etc/prometheus/consoles
    chown -R prometheus:prometheus /etc/prometheus/console_libraries
    ````
8. Move prometheus.yml file to /etc/prometheus directory
    ````shell
    cp /root/prometheus-2.40.1.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml
    ````
9. Change the ownership of file /etc/prometheus/prometheus.yml
    ````shell
    chown prometheus:prometheus /etc/prometheus/prometheus.yml
    ````
10. Create a service for prometheus
    ```shell
    vi /etc/systemd/system/prometheus.service 
    ```
- `prometheus.service`:
    ```text
    [Unit]
    Description=Prometheus
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
    ```
11. Reload Configuration
    ```shell
    systemctl daemon-reload
    systemctl start prometheus
    systemctl enable prometheus
    systemctl status prometheus
    ```
## Secure Connection between Prometheus and Node Exporter:
### Authentication
1. Create and configure the `config.yml` file for Node Exporter:
    ```shell
    mkdir -p /etc/node_exporter
    touch /etc/node_exporter/config.yml
    chmod 700 /etc/node_exporter
    chmod 600 /etc/node_exporter/config.yml
    chown -R nodeusr:nodeusr /etc/node_exporter
   ```
2. Update the Node Exporter service configuration to use the config.yml file:
   ```shell
   sed -i 's|ExecStart=/usr/local/bin/node_exporter|ExecStart=/usr/local/bin/node_exporter --web.config=/etc/node_exporter/config.yml|' /etc/systemd/system/node_exporter.service
   ```
3. Install the Apache2 utilities package for password hashing:
    ```shell
   apt update
   apt install apache2-utils -y
   ```
4. Generate a hashed password:
   ```shell
   htpasswd -nBC 10 "" | tr -d ':\n'; echo
   ```
   Enter the password (e.g., `secret-password`) twice when prompted. This will output the hashed value of your password.
5. Edit the `/etc/node_exporter/config.yml` file and add the following lines:
    ```yaml
   basic_auth_users:
     prometheus: <hashed-password>
   ```
   Replace `<hashed-password>` with the hashed password generated earlier.
6. Reload the systemd daemon and restart the Node Exporter service:
   ```shell
   systemctl daemon-reload
   systemctl restart node_exporter
   ```
7. Update `prometheus.yml`:
   ```yaml
   basic_auth:
     username: prometheus
     password: secret-password
    ```
8. Restart Prometheus
    ```shell
    systemctl restart prometheus 
   ```
### Encryption (Setup TLS)
1. Generate the certificate and key
    ```shell
    openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"
     ```
2. Move the crt and key file under `/etc/node_exporter/` directory
    ```shell
    mv node_exporter.crt node_exporter.key /etc/node_exporter/
   ```
3. Change ownership:
    ```shell
   chown nodeusr.nodeusr /etc/node_exporter/node_exporter.key
   chown nodeusr.nodeusr /etc/node_exporter/node_exporter.crt
    ```
4. Edit `/etc/node_exporter/config.yml` file:
    ```shell
   vi /etc/node_exporter/config.yml
   ```
5. Add below lines in this file:
    ````yaml
   tls_server_config:
     cert_file: node_exporter.crt
     key_file: node_exporter.key
   ````
6. Restart node exporter service:
    ```shell
    systemctl restart node_exporter 
   ```
7. You can verify your changes using curl command:
    ```shell
   curl -u prometheus:secret-password -k https://node01:9100/metrics
   ```
8. Copy the certificate from node01 to Prometheus server
    ```shell
   scp root@node01:/etc/node_exporter/node_exporter.crt /etc/prometheus/node_exporter.crt
   ```
9. Change certificate file ownership:
    ```shell
   chown prometheus.prometheus /etc/prometheus/node_exporter.crt
   ```
10. Edit /etc/prometheus/prometheus.yml file
    ```shell
    vi /etc/prometheus/prometheus.yml
    ```
11. Add below given lines under - job_name: "nodes"
    ```yaml
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
    ```
12. Restart prometheus service
    ````shell
    systemctl restart prometheus
    ````