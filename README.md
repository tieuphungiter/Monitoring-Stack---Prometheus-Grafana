# Monitoring Stack - Prometheus + Grafana

Deploy server: 10.0.X.A  
Monitored servers: 10.0.X.A, 10.0.X.B, 10.0.X.C, 10.0.X.D

## Architecture

```text
+-------------------------------------------------------------+
|  10.0.X.A (Deploy server)                                   |
|  +------------+   +---------+   +--------------+            |
|  | Prometheus |   | Grafana |   | node-exporter|            |
|  | :9090      |   | :3000   |   | :9100        |            |
|  +-----+------+   +----+----+   +--------------+            |
|        |               |                                    |
|        | scrapes       | visualizes / alerts                |
|  +-----v--------------------------------------+             |
|  | cAdvisor :8090                            |             |
|  +-------------------------------------------+             |
+-------------------------------------------------------------+
         |
         +-- 10.0.X.B:9100 (node-exporter)
         |   10.0.X.B:8090 (cAdvisor)
         |   10.0.X.B:8081 (Java service)
         |
         +-- 10.0.X.C:9100 (node-exporter)
         |   10.0.X.C:8090 (cAdvisor)
         |   10.0.X.C:8081 (Java service)
         |
         +-- 10.0.X.D:9100 (node-exporter)
           10.0.X.D:8090 (cAdvisor)
           10.0.X.D:8081 (Java service)
```

## 1. Deploy The Monitoring Stack

```bash
# Copy project to the deploy server
scp -r monitoring/ user@10.0.X.A:~/

# Connect to the deploy server
ssh user@10.0.X.A
cd ~/monitoring

# Create environment file
cp .env.example .env
nano .env

# Start all services
docker-compose up -d
docker-compose ps
```

## 2. Deploy node-exporter + cAdvisor On Other Servers

The exporter compose file is available at exporters/docker-compose.yml.

Run on each target host:

```bash
# From deploy server, copy exporter compose file to a target server
scp exporters/docker-compose.yml user@10.0.X.B:~/monitoring-exporters/docker-compose.yml

# On the target server
ssh user@10.0.X.B
cd ~/monitoring-exporters

# Start services
docker compose up -d
docker compose ps

# Verify metrics endpoints
curl http://localhost:9100/metrics | head -5
curl http://localhost:8090/metrics | head -5
```

## 3. Add A New Server (No Prometheus Restart Required)

Step 1: Deploy exporters on the new server (see section 2).

Step 2: Add the new target in Prometheus file-based discovery files.

```bash
ssh user@10.0.X.A
cd ~/monitoring
```

Update prometheus/targets/node_exporter.yml:

```yaml
- targets: ["10.0.X.N:9100"]
  labels:
    hostname: "10.0.X.N"
    server_role: "app"
    env: "production"
```

Update prometheus/targets/cadvisor.yml with port 8090 in the same pattern.

Step 3: Prometheus auto-reloads file_sd configs in ~30 seconds, or reload immediately:

```bash
curl -X POST http://10.0.X.A:9090/-/reload
```

Check Prometheus targets page:

- http://10.0.X.A:9090/targets

## 4. Spring Boot (Java Microservice)

pom.xml dependencies:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

application.yml:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus
  endpoint:
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
```

## 5. Firewall / Security Group

Allow these ports only inside your VPC or private network:

| Port | Service       | Allowed Source                  |
|------|---------------|----------------------------------|
| 9100 | node-exporter | Prometheus server (10.0.X.A)    |
| 8090 | cAdvisor      | Prometheus server (10.0.X.A)    |
| 8081 | Java service  | Prometheus server (10.0.X.A)    |
| 9090 | Prometheus    | Internal admins / VPN           |
| 3000 | Grafana       | Internal users / VPN            |

## 6. SMTP Alert Configuration (Grafana + AWS SES)

SMTP values are loaded from .env and mapped in docker-compose.yml.
Do not hardcode secrets in docker-compose.yml.

Required .env keys:

```env
SES_SMTP_ENABLED=true
SES_SMTP_HOST=email-smtp.ap-southeast-2.amazonaws.com:258
SES_SMTP_USER=<your_ses_smtp_username>
SES_SMTP_PASSWORD=<your_ses_smtp_password>
SES_SMTP_FROM_ADDRESS=alerts@domain_smpt.com
SES_SMTP_FROM_NAME=Grafana Alerts
SES_SMTP_STARTTLS_POLICY=MandatoryStartTLS
SES_SMTP_SKIP_VERIFY=false
```

Apply changes:

```bash
docker-compose up -d grafana
```

In Grafana:

1. Go to Alerting > Contact points.
2. Create or edit an Email contact point.
3. Click Test to send a test notification.

If delivery fails, validate SMTP host/port, credentials, SES sandbox restrictions, and outbound connectivity from the Grafana host.

## 7. Grafana

- URL: http://10.0.X.A:3000
- Login: admin / value configured in .env
- Included dashboards: EC2 Host, Docker Containers, Java Microservice
- Optional imports from grafana.com: 1860 (Node Exporter Full), 14282 (cAdvisor)

## 8. Useful Commands

```bash
# View logs
docker-compose logs -f prometheus
docker-compose logs -f grafana

# Reload Prometheus config without restart
curl -X POST http://10.0.X.A:9090/-/reload

# Check metrics endpoints manually
curl http://10.0.X.B:9100/metrics
curl http://10.0.X.B:8090/metrics
curl http://10.0.X.B:8081/actuator/prometheus

# Restart one service
docker-compose restart prometheus
```
