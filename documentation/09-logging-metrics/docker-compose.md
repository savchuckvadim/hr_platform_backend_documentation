# Docker Compose - Конфигурация Docker

## Обзор

Конфигурация Docker Compose для запуска Prometheus, Loki, Promtail и Grafana.

## Docker Compose

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: ${PROM_CONTAINER_NAME:-hr-platform-prometheus}
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data/prometheus:/prometheus
    ports:
      - "${PROM_PORT:-9090}:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - app-network
    restart: unless-stopped

  # Grafana Loki
  loki:
    image: grafana/loki:2.9.3
    container_name: ${LOKI_CONTAINER_NAME:-hr-platform-loki}
    ports:
      - "${LOKI_PORT:-3100}:3100"
    volumes:
      - ./config/loki/config.yaml:/etc/loki/config.yaml
      - ./data/loki/chunks:/loki/chunks
      - ./data/loki/index:/loki/index
      - ./data/loki/cache:/loki/boltdb-cache
      - ./data/loki/compactor:/loki/compactor
      - ./data/loki/wal:/loki/wal
    command: -config.file=/etc/loki/config.yaml
    networks:
      - app-network
    restart: unless-stopped

  # Promtail
  promtail:
    image: grafana/promtail:2.9.3
    container_name: ${PROMTAIL_CONTAINER_NAME:-hr-platform-promtail}
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/machine-id:/etc/machine-id:ro
      - ./config/promtail/config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml
    networks:
      - app-network
    restart: unless-stopped
    depends_on:
      - loki

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: ${GRAFANA_CONTAINER_NAME:-hr-platform-grafana}
    ports:
      - "${GRAFANA_PORT:-3000}:3000"
    volumes:
      - ./data/grafana:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=${GF_SECURITY_ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD:-admin}
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    networks:
      - app-network
    restart: unless-stopped
    depends_on:
      - prometheus
      - loki

networks:
  app-network:
    external: true
```

## Environment Variables

```env
# Prometheus
PROM_CONTAINER_NAME=hr-platform-prometheus
PROM_PORT=9090

# Loki
LOKI_CONTAINER_NAME=hr-platform-loki
LOKI_PORT=3100

# Promtail
PROMTAIL_CONTAINER_NAME=hr-platform-promtail

# Grafana
GRAFANA_CONTAINER_NAME=hr-platform-grafana
GRAFANA_PORT=3000
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=your-secure-password
```

## Запуск

```bash
# Создание сети (если еще не создана)
docker network create app-network

# Запуск мониторинга
docker-compose -f docker-compose.monitoring.yml up -d

# Проверка статуса
docker-compose -f docker-compose.monitoring.yml ps

# Просмотр логов
docker-compose -f docker-compose.monitoring.yml logs -f
```

## Доступ

- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000
- **Loki**: http://localhost:3100

## Best Practices

1. **Используйте volumes** - для персистентности данных
2. **Настройте retention** - для управления размером
3. **Используйте external network** - для связи с основным приложением
4. **Настройте restart policy** - для автоматического перезапуска
