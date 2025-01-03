# Monitor containers


## Docker engine metrics

1. Create/edit `/etc/docker/daemon.json` file:
    ```shell
    vi /etc/docker/daemon.json
    ```
2. Add below given lines in it
    ```json
      {
       "metrics-addr" : "127.0.0.1:9323",
       "experimental" : true
      }
    ```
3. Restart docker service
    ```shell
    systemctl restart docker
    ```
4. Verify if docker is exporting the metrics now
    ```shell
    curl localhost:9323/metrics
    ```
**prometheus.yml**
Add docker to scrape configs:
````yaml
scrape_configs:
  - job_name: "docker"
    static_configs:
      - targets: ['127.0.0.1:9323'] # @Ip and port specified in metrics-addr field in daemon.json
````
## Containers Metrics
To get the container-level metrics, a `cAdvisor` container needs to run on the docker host. The `cAdvisor` can collect and expose metrics

**Docker-compose.yml :**
````yaml
version: '3.4'
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    container_name: cadvisor
    privileged: true
    devices:
      - "/dev/kmsg:/dev/kmsg"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - 8070:8080
````
Add below given lines under `scrape_configs`:
````yaml
  - job_name: "cadvisor"
    static_configs:
      - targets: ["localhost:8070"]
````
Restart prometheus service:
````shell
systemctl restart prometheus
````