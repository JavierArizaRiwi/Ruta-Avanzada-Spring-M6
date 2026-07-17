# Infraestructura local gratuita para la ruta

## Propósito

Esta guía reúne servicios que se levantan por etapas. No es necesario ejecutarlos todos en las primeras semanas.

| Etapa | Servicios |
| --- | --- |
| Persistencia | PostgreSQL |
| Caché | PostgreSQL + Redis/Valkey |
| Eventos | anteriores + Kafka |
| Observabilidad | anteriores + Prometheus + Grafana |

## Compose integrado

Combina los servicios descritos en las guías de PostgreSQL, Redis y Kafka. Añade:

```yaml
  prometheus:
    image: prom/prometheus:v3.5.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana-oss:12.1.0
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
    ports: ["3000:3000"]
```

## Comandos

```bash
docker compose config
docker compose up -d postgres
docker compose up -d postgres redis kafka
docker compose ps
docker compose logs -f kafka
docker compose stop
```

`docker compose down -v` elimina los datos del laboratorio. Usarlo solo cuando se quiera reiniciar todo.

## Límites

- Sin servicios pagos ni cuenta cloud.
- LocalStack no forma parte de la infraestructura principal.
- No usar etiquetas `latest`.
- Los valores de ejemplo nunca se reutilizan en producción.
- Iniciar únicamente lo necesario para no exigir equipos de alto rendimiento.

