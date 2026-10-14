# campuslab-infra

Repositorio de infraestructura del sistema CampusLab.

## Descripción

Este repositorio contiene la configuración necesaria para levantar los servicios backend de CampusLab mediante Docker Compose, utilizando imágenes publicadas en Docker Hub.

El objetivo es permitir la ejecución de los servicios mediante:

```bash
docker compose pull
docker compose up -d
```

Esto deja la solución preparada para despliegue local y posterior despliegue en AWS EC2.

## Servicios incluidos

| Servicio | Imagen Docker Hub | Puerto |
|---|---|---|
| campuslab-bff | `lukmezac/campuslab-bff:latest` | 8080 |
| campuslab-ms-bookings | `lukmezac/campuslab-ms-bookings:latest` | 8081 |

## Arquitectura

```text
Angular Frontend → BFF → ms-campuslab-bookings → Base de datos
```

El BFF funciona como punto de entrada protegido.  
Valida el JWT emitido por Azure AD y luego redirige las solicitudes hacia el microservicio de reservas.

## Docker Compose

Archivo principal:

```text
docker-compose.yml
```

Contenido general:

```yaml
services:
  campuslab-ms-bookings:
    image: lukmezac/campuslab-ms-bookings:latest
    container_name: campuslab-ms-bookings
    ports:
      - "8081:8081"
    environment:
      DB_URL: jdbc:h2:mem:testdb
      DB_DRIVER: org.h2.Driver
      DB_USERNAME: sa
      DB_PASSWORD: password
      DB_DIALECT: org.hibernate.dialect.H2Dialect

  campuslab-bff:
    image: lukmezac/campuslab-bff:latest
    container_name: campuslab-bff
    ports:
      - "8080:8080"
    environment:
      BOOKINGS_SERVICE_URL: http://campuslab-ms-bookings:8081
    depends_on:
      - campuslab-ms-bookings
```

## Variables de entorno

### campuslab-bff

| Variable | Descripción |
|---|---|
| `BOOKINGS_SERVICE_URL` | URL interna del microservicio de reservas |

### campuslab-ms-bookings

| Variable | Descripción |
|---|---|
| `DB_URL` | URL de conexión a base de datos |
| `DB_DRIVER` | Driver JDBC |
| `DB_USERNAME` | Usuario de base de datos |
| `DB_PASSWORD` | Contraseña de base de datos |
| `DB_DIALECT` | Dialecto Hibernate |

Por defecto, el microservicio usa H2 en memoria para ejecución local.  
La configuración queda preparada para conexión a una base de datos cloud mediante variables de entorno.

## Ejecución

Descargar imágenes desde Docker Hub:

```bash
docker compose pull
```

Levantar servicios:

```bash
docker compose up -d
```

Verificar contenedores activos:

```bash
docker ps
```

## Verificación de servicios

Microservicio de reservas:

```bash
curl http://localhost:8081/api/bookings
```

Respuesta esperada:

```json
[]
```

BFF protegido:

```bash
curl http://localhost:8080/api/bookings
```

Respuesta esperada sin token:

```text
401 Unauthorized
```

Esto demuestra que el BFF no permite consumir el endpoint si no existe un JWT válido.

## Despliegue en EC2

En una instancia EC2 con Docker y Git instalados:

```bash
git clone https://github.com/campuslab-projecto/campuslab-infra.git
cd campuslab-infra
docker compose pull
docker compose up -d
```

Luego se debe configurar el Security Group de EC2 para permitir acceso al puerto necesario, idealmente exponiendo el BFF como punto de entrada.

## Flujo esperado en nube

```text
Angular Frontend → AWS API Gateway → campuslab-bff en EC2 → campuslab-ms-bookings → Base de datos cloud
```

## Evidencia esperada

- `docker compose pull` descarga correctamente las imágenes desde Docker Hub.
- `docker compose up -d` levanta ambos contenedores.
- `docker ps` muestra:
  - `campuslab-bff` en puerto 8080.
  - `campuslab-ms-bookings` en puerto 8081.
- `http://localhost:8081/api/bookings` responde correctamente.
- `http://localhost:8080/api/bookings` responde `401 Unauthorized` sin token.

## Gestión del proyecto

Este repositorio se gestiona mediante GitHub Projects y metodología Kanban.

Flujo utilizado:

```text
Issue → Rama feature → Commit → Pull Request → Revisión → Merge a main
```

La rama `main` se mantiene protegida y los cambios se integran mediante Pull Request.
