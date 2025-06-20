# Prometheus & Grafana DevOps Lab

A complete monitoring stack using Prometheus, Node Exporter, and Grafana with Docker Compose.

## Quick Start

1. **Prerequisites**
   - Docker & Docker Compose installed

2. **Setup**
   ```bash
   git clone <repository-url>
   cd grafana-dashboards-for-Prometheus
   docker compose up -d
   ```

3. **Access Services**
   - Prometheus: http://localhost:9090
   - Node Exporter: http://localhost:9100  
   - Grafana: http://localhost:3000 (admin/admin)

## Architecture

```
┌────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Prometheus   │◄──►│  Node Exporter  │◄──►│     Grafana     │
│   Port 9090    │    │   Port 9100     │    │   Port 3000     │
│ Scrapes Metrics│    │  Host Metrics   │    │ Visualizes Data │
└────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
[prometheus_data]        [/proc, /sys, /]        [grafana_data]
     Volume                Host Mounts              Volume
```

## Key Files

- `docker-compose.yml` - Container orchestration
- `prometheus.yml` - Prometheus configuration

## Assessment Answers

### Section 1: Architecture & Setup

**Q1: Why Node Exporter needs host mounts?**
- `/proc`, `/sys`, `/` contain system metrics
- Without mounts: no host metrics collected
- Containerized monitoring principle: isolated service with secure host access

**Q2: Network security implications**
- Custom network isolates monitoring stack
- Exposed ports risk unauthorized access
- Production fix: bind to localhost, use reverse proxy + auth

**Q3: Data persistence**
- Prometheus: time-series data storage
- Grafana: dashboard/config persistence
- Without volumes: data lost on restart

### Section 2: Metrics & Queries

**Q4: Uptime calculation**
```promql
node_time_seconds - node_boot_time_seconds
```
- Current time minus boot time = uptime
- Issue: clock drift affects accuracy
- Alternative: `time() - node_boot_time_seconds`

**Q5: Memory metrics**
```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```
- MemAvailable includes reclaimable cache/buffers
- More accurate than MemFree for Linux systems

**Q6: Filesystem usage**
```promql
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```
- Calculates percentage used
- Exclude temp filesystems: `fstype!~"tmpfs|devtmpfs"`

### Section 3: Visualization & Dashboards

**Q7: Visualization choices**
- **Stat**: Single values (uptime)
- **Time Series**: Trends over time (memory)
- **Gauge**: Current percentage with thresholds (disk)

**Q8: Threshold strategy**
- 80% standard disk warning
- Database servers: 70% warning, 85% critical
- Web servers: 80% warning, 90% critical

**Q9: Dashboard variables**
- `$job` replaces hardcoded labels
- Dynamic queries from metric labels
- Test with multiple job values

### Section 4: Production & Scalability

**Q10: Resource planning (100 servers)**
- **CPU**: 4-8 cores
- **RAM**: 16-32GB
- **Storage**: 1-2TB (30 days retention)
- **Bottlenecks**: Storage I/O, query load

**Q11: High availability**
- Multiple Prometheus instances
- Thanos for long-term storage
- Load-balanced Grafana
- Remote backups

**Q12: Security hardening**
1. Enable authentication (OAuth)
2. Bind ports to localhost
3. Add TLS encryption
4. Change default passwords
5. Use secrets management (Vault)

### Section 5: Troubleshooting & Operations

**Q13: Target DOWN debugging**
1. Check containers: `docker compose ps`
2. Verify config: `prometheus.yml`
3. Check logs: `docker logs prometheus`
4. Test connectivity: `curl http://node-exporter:9100`

**Q14: Performance optimization**
- Complex queries slow dashboards
- Use specific labels, shorter ranges
- Implement recording rules
- Monitor with `prometheus_tsdb_head_series`

**Q15: Capacity planning**
- Storage grows with metrics + retention
- Set 15-90 day retention based on needs
- Use remote storage (S3) for archives
- Balance: retention time vs storage cost

## Screenshots Included

-  Docker containers running
-  Prometheus targets (UP status)
-  Grafana dashboard panels
-  Data source configuration
-  Dashboard variables setup

## Dashboard Panels Created

1. **System Uptime** (Stat visualization)
2. **Memory Usage** (Time series chart)
3. **Disk Usage** (Gauge with thresholds)

## Configuration Files

### docker-compose.yml
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  node-exporter:
    image: prom/node-exporter:latest
    ports: ["9100:9100"]
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  grafana:
    image: grafana/grafana-oss:latest
    ports: ["3000:3000"]
    volumes:
      - grafana_data:/var/lib/grafana
```

### prometheus.yml
```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: node-exporter
    static_configs:
      - targets: ['node-exporter:9100']
```

## Learning Outcomes

- Container-based monitoring architecture
- Prometheus metric collection and queries
- Grafana dashboard creation and visualization
- Production considerations for monitoring stacks
- Security and performance optimization

---

