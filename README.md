# digitalfix-infra

Repositorio de infraestructura y orquestación para desplegar la plataforma **DigitalFix** mediante **Docker Compose** (local y AWS EC2 + Amazon RDS PostgreSQL).

## Contenido

| Archivo | Propósito |
| :--- | :--- |
| `docker-compose.messaging.yml` | Capa de mensajería: Kafka (3 Zookeeper + 3 brokers + Kafka UI) y clúster RabbitMQ (2 nodos). |
| `docker-compose.apps.yml` | Los 8 microservicios Spring Boot conectados a RDS, Kafka y RabbitMQ con límites de memoria. |
| `rabbitmq.conf` | Configuración de auto-clustering clásico y límite de memoria para los 2 nodos de RabbitMQ. |
| `init-db.sql` | Script SQL para crear las 4 bases de datos independientes en Amazon RDS PostgreSQL. |
| `.env.example` | Plantilla de variables de entorno requeridas (RDS, RabbitMQ, Microsoft Entra ID y CORS). |

## Estructura esperada de carpetas

Los repositorios deben clonarse al mismo nivel dentro de una carpeta común (por ejemplo `~/DigitalFix`):

```text
DigitalFix/
├── digitalfix-infra/
├── ms-digitalfix-bff/
├── ms-digitalfix-workorders/
├── ms-digitalfix-catalog/
├── ms-digitalfix-audit/
├── ms-digitalfix-report/
├── ms-digitalfix-notify/
├── ms-digitalfix-rabbit-admin/
└── ms-digitalfix-kafka-admin/
```

## Despliegue paso a paso

### 1. Inicializar bases de datos en Amazon RDS PostgreSQL
Ejecutar una sola vez contra la instancia RDS:
```bash
psql -h <RDS_HOST> -U postgres -f init-db.sql
```

### 2. Configurar variables de entorno
```bash
cp .env.example .env
# Editar .env con credenciales reales de RDS, RabbitMQ, Entra ID y URL de CloudFront
```

### 3. Levantar la plataforma
```bash
# Levantar mensajería y los 8 microservicios
docker compose -f docker-compose.messaging.yml -f docker-compose.apps.yml up -d --build

# Ver estado de contenedores
docker compose -f docker-compose.messaging.yml -f docker-compose.apps.yml ps
```

> **Nota:** tras el primer arranque exitoso, cambiar `CATALOG_SQL_INIT_MODE=never` en `.env` para no re-ejecutar `data.sql` en reinicios posteriores.

### 4. Detener la plataforma
```bash
docker compose -f docker-compose.messaging.yml -f docker-compose.apps.yml down
```

## Puertos y exposición

| Servicio | Puerto | Alcance |
| :--- | :---: | :--- |
| `ms-bff` | `8080` | Expuesto hacia AWS API Gateway |
| `kafka-ui` | `8090` | Consola Kafka (restringir por Security Group / túnel SSH) |
| `rabbitmq-1` (UI / AMQP) | `15672` / `5672` | Management UI del clúster RabbitMQ (2 nodos) |
| Microservicios internos (`8081`–`8088`) | `127.0.0.1:*` | Solo red Docker interna / SSH local (no expuestos a Internet) |