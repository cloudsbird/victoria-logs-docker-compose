# Victoria Logs Docker Compose

A production-ready Docker Compose configuration for deploying [Victoria Logs](https://docs.victoriametrics.com/victorialogs/) — a fast, cost-effective log storage and search engine optimized for constrained environments, with [Grafana](https://grafana.com/) for visualization and querying.

## Overview

Victoria Logs is a time-series log storage built by VictoriaMetrics that provides:
- High compression ratio and low resource consumption
- Fast log search and filtering
- Integration with popular log collectors (Fluentd, Logstash, Promtail, etc.)
- JSON-structured logging support
- Cost-effective storage for 2C/8GB+ environments

This setup includes:
- **Victoria Logs**: Log storage and search engine
- **Grafana**: Web-based visualization and query interface

## Prerequisites

- Docker and Docker Compose installed
- At least 2 CPU cores and 10GB RAM recommended (for both Victoria Logs and Grafana)
- Sufficient disk space for log storage (adjust retention period as needed)

## Quick Start

1. **Clone the repository:**
   ```bash
   git clone <repo-url>
   cd victoria-logs-docker-compose
   ```

2. **Start the container:**
   ```bash
   docker compose up -d
   ```

3. **Verify it's running:**
   ```bash
   docker compose ps
   docker compose logs
   ```

4. **Access the services:**
   - **Grafana**: http://localhost:3000 (username: `admin`, password: `admin`)
   - **Victoria Logs API**: http://localhost:9428

## Configuration

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `VM_MEMORY_ALLOWED_PERCENT` | 75 | Max memory threshold before optimizations kick in |

### Command-line Flags

| Flag | Value | Purpose |
|------|-------|---------|
| `-storageDataPath` | `/victoria-logs-data` | Directory for log storage |
| `-retentionPeriod` | `7d` | How long logs are retained |
| `-search.maxConcurrentRequests` | `4` | Concurrent query limit (resource optimization) |
| `-loggerFormat` | `json` | Log format (json, default) |

### Resource Limits

Configured for constrained environments (2C/8GB):

```yaml
CPU:      2 cores (hard limit)
Memory:   6GB (hard limit) / 2GB (reservation)
```

These can be adjusted in the `deploy.resources` section if you have different hardware.

## Port Configuration

The following ports are exposed on localhost only for security:

| Service | Port | Binding | Purpose |
|---------|------|---------|---------|
| Victoria Logs | 9428 | 127.0.0.1 | Log ingestion and query API |
| Grafana | 3000 | 127.0.0.1 | Web UI for visualization |

**Security Note:** Ports are bound to `127.0.0.1` (localhost only), which means the services are only accessible from the host machine. This prevents unauthorized external access. If you need to access these services from other machines, you'll need to modify the port bindings in `docker-compose.yml` or use a reverse proxy.

## Data Persistence

Data is stored in Docker named volumes, which persist across container restarts:

- `victoria-logs-data`: Log storage data
- `grafana-data`: Grafana dashboards and configuration

```bash
# Inspect volumes
docker volume inspect victoria-logs-docker-compose_victoria-logs-data
docker volume inspect victoria-logs-docker-compose_grafana-data

# Remove volumes (WARNING: deletes all data)
docker volume rm victoria-logs-docker-compose_victoria-logs-data
docker volume rm victoria-logs-docker-compose_grafana-data
```

## Health Checks

A health check is configured to monitor the container:

```bash
docker compose ps  # Shows health status
```

Health check details:
- **Endpoint:** `http://localhost:9428/health`
- **Interval:** 30 seconds
- **Timeout:** 5 seconds
- **Retries:** 3

## Logging & Rotation

Container logs are rotated to prevent disk fill:

```yaml
max-size: "50m"   # Max single log file
max-file: "3"     # Keep 3 rotated files
```

View logs:
```bash
docker compose logs -f victoria-logs
docker compose logs -f grafana
```

## Grafana Setup

### Initial Login

1. Access Grafana at http://localhost:3000
2. Login with default credentials:
   - **Username:** `admin`
   - **Password:** `admin`
3. You'll be prompted to change the password on first login

### Adding Victoria Logs as a Data Source

1. In Grafana, go to **Configuration** (gear icon) → **Data Sources**
2. Click **Add data source**
3. Search for and select **Loki** (Victoria Logs is compatible with Loki API)
4. Configure the data source:
   - **Name:** Victoria Logs
   - **URL:** `http://victoria-logs:9428`
   - **Access:** Server (default)
5. Click **Save & Test**

**Note:** Victoria Logs implements the Loki API, so you can use the Loki data source in Grafana. The queries will be sent to Victoria Logs' `/select/logsql/query` endpoint automatically.

### Creating Dashboards

Once the data source is configured, you can:
- Use the **Explore** tab to query logs interactively
- Create custom dashboards to visualize log patterns
- Set up alerts based on log queries

### Grafana Configuration

Default environment variables (can be customized in `docker-compose.yml`):

| Variable | Default | Purpose |
|----------|---------|---------|
| `GF_SECURITY_ADMIN_USER` | `admin` | Admin username |
| `GF_SECURITY_ADMIN_PASSWORD` | `admin` | Admin password (change this!) |
| `GF_USERS_ALLOW_SIGN_UP` | `false` | Disable user registration |
| `GF_LOG_LEVEL` | `info` | Logging level |

**Security Note:** Change the default admin password in production by modifying the environment variables in `docker-compose.yml`:

```yaml
environment:
  - GF_SECURITY_ADMIN_USER=admin
  - GF_SECURITY_ADMIN_PASSWORD=<your-secure-password>
```

Or use Docker secrets or environment variable files for better security.

## Customization

### Increase Retention Period

Edit `docker-compose.yml` and change `-retentionPeriod`:

```yaml
- "-retentionPeriod=30d"  # Keep logs for 30 days
```

### Adjust Memory Limits

For systems with more resources (recommended when running both services):

```yaml
deploy:
  resources:
    limits:
      memory: 8G        # Increase from 6G
    reservations:
      memory: 4G        # Increase from 2G
```

**Note:** When running both Victoria Logs and Grafana, consider increasing available memory to at least 10GB total system RAM.

### Enable External Access

By default, the ports are bound to localhost only (`127.0.0.1`) for security. To allow access from other machines on your network, modify the port bindings in `docker-compose.yml`:

```yaml
# Change from localhost-only binding:
ports:
  - "127.0.0.1:9428:9428"

# To all interfaces binding:
ports:
  - "9428:9428"
```

**Warning:** Exposing these services to external networks may pose security risks. Consider using a reverse proxy with authentication or a VPN for remote access.

## Troubleshooting

### Container exits/restarts frequently

Check logs for memory issues:
```bash
docker compose logs victoria-logs | grep -i "oom\|memory"
```

**Solution:** Increase memory limits or reduce retention period.

### Out of Disk Space

Monitor volume:
```bash
docker volume inspect victoria-logs-docker-compose_victoria-logs-data
df -h /var/lib/docker/volumes/  # Check actual disk usage
```

**Solution:** Reduce retention period, increase disk space, or implement log rotation/cleanup.

### API Port Not Accessible

Verify port is uncommented:
```bash
docker compose config | grep ports -A 2
```

Check if it's listening:
```bash
docker compose exec victoria-logs netstat -tuln | grep 9428
```

### Health Check Failing

Test connectivity:
```bash
docker compose exec victoria-logs wget -O- http://localhost:9428/health
```

## Integration with Log Collectors

To send logs to Victoria Logs, configure your log collector:

### Fluentd
```
<match **>
  @type http
  endpoint http://victoria-logs:9428/insert/logs
  json_array true
</match>
```

### Promtail (Loki client)
```yaml
clients:
  - url: http://victoria-logs:9428/insert/loki/api/v1/push
```

## Maintenance

### Backup Data

```bash
docker run --rm -v victoria-logs-docker-compose_victoria-logs-data:/data \
  -v /backup:/backup ubuntu tar czf /backup/victoria-logs-backup.tar.gz -C /data .
```

### Clean Up Old Logs

Victoria Logs automatically purges old logs based on retention period. No manual cleanup needed.

### Update Victoria Logs Version

1. Edit `docker-compose.yml` and update the image version
2. Pull the new image: `docker compose pull`
3. Restart: `docker compose up -d`

## Performance Tuning

For larger deployments or higher log volumes:

1. **Increase concurrent requests:** `-search.maxConcurrentRequests=8`
2. **Adjust memory threshold:** `-VM_MEMORY_ALLOWED_PERCENT=90`
3. **Enable caching:** Add `-cacheExpireDuration=1h`

See [Victoria Logs docs](https://docs.victoriametrics.com/victorialogs/) for more options.

## License & Attribution

This Docker Compose configuration is for deploying [Victoria Logs](https://github.com/VictoriaMetrics/VictoriaMetrics) by VictoriaMetrics.

## Support

For issues with Victoria Logs itself, see:
- [Official Documentation](https://docs.victoriametrics.com/victorialogs/)
- [GitHub Issues](https://github.com/VictoriaMetrics/VictoriaMetrics/issues)
