# Redis y Valkey: caché local y estructuras en memoria

## Qué problema resuelven

Redis y Valkey almacenan datos en memoria con estructuras y expiración. En esta ruta se usan como caché; PostgreSQL continúa siendo fuente de verdad.

## Conceptos

- key/value, TTL y expiración;
- cache hit/miss;
- cache-aside;
- serialización;
- invalidación y datos obsoletos;
- eviction policy;
- stampede y hot keys;
- Pub/Sub no durable frente a streams/logs durables.

## Ejecución local

```yaml
services:
  redis:
    image: redis:8.8.0-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports: ["6379:6379"]
    volumes: ["redis_data:/data"]
volumes:
  redis_data:
```

```bash
docker compose up -d redis
docker compose exec redis redis-cli PING
```

## Diseño de claves

```text
learning-route:v1:{routeId}
progress-summary:v1:{enrollmentId}
```

Versionar prefijos ayuda a cambiar serialización. Evitar datos sensibles, TTL infinito y `KEYS *` en conjuntos grandes.

## Redis frente a Valkey

Valkey es una alternativa compatible para muchos casos y clientes. El ejercicio opcional consiste en cambiar la imagen, ejecutar las mismas pruebas y documentar comandos o módulos que no sean portables.

## Cuándo no usarlo

- lectura barata y poco frecuente;
- datos que no toleran obsolescencia;
- sistema pequeño sin evidencia de cuello de botella;
- intento de esconder consultas JPA deficientes.

## Recursos oficiales

- [Redis](https://redis.io/docs/latest/)
- [Valkey](https://valkey.io/topics/)

