# PostgreSQL Monitoring Lab

Two-VM PostgreSQL observability lab using VirtualBox and Vagrant on Apple Silicon Mac.

## Architecture

| Host | IP | Role |
|---|---:|---|
| `postgres-server` | `192.168.56.16` | PostgreSQL 16, `node_exporter`, `postgres_exporter`, Grafana Alloy |
| `observer` | `192.168.56.20` | Prometheus, Grafana, Loki, Alertmanager, pgAdmin |

## Data Flow

| Flow | Path | Meaning |
|---|---|---|
| PostgreSQL metrics | `postgres_exporter -> Prometheus -> Grafana` | PostgreSQL health and activity metrics |
| OS metrics | `node_exporter -> Prometheus -> Grafana` | CPU, memory, disk, and network metrics |
| PostgreSQL logs | `PostgreSQL logs -> Alloy -> Loki -> Grafana` | Queryable database logs |
| systemd logs | `systemd journal -> Alloy -> Loki -> Grafana` | Queryable service logs |
| Alerts | `Prometheus -> Alertmanager` | Rule-based alert flow |
| Administration | `pgAdmin -> PostgreSQL` | Browser-based PostgreSQL administration |

## Main URLs

| Service | URL |
|---|---|
| Prometheus | `http://192.168.56.20:9090` |
| Grafana | `http://192.168.56.20:3000` |
| Loki health | `http://192.168.56.20:3100/ready` |
| Alertmanager | `http://192.168.56.20:9093` |
| pgAdmin | `http://192.168.56.20:5050` |

## Placeholder Credentials

These are lab placeholders only. Change them before using the lab anywhere sensitive.

| Component | User | Password |
|---|---|---|
| pgAdmin | `admin@example.com` | `pgadmin_password` |
| PostgreSQL exporter | `exporter` | `exporter_password` |
| Grafana | `admin` | `Admin@123` |

## Files

- [Commands](commands.md)
- [Observability Stack Explained](observability-stack.md)
- [PDF Runbook](assets/psql_monitoring_lab_runbook.pdf)

## Important Notes

- `http://192.168.56.20:3100/` returning `404` is normal for Loki.
- Use `http://192.168.56.20:3100/ready` for Loki health.
- In Grafana datasource settings, use Docker service names:
  - Prometheus: `http://prometheus:9090`
  - Loki: `http://loki:3100`

