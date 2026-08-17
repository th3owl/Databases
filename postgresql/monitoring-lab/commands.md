PSQL MONITORING LAB - CLEAN COMMANDS ONLY

File: /Users/rrsolomo/Documents/PSQL/psql_monitoring_lab_commands.txt

============================================================
1. MAC HOST - CREATE DIRECTORIES AND START VMS
============================================================

cd /Users/rrsolomo/Documents/PSQL
mkdir -p database monitoring

cd /Users/rrsolomo/Documents/PSQL/database
vagrant up
vagrant ssh

cd /Users/rrsolomo/Documents/PSQL/monitoring
vagrant up
vagrant ssh

============================================================
2. MAC HOST - DATABASE VAGRANTFILE CONTENT
============================================================

cd /Users/rrsolomo/Documents/PSQL/database

cat > Vagrantfile <<'EOF'
Vagrant.configure("2") do |config|
  config.vm.box = "bento/almalinux-9"
  config.vm.box_architecture = "arm64"
  config.vm.hostname = "postgres-server"
  config.vm.network "private_network", ip: "192.168.56.16"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "postgres-server"
    vb.cpus = 4
    vb.memory = 8192
  end
end
EOF

============================================================
3. MAC HOST - OBSERVER VAGRANTFILE CONTENT
============================================================

cd /Users/rrsolomo/Documents/PSQL/monitoring

cat > Vagrantfile <<'EOF'
Vagrant.configure("2") do |config|
  config.vm.box = "bento/almalinux-9"
  config.vm.box_architecture = "arm64"
  config.vm.hostname = "observer"
  config.vm.network "private_network", ip: "192.168.56.20"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "observer"
    vb.cpus = 4
    vb.memory = 8192
  end
end
EOF

============================================================
4. POSTGRES-SERVER - INSTALL POSTGRESQL 16
============================================================

sudo -i

dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-aarch64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16-contrib

/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl enable --now postgresql-16

sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "SHOW data_directory;"

============================================================
5. POSTGRES-SERVER - ENABLE REMOTE ACCESS
============================================================

PGDATA=/var/lib/pgsql/16/data

echo "listen_addresses = '*'" >> "$PGDATA/postgresql.conf"
echo "host all all 0.0.0.0/0 scram-sha-256" >> "$PGDATA/pg_hba.conf"

systemctl reload postgresql-16
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --reload

============================================================
6. POSTGRES-SERVER - ENABLE POSTGRESQL LOGGING
============================================================

sudo -u postgres psql <<'SQL'
ALTER SYSTEM SET logging_collector = 'on';
ALTER SYSTEM SET log_directory = 'log';
ALTER SYSTEM SET log_filename = 'postgresql-%a.log';
ALTER SYSTEM SET log_min_messages = 'warning';
SELECT pg_reload_conf();
SQL

systemctl restart postgresql-16

============================================================
7. POSTGRES-SERVER - CREATE POSTGRES_EXPORTER USER
============================================================

sudo -u postgres psql <<'SQL'
CREATE USER exporter WITH PASSWORD 'exporter_password';
GRANT pg_monitor TO exporter;
SQL

============================================================
8. POSTGRES-SERVER - INSTALL NODE_EXPORTER
============================================================

VER=1.8.2
curl -fsSL -o /tmp/node_exporter.tar.gz "https://github.com/prometheus/node_exporter/releases/download/v${VER}/node_exporter-${VER}.linux-arm64.tar.gz"

tar xzf /tmp/node_exporter.tar.gz -C /tmp
cp /tmp/node_exporter-${VER}.linux-arm64/node_exporter /usr/local/bin/
chmod +x /usr/local/bin/node_exporter

cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
curl -s http://localhost:9100/metrics | head -3

============================================================
9. POSTGRES-SERVER - INSTALL POSTGRES_EXPORTER
============================================================

VER=0.18.1
curl -fsSL -o /tmp/postgres_exporter.tar.gz "https://github.com/prometheus-community/postgres_exporter/releases/download/v${VER}/postgres_exporter-${VER}.linux-arm64.tar.gz"

tar xzf /tmp/postgres_exporter.tar.gz -C /tmp
cp /tmp/postgres_exporter-${VER}.linux-arm64/postgres_exporter /usr/local/bin/
chmod +x /usr/local/bin/postgres_exporter

cat > /etc/postgres_exporter.env <<'EOF'
DATA_SOURCE_NAME="postgresql://exporter:exporter_password@localhost:5432/postgres?sslmode=disable"
EOF

chmod 600 /etc/postgres_exporter.env

cat > /etc/systemd/system/postgres_exporter.service <<'EOF'
[Unit]
Description=Prometheus PostgreSQL Exporter
After=network.target

[Service]
User=postgres
Group=postgres
EnvironmentFile=/etc/postgres_exporter.env
ExecStart=/usr/local/bin/postgres_exporter
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now postgres_exporter
firewall-cmd --permanent --add-port=9187/tcp
firewall-cmd --reload
curl -s http://localhost:9187/metrics | grep '^pg_up'

============================================================
10. POSTGRES-SERVER - INSTALL GRAFANA ALLOY
============================================================

dnf install -y https://github.com/grafana/alloy/releases/download/v1.17.1/alloy-1.17.1-1.arm64.rpm
alloy --version

cat > /etc/alloy/config.alloy <<'EOF'
// PostgreSQL log files
loki.source.file "postgresql" {
  targets = [{
    __path__ = "/var/lib/pgsql/16/data/log/postgresql-*.log",
    job      = "postgresql",
    host     = "postgres-server",
  }]

  forward_to    = [loki.write.monitoring.receiver]
  tail_from_end = true

  file_match {
    enabled     = true
    sync_period = "10s"
  }
}

