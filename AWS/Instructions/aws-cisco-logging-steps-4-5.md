# AWS Cisco Logging Dashboard — Steps 4 & 5 (Docker stack: Loki + Promtail + Grafana + Prometheus)

These steps assume **Steps 2 & 3** are complete and your EC2 host is receiving:
- Syslog into `/var/log/cisco-lab.log`
- (Optional) SNMP traps into `/var/log/snmptraps.log`

---

## Step 4 — Install Docker and Docker Compose v2

### 4.1 Install Docker
```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Log out and back in so the group change applies:

```bash
exit
```

SSH back into the instance, then verify Docker works:

```bash
docker ps
```

### 4.2 Install Docker Compose v2 plugin (manual method)
Some Amazon Linux repos do not include `docker-compose-plugin`. This method is consistent:

```bash
sudo mkdir -p /usr/lib/docker/cli-plugins
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64"   -o /usr/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/lib/docker/cli-plugins/docker-compose

docker compose version
```

---

## Step 5 — Create the log UI stack (Loki + Promtail + Grafana + Prometheus)

### 5.1 Create a working folder
```bash
mkdir -p ~/log-ui
cd ~/log-ui
```

### 5.2 Create Promtail config (logs + traps + interface-up metric)
Create `promtail-config.yml`:

```bash
nano promtail-config.yml
```

Paste:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/promtail-positions.yml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Cisco syslog file
  - job_name: cisco-syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: cisco-syslog
          __path__: /var/log/cisco-lab.log

    # Extract per-device/per-interface "up/down" events into a metric
    # This avoids collisions when multiple devices have the same interface names.
    pipeline_stages:
      - match:
          selector: '{job="cisco-syslog"} |~ "Interface .* changed state to (up|down)"'
          stages:
            - regex:
                # Adjust if your syslog header differs.
                expression: '^\w{3}\s+\d+\s+\d+:\d+:\d+\s+(?P<device>\S+).*Interface (?P<iface>[^,]+), changed state to (?P<state>up|down)'
            - labels:
                device:
                iface:
            - template:
                source: if_up
                template: '{{ if eq .state "up" }}1{{ else }}0{{ end }}'
            - metrics:
                interface_up:
                  type: Gauge
                  prefix: cisco_
                  description: "Cisco interface up (1) / down (0) derived from syslog"
                  source: if_up
                  config:
                    action: set

  # SNMP trap log file (optional)
  - job_name: snmp-traps
    static_configs:
      - targets: [localhost]
        labels:
          job: snmp-traps
          __path__: /var/log/snmptraps.log
```

Notes:
- The uptime % is computed later in Grafana/Prometheus from `cisco_interface_up{device=...,iface=...}`.
- If your syslog header format differs, the regex may need a minor tweak.

### 5.3 Create Prometheus config (scrape Promtail metrics)
Create `prometheus.yml`:

```bash
nano prometheus.yml
```

Paste:

```yaml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: promtail
    static_configs:
      - targets: ["promtail:9080"]
```

### 5.4 Create docker-compose.yml
Create `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Paste the following and replace `<EC2_PUBLIC_IP>` with your instance public IP:

```yaml
version: "3.8"

services:
  loki:
    image: grafana/loki:latest
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=ChangeMe123
      - GF_SERVER_DOMAIN=<EC2_PUBLIC_IP>
      - GF_SERVER_ROOT_URL=http://<EC2_PUBLIC_IP>:3000/
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    restart: unless-stopped

volumes:
  prometheus-data:
```

### 5.5 Start the stack
```bash
docker compose up -d
docker ps
```

You should see containers for:
- grafana
- loki
- promtail
- prometheus

### 5.6 Quick health checks
Promtail metrics endpoint (from EC2 host):

```bash
curl -s http://127.0.0.1:9080/metrics | head
```

Loki ready endpoint (from EC2 host):

```bash
curl -s http://127.0.0.1:3100/ready
```

Grafana reachable (from EC2 host):

```bash
curl -I http://127.0.0.1:3000 | head
```

If the syslog/trap files do not exist yet, that’s fine; they will populate once Cisco sends messages.
