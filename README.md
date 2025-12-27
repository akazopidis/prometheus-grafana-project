# prometheus-grafana-project

//help
net stop Grafana | net start Grafana


--Start Prometheus container using Docker--

1)
cd ../prometheus

2)
docker volume create prometheus-data

3)
docker run --name prometheus \
  -p 9090:9090 \
  -d \
  -v "C:\Users\cishelpdesk\Downloads\prometheus-3.8.1.windows-amd64\prometheus-3.8.1.windows-amd64\prometheus.yml:/etc/prometheus/prometheus.yml" \
  -v prometheus-data:/prometheus \
  prom/prometheus

--Setup Grafana using docker--
//Create persistent volume for your data

1)
docker volume create grafana-storage

2)
docker run -d -p 3000:3000 --name=grafana \
  --volume grafana-storage:/var/lib/grafana \
  grafana/grafana-enterprise

3) login to Grafana credentials admin admin


--prometheus custom yml--
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  - job_name: "windows_exporter"
    static_configs:
      - targets: ["host.docker.internal:9182"]
        labels:
          app: "windows"

//notes
Twra to docker compose:
Compose Project → multiple Containers → each Container uses an Image

1)Go to the folder that contains your compose.yml
cd ../prometheus-grafana-project
2)
docker compose up -d // start the stack
3)
docker compose ps // verify its running
4)
dokcer images //You’ll see two services/containers.
5)
//View logs if something looks wrong
docker compose logs -f prometheus
docker compose logs -f Grafana
6)
docker compose down //stop containers
OR
docker compose down -v //Stop + delete data volumes (wipes Prometheus + Grafana data):