// systemd journal labels
loki.relabel "journal" {
  forward_to = []

  rule {
    source_labels = ["__journal__systemd_unit"]
    target_label  = "unit"
  }

  rule {
    source_labels = ["__journal_priority_keyword"]
    target_label  = "level"
  }
}

// systemd journal
loki.source.journal "systemd" {
  forward_to    = [loki.write.monitoring.receiver]
  relabel_rules = loki.relabel.journal.rules
  max_age       = "12h"
  labels        = { job = "systemd", host = "postgres-server" }
}

// Loki endpoint on observer
loki.write "monitoring" {
  endpoint {
    url = "http://192.168.56.20:3100/loki/api/v1/push"
  }
}
EOF

usermod -aG adm,systemd-journal,postgres alloy

setfacl -m u:alloy:--x /var/lib/pgsql /var/lib/pgsql/16 /var/lib/pgsql/16/data
setfacl -m u:alloy:r-x /var/lib/pgsql/16/data/log
setfacl -d -m u:alloy:r-x /var/lib/pgsql/16/data/log
setfacl -m u:alloy:r /var/lib/pgsql/16/data/log/postgresql-*.log
setfacl -d -m u:alloy:r /var/lib/pgsql/16/data/log

sudo -u alloy ls /var/lib/pgsql/16/data/log/
alloy fmt /etc/alloy/config.alloy
systemctl reset-failed alloy
systemctl enable --now alloy
systemctl restart alloy
curl -s http://127.0.0.1:12345/-/healthy

============================================================
11. OBSERVER - INSTALL DOCKER
============================================================

sudo -i

dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

docker --version
docker compose version

mkdir -p /root/monitoring/prometheus/rules
mkdir -p /root/monitoring/loki
mkdir -p /root/monitoring/alertmanager
mkdir -p /root/monitoring/grafana/provisioning/datasources
cd /root/monitoring

============================================================
12. OBSERVER - CREATE ENV FILE
============================================================

cat > /root/monitoring/.env <<'EOF'
DB_HOST=192.168.56.16
DB_STANZA=prod01
GRAFANA_ADMIN_PASSWORD=Admin@123
PGADMIN_EMAIL=admin@example.com
PGADMIN_PASSWORD=pgadmin_password
EOF

============================================================
13. OBSERVER - CREATE PROMETHEUS CONFIG
============================================================

cat > /root/monitoring/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'postgresql'
    static_configs:
      - targets: ['192.168.56.16:9187']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.56.16:9100']
EOF

============================================================
14. OBSERVER - CREATE ALERT RULES
============================================================

cat > /root/monitoring/prometheus/rules/alerts.yml <<'EOF'
groups:
- name: postgresql_alerts
  rules:
  - alert: PostgreSQLDown
    expr: pg_up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "PostgreSQL is down on {{ $labels.instance }}"

- name: node_alerts
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Target {{ $labels.job }} down on {{ $labels.instance }}"
EOF

============================================================
15. OBSERVER - CREATE DOCKER COMPOSE FILE
============================================================

cat > /root/monitoring/docker-compose.yml <<'EOF'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-Admin@123}
    volumes:
      - grafana_data:/var/lib/grafana

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    restart: unless-stopped
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL:-admin@example.com}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD:-pgadmin_password}
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  prometheus_data:
  grafana_data:
  pgadmin_data:
EOF

cd /root/monitoring
docker compose up -d
docker compose ps

============================================================
16. OPTIONAL - SET POSTGRES SUPERUSER PASSWORD FOR PGADMIN
============================================================

sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'postgres_secret';"

============================================================
17. POSTGRES-SERVER HEALTH CHECKS
============================================================

systemctl is-active postgresql-16 node_exporter postgres_exporter alloy
curl -s http://localhost:9100/metrics | head -3
curl -s http://localhost:9187/metrics | grep '^pg_up'
curl -s http://127.0.0.1:12345/-/healthy

sudo -u postgres psql -c "DO \$\$ BEGIN RAISE WARNING 'loki test from postgres-server'; END \$\$;"

============================================================
18. OBSERVER HEALTH CHECKS
============================================================

cd /root/monitoring
docker compose ps

curl -s http://localhost:9093/-/healthy
curl -s http://localhost:3100/ready
curl -s "http://localhost:3100/loki/api/v1/labels"

curl -G "http://192.168.56.20:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="postgresql"}' \
  --data-urlencode 'limit=20' \
  --data-urlencode 'since=30m'

curl -G "http://192.168.56.20:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="systemd"}' \
  --data-urlencode 'limit=20' \
  --data-urlencode 'since=30m'

============================================================
19. BROWSER URLS
============================================================

Prometheus:
http://192.168.56.20:9090

Prometheus targets:
http://192.168.56.20:9090/targets

Grafana:
http://192.168.56.20:3000

Alertmanager:
http://192.168.56.20:9093

Loki health:
http://192.168.56.20:3100/ready

pgAdmin:
http://192.168.56.20:5050

============================================================
20. GRAFANA DATASOURCE VALUES
============================================================

Prometheus datasource URL:
http://prometheus:9090

Loki datasource URL:
http://loki:3100

============================================================
21. GRAFANA EXPLORE TEST QUERIES
============================================================

up

pg_up

{job="postgresql"}

{job="systemd"}

============================================================
22. PGADMIN LOGIN AND SERVER REGISTRATION VALUES
============================================================

pgAdmin URL:
http://192.168.56.20:5050

pgAdmin email:
admin@example.com

pgAdmin password:
pgadmin_password

pgAdmin server registration:
Name: postgres-server
Host: 192.168.56.16
Port: 5432
Maintenance database: postgres
Username: postgres
Password: postgres_secret
