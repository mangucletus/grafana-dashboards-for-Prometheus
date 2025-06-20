# Prometheus & Grafana DevOps Lab

THis is a complete monitoring stack using Prometheus, Node Exporter, and Grafana with Docker Compose.

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

Node Exporter runs inside a container but needs to see the real computer's information. The `/proc`, `/sys`, and `/` folders contain all the important system data like CPU usage and memory stats. When we mount these folders, we're basically giving the container permission to read the host computer's information. 

Without these mounts, Node Exporter would only see its own tiny container world and couldn't tell us anything useful about the actual server. It's like trying to check someone's health by only looking at their shadow - you need the real thing.

**Q2: Network security implications**

The custom Docker network "prometheus-grafana" keeps our monitoring tools separate from other containers. This is good because it prevents conflicts and makes communication easier between Prometheus and Grafana.

However, we're exposing ports 9090, 9100, and 3000 to the whole internet, which is risky. Anyone could potentially access these services. In a real production environment, we should only allow local access (127.0.0.1), put everything behind a secure proxy, and add proper authentication.

**Q3: Data persistence**

Both Prometheus and Grafana need to remember things when they restart. Prometheus stores all the metrics data it collects over time, while Grafana saves our dashboard designs and user settings.

Docker volumes are like permanent storage boxes. Without them, every time we restart the containers, we lose everything - all our historical data and custom dashboards would disappear. It's like having amnesia every time you wake up.

### Section 2: Metrics & Queries

**Q4: Uptime calculation**
```promql
node_time_seconds - node_boot_time_seconds
```

This query is like checking how long someone has been awake. `node_time_seconds` tells us what time it is right now, and `node_boot_time_seconds` tells us when the computer first started up. When we subtract the start time from now, we get how long the system has been running.

The problem is that if the computer's clock gets messed up or changes, our calculation becomes wrong. A better way might be to use `time() - node_boot_time_seconds` because `time()` is more reliable.

**Q5: Memory metrics**
```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

Linux is smart about memory management. When programs aren't using all the memory, Linux uses the extra space to cache files and make things faster. The `MemFree` metric only shows completely empty memory, but `MemAvailable` shows memory that programs can actually use (including cached memory that can be freed up quickly).

Think of it like a parking lot - MemFree shows only empty spaces, but MemAvailable shows empty spaces plus spaces that people are just temporarily using and can move from quickly.

**Q6: Filesystem usage**
```promql
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

This is like calculating how full a water bottle is. We take the available space, divide it by total space to get the percentage that's empty, then subtract from 1 to get the percentage that's full.

For example, if we have 100GB total and 20GB available, then 20/100 = 0.2 (20% free), so 1 - 0.2 = 0.8 (80% used).

The tricky part is when you have multiple hard drives - you need to specify which one you're checking with `mountpoint="/"` or you might get confusing mixed results.

### Section 3: Visualization & Dashboards

**Q7: Visualization choices**

I picked different chart types based on what made the most sense for each metric:

- **Stat (for uptime)**: Shows one big number that's easy to read at a glance. Perfect for uptime because you just want to know "how long has this been running?" You don't need to see how it changed over time.

- **Time Series (for memory)**: Shows a line graph over time because memory usage goes up and down constantly. You want to see patterns like "does memory spike every morning?" or "is there a memory leak?"

- **Gauge (for disk usage)**: Looks like a speedometer and shows percentage with colors. Red means "almost full, danger!" This makes it obvious when you're running out of disk space.

The key is matching the chart to how people actually use the information.

**Q8: Threshold strategy**

The 80% disk warning is pretty standard, but different types of servers need different thresholds. Database servers fill up faster and are more critical, so maybe warn at 70%. Web servers might be fine until 85%.

I'd set up multiple levels like a traffic light system:
- Green: Normal (under 70%)
- Yellow: Warning (70-85%) 
- Red: Critical (85%+)

Then hook up alerts to send emails or Slack messages so people know before things break completely.

**Q9: Dashboard variables**

The `$job` variable is like a dropdown menu that lets you switch between different groups of servers. Instead of hardcoding "node-exporter" in every query, you use `$job` and then you can easily switch to monitor different services.

Grafana looks at your metrics and finds all the different job values automatically. If you add a new server with job="web-server", it'll show up in the dropdown.

The tricky part is testing - you need to make sure your dashboard still works when someone picks different values or if a job disappears.

### Section 4: Production & Scalability

**Q10: Resource planning (100 servers)**

When you're monitoring 100 servers instead of just one, everything needs to be bigger. Based on what I learned from the lab, here's what you'd probably need:

- **CPU**: 4-8 cores (Prometheus works hard processing all those metrics)
- **RAM**: 16-32GB (lots of data to keep in memory for fast queries)
- **Storage**: 1-2TB for a month of data (each server sends about 10MB of metrics per day)

The first thing that'll probably break is storage - all those metrics add up fast. Then you'll hit problems with slow queries when people try to look at dashboards. Solutions include using faster storage, splitting the work across multiple Prometheus servers, or using tools like Thanos for long-term storage.

**Q11: High availability**

Right now if our single Prometheus server dies, we lose everything. For production, you'd want backup systems.

The approach is like having multiple backup generators - run several Prometheus instances collecting the same data, use something like Thanos to keep them all in sync, and put multiple Grafana servers behind a load balancer so if one breaks, users automatically get switched to another.

The downside is complexity - more moving parts means more things that can break. But the upside is your monitoring keeps working even when individual pieces fail.

**Q12: Security hardening**

Looking at our lab setup, there are several obvious security holes:

1. **No passwords**: Anyone can access Prometheus and see all your system data
2. **Everything exposed**: All ports are open to the internet 
3. **No encryption**: Data travels in plain text
4. **Default passwords**: Grafana starts with admin/admin
5. **No secrets management**: Passwords are hardcoded in config files

To fix this, add authentication (like Google login), only allow access from your company network, use HTTPS encryption, change all default passwords, and use proper secret management tools.

### Section 5: Troubleshooting & Operations

**Q13: Target DOWN debugging**

When Prometheus shows a target as "DOWN", it means it can't talk to that server. Here's how I'd figure out what's wrong:

1. **Check if containers are running**: `docker compose ps` - maybe something crashed
2. **Look at the config file**: Make sure `prometheus.yml` has the right server addresses
3. **Check the logs**: `docker logs prometheus` often tells you exactly what's wrong
4. **Test the connection manually**: Try `curl http://node-exporter:9100` to see if you can reach it yourself

Most of the time it's something simple like a typo in the config file, a container that didn't start properly, or network issues blocking the connection.

**Q14: Performance optimization**

Some dashboard queries are slow and make Grafana feel sluggish. The worst ones usually involve looking at lots of data over long time periods or doing complex calculations.

To speed things up:
- Use more specific labels in queries (don't grab everything)
- Look at shorter time ranges when possible
- Create "recording rules" that pre-calculate complex queries
- Monitor your monitoring system itself to see what's slow

It's like optimizing a slow website - you need to find the bottlenecks and fix them one by one.

**Q15: Capacity planning**

Over time, Prometheus eats more and more disk space because it keeps storing metrics. The amount depends on how many servers you're monitoring, how often you collect data, and how long you keep it.

You need to decide: Do you need every detail from 6 months ago, or is it okay to keep only daily summaries? Maybe keep detailed data for 30 days, then weekly summaries for a year, then delete everything older.

For really old data, you can archive it to cheaper cloud storage like Amazon S3. It's about balancing cost (storage is expensive) with usefulness (historical data helps spot long-term trends).

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

