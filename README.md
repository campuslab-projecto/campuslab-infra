# campuslab-infra

Repositorio de infraestructura y documentación global del sistema CampusLab.

---

## Descripción

Este repositorio contiene la configuración necesaria para levantar los servicios backend de CampusLab mediante Docker Compose, utilizando imágenes Docker publicadas para cada microservicio.

La infraestructura actual permite ejecutar localmente:

- BFF de CampusLab.
- Microservicio de reservas.
- Microservicio administrador de RabbitMQ.
- RabbitMQ con Management UI.

También contiene la documentación general de arquitectura, puertos, servicios y la matriz de funcionalidades del proyecto.

El objetivo es permitir la ejecución de la solución mediante:

```bash
docker compose up -d
```

y mantener la infraestructura preparada para posterior despliegue en AWS EC2.

---

## Repositorios del proyecto

CampusLab está dividido en repositorios independientes.

| Componente | Repositorio |
|---|---|
| Frontend Angular | `campuslab-frontend` |
| BFF Spring Boot | `campuslab-bff` |
| Microservicio de reservas | `campuslab-ms-bookings` |
| Microservicio de catálogo | `campuslab-ms-catalog` |
| Microservicio de notificaciones | `campuslab-ms-notify` |
| Microservicio de reportería | `campuslab-ms-report` |
| Microservicio de auditoría | `campuslab-ms-audit` |
| Administrador RabbitMQ | `campuslab-ms-rabbit-admin` |
| Infraestructura | `campuslab-infra` |

Organización GitHub:

```text
https://github.com/campuslab-projecto
```

---

## Arquitectura general

Arquitectura principal del sistema:

```text
Angular Frontend
       ↓
Azure AD / MSAL
       ↓
AWS API Gateway
       ↓
campuslab-bff
       ↓
Microservicios CampusLab
       ↓
RabbitMQ / Base de datos
```

Para el flujo de reservas y mensajería:

```text
Frontend Angular
       ↓
BFF
       ↓
campuslab-ms-bookings
       ↓
RabbitMQ
   ↙      ↓       ↘
Email   Prep    Voucher
```

---

## Servicios locales

Actualmente la infraestructura local incluye:

| Servicio | Imagen | Puerto |
|---|---|---:|
| campuslab-bff | `lukmezac/campuslab-bff:latest` | 8080 |
| campuslab-ms-bookings | `lukmezac/campuslab-ms-bookings:latest` | 8081 |
| campuslab-ms-rabbit-admin | `lukmezac/campuslab-ms-rabbit-admin:latest` | 8082 |
| RabbitMQ AMQP | `rabbitmq:3-management` | 5672 |
| RabbitMQ Management | `rabbitmq:3-management` | 15672 |

---

## Docker Compose

Archivo principal:

```text
docker-compose.yml
```

Configuración utilizada:

```yaml
services:

  rabbitmq:
    image: rabbitmq:3-management
    container_name: campuslab-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: campuslab
      RABBITMQ_DEFAULT_PASS: campuslab123
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

  campuslab-ms-bookings:
    image: lukmezac/campuslab-ms-bookings:latest
    container_name: campuslab-ms-bookings
    ports:
      - "8081:8081"
    environment:
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PORT: 5672
      RABBITMQ_USERNAME: campuslab
      RABBITMQ_PASSWORD: campuslab123
    depends_on:
      - rabbitmq

  campuslab-ms-rabbit-admin:
    image: lukmezac/campuslab-ms-rabbit-admin:latest
    container_name: campuslab-ms-rabbit-admin
    ports:
      - "8082:8082"
    environment:
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PORT: 5672
      RABBITMQ_USERNAME: campuslab
      RABBITMQ_PASSWORD: campuslab123
    depends_on:
      - rabbitmq

  campuslab-bff:
    image: lukmezac/campuslab-bff:latest
    container_name: campuslab-bff
    ports:
      - "8080:8080"
    environment:
      BOOKINGS_SERVICE_URL: http://campuslab-ms-bookings:8081
    depends_on:
      - campuslab-ms-bookings

volumes:
  rabbitmq_data:
```

---

## Variables de entorno

### campuslab-bff

| Variable | Descripción |
|---|---|
| `BOOKINGS_SERVICE_URL` | URL interna del microservicio de reservas |

Ejemplo:

```text
http://campuslab-ms-bookings:8081
```

### campuslab-ms-bookings

| Variable | Descripción |
|---|---|
| `RABBITMQ_HOST` | Host de RabbitMQ |
| `RABBITMQ_PORT` | Puerto AMQP |
| `RABBITMQ_USERNAME` | Usuario RabbitMQ |
| `RABBITMQ_PASSWORD` | Contraseña RabbitMQ |

Configuración Docker:

```text
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=campuslab
RABBITMQ_PASSWORD=campuslab123
```

El microservicio utiliza H2 en memoria como configuración local por defecto.

### campuslab-ms-rabbit-admin

| Variable | Descripción |
|---|---|
| `RABBITMQ_HOST` | Host de RabbitMQ |
| `RABBITMQ_PORT` | Puerto AMQP |
| `RABBITMQ_USERNAME` | Usuario RabbitMQ |
| `RABBITMQ_PASSWORD` | Contraseña RabbitMQ |

Configuración Docker:

```text
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=campuslab
RABBITMQ_PASSWORD=campuslab123
```

---

## RabbitMQ

RabbitMQ es utilizado para ejecutar tareas asíncronas relacionadas con las reservas.

### Exchanges

```text
cmd.direct
cmd.topic
cmd.dead.dlx
```

| Exchange | Tipo |
|---|---|
| `cmd.direct` | direct |
| `cmd.topic` | topic |
| `cmd.dead.dlx` | direct |

### Colas principales

```text
q.cmd.email
q.cmd.prep
q.cmd.voucher
```

| Cola | Propósito |
|---|---|
| `q.cmd.email` | Notificaciones al estudiante |
| `q.cmd.prep` | Ticket de preparación |
| `q.cmd.voucher` | Voucher o acta de devolución |

### Dead Letter Queues

```text
q.cmd.email.dlq
q.cmd.prep.dlq
q.cmd.voucher.dlq
```

Las DLQ permiten almacenar mensajes que no pudieron ser procesados correctamente.

Durante las pruebas se verificó un flujo donde un mensaje rechazado mediante `NACK` llegó a:

```text
q.cmd.voucher.dlq
```

con:

```text
Ready = 1
```

### Routing Keys

Routing keys principales:

```text
email.send
prep.ticket
voucher.gen
```

Bindings topic definidos para la arquitectura:

```text
email.*
prep.#
voucher.*
```

---

## Flujo de mensajería

### Reserva aprobada

```text
APROBADA
   ↓
BookingMessageProducer
   ↓
cmd.direct
 ↙         ↘
email     prep
 ↓          ↓
q.cmd.email q.cmd.prep
 ↓          ↓
EmailConsumer
PrepConsumer
```

### Reserva devuelta

```text
DEVUELTA
   ↓
BookingMessageProducer
   ↓
cmd.direct
 ↙           ↘
email       voucher
 ↓            ↓
q.cmd.email q.cmd.voucher
 ↓            ↓
EmailConsumer
VoucherConsumer
```

---

## Confirmación de mensajes

Los consumidores utilizan confirmación manual.

Cuando un mensaje se procesa correctamente:

```java
channel.basicAck(deliveryTag, false);
```

Ante un error:

```java
channel.basicNack(
    deliveryTag,
    false,
    false
);
```

Al utilizar `requeue = false`, RabbitMQ puede derivar el mensaje hacia la Dead Letter Queue correspondiente.

---

## RabbitMQ Management

RabbitMQ Management está disponible localmente en:

```text
http://localhost:15672
```

Credenciales locales:

```text
Usuario: campuslab
Contraseña: campuslab123
```

Desde esta interfaz se pueden revisar:

- Queues
- Exchanges
- Bindings
- DLQ
- Cantidad de mensajes
- Consumers

---

## RabbitMQ Admin API

El microservicio:

```text
campuslab-ms-rabbit-admin
```

se ejecuta en:

```text
http://localhost:8082
```

URL base:

```text
http://localhost:8082/api/rabbit
```

Su objetivo es administrar recursos RabbitMQ mediante una API REST.

### Crear cola

```http
POST http://localhost:8082/api/rabbit/queues
```

Ejemplo:

```json
{
  "name": "q.prueba.campuslab",
  "durable": true
}
```

### Eliminar cola

```http
DELETE http://localhost:8082/api/rabbit/queues/q.prueba.campuslab
```

### Crear exchange

```http
POST http://localhost:8082/api/rabbit/exchanges
```

Ejemplo:

```json
{
  "name": "ex.prueba.campuslab",
  "type": "direct"
}
```

Tipos soportados:

```text
direct
topic
fanout
```

### Eliminar exchange

```http
DELETE http://localhost:8082/api/rabbit/exchanges/ex.prueba.campuslab
```

### Crear binding

```http
POST http://localhost:8082/api/rabbit/bindings
```

Ejemplo:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.prueba.campuslab",
  "routingKey": "prueba.route"
}
```

### Eliminar binding

```http
DELETE http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.prueba.campuslab",
  "routingKey": "prueba.route"
}
```

---

## Validaciones RabbitMQ Admin

Se utilizan DTOs y Jakarta Validation.

Ejemplo:

```java
@NotBlank(message = "El nombre de la cola es obligatorio")
```

Una cola con nombre vacío genera:

```text
400 Bad Request
```

También se valida el tipo de exchange.

Ejemplo inválido:

```json
{
  "name": "ex.invalido",
  "type": "cualquier-cosa"
}
```

Respuesta:

```json
{
  "error": "Bad Request",
  "message": "Tipo de exchange no soportado. Use direct, topic o fanout",
  "status": 400
}
```

---

## Manejo global de errores

El microservicio administrador utiliza:

```text
GlobalExceptionHandler
```

para transformar errores internos en respuestas HTTP claras.

Ejemplo:

```text
IllegalArgumentException
       ↓
GlobalExceptionHandler
       ↓
400 Bad Request
```

---

## Ejecución local

Levantar todos los servicios:

```bash
docker compose up -d
```

Comprobar contenedores:

```bash
docker ps
```

Servicios esperados:

```text
campuslab-rabbitmq
campuslab-ms-bookings
campuslab-ms-rabbit-admin
campuslab-bff
```

---

## Verificación de servicios

### Reservas

```bash
curl http://localhost:8081/api/bookings
```

### RabbitMQ Admin

```text
http://localhost:8082
```

### RabbitMQ Management

```text
http://localhost:15672
```

### BFF

```bash
curl http://localhost:8080/api/bookings
```

Sin JWT válido, el BFF debe proteger los endpoints configurados mediante Spring Security.

---

## Colección Postman

Se dispone de una colección Postman organizada de la siguiente manera:

```text
CampusLab - Cloud Native
│
├── Reservas
│   ├── Crear reserva
│   ├── Aprobar reserva
│   └── Marcar reserva como devuelta
│
└── RabbitMQ Admin
    │
    ├── Colas
    │   ├── Crear cola
    │   └── Eliminar cola
    │
    ├── Exchanges
    │   ├── Crear exchange
    │   ├── Eliminar exchange
    │   └── Validar exchange inválido
    │
    ├── Bindings
    │   ├── Crear binding
    │   └── Eliminar binding
    │
    └── Validaciones
        └── Crear cola con nombre vacío
```

---

## Secuencia recomendada de demostración

### Reservas

```text
Crear reserva
      ↓
APROBADA
      ↓
DEVUELTA
```

### Administración RabbitMQ

```text
Crear cola
      ↓
Crear exchange
      ↓
Crear binding
      ↓
Verificar en RabbitMQ Management
      ↓
Eliminar binding
      ↓
Eliminar cola
      ↓
Eliminar exchange
```

---

## Despliegue previo en AWS

En la evaluación anterior se implementaron:

- Frontend Angular desplegado mediante AWS Amplify.
- BFF Spring Boot.
- API Gateway.
- Microservicio de reservas.
- Autenticación mediante Azure AD / MSAL.
- Validación JWT en backend.
- Infraestructura Docker / EC2.

Flujo utilizado:

```text
Angular Frontend
       ↓
AWS API Gateway
       ↓
campuslab-bff
       ↓
campuslab-ms-bookings
```

La incorporación de RabbitMQ extiende esta arquitectura sin reemplazar la lógica existente.

---

## Despliegue en EC2

En una instancia EC2 con Docker y Git instalados:

```bash
git clone https://github.com/campuslab-projecto/campuslab-infra.git
cd campuslab-infra
docker compose up -d
```

Para un despliegue productivo se deben configurar los Security Groups correspondientes y evitar exponer directamente servicios internos que no requieran acceso público.

---

## Matriz de funcionalidades

La siguiente matriz sistematiza el estado actual del proyecto CampusLab.

| Funcionalidad | Estado | Componente responsable | Evidencia |
|---|---|---|---|
| Login con Azure AD / MSAL | Implementada | `campuslab-frontend` | Frontend desplegado en AWS Amplify |
| Validación JWT | Implementada | `campuslab-bff` | Spring Security / Resource Server |
| Crear reserva | Implementada | `campuslab-ms-bookings` | `POST /api/bookings` |
| Consultar reservas | Implementada | `campuslab-ms-bookings` | `GET /api/bookings` |
| Consultar reserva por ID | Implementada | `campuslab-ms-bookings` | `GET /api/bookings/{id}` |
| Cambiar estado de reserva | Implementada | `campuslab-ms-bookings` | `PUT /api/bookings/{id}/status` |
| Evento BOOKING_CREATED | Implementada | `campuslab-ms-bookings` | Producer RabbitMQ |
| Evento BOOKING_STATUS_CHANGED | Implementada | `campuslab-ms-bookings` | Producer RabbitMQ |
| Notificación email asíncrona | Implementada a nivel de mensajería | `campuslab-ms-bookings` | `q.cmd.email` + `EmailConsumer` |
| Ticket de preparación | Implementada a nivel de mensajería | `campuslab-ms-bookings` | `q.cmd.prep` + `PrepConsumer` |
| Voucher / acta de devolución | Parcial | `campuslab-ms-bookings` | `q.cmd.voucher` + `VoucherConsumer`; mensajería operativa |
| Exchange direct | Implementada | RabbitMQ | `cmd.direct` |
| Exchange topic | Implementada | RabbitMQ | `cmd.topic` |
| Dead Letter Exchange | Implementada | RabbitMQ | `cmd.dead.dlx` |
| DLQ email | Implementada | RabbitMQ | `q.cmd.email.dlq` |
| DLQ preparación | Implementada | RabbitMQ | `q.cmd.prep.dlq` |
| DLQ voucher | Implementada | RabbitMQ | `q.cmd.voucher.dlq` |
| ACK manual | Implementada | Consumers RabbitMQ | `basicAck()` |
| NACK sin requeue | Implementada | Consumers RabbitMQ | `basicNack(..., false, false)` |
| Envío a DLQ | Implementada y probada | RabbitMQ | `q.cmd.voucher.dlq Ready = 1` |
| Crear cola vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/queues` |
| Eliminar cola vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/queues/{name}` |
| Crear exchange vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/exchanges` |
| Eliminar exchange vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/exchanges/{name}` |
| Crear binding vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/bindings` |
| Eliminar binding vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/bindings` |
| Validación de nombre vacío | Implementada | `campuslab-ms-rabbit-admin` | `400 Bad Request` |
| Validación de tipo de exchange | Implementada | `campuslab-ms-rabbit-admin` | `400 Bad Request` |
| Manejo global de errores | Implementada | `campuslab-ms-rabbit-admin` | `GlobalExceptionHandler` |
| Docker Compose local | Implementada | `campuslab-infra` | RabbitMQ, bookings, rabbit-admin y BFF |
| Frontend AWS Amplify | Implementada previamente | `campuslab-frontend` | Deploy evaluación anterior |
| API Gateway | Implementada previamente | AWS / BFF | Endpoint evaluación anterior |
| Despliegue de RabbitMQ en AWS | Pendiente de validación | `campuslab-infra` | Pendiente |
| Catálogo | Por verificar | `campuslab-ms-catalog` | Revisión pendiente |
| Servicio notify independiente | Por verificar | `campuslab-ms-notify` | Revisión pendiente |
| Reportería | Por verificar | `campuslab-ms-report` | Revisión pendiente |
| Auditoría | Por verificar | `campuslab-ms-audit` | Revisión pendiente |

---

## Evidencias de EP3

Durante el desarrollo se verificó:

- Creación de las seis colas RabbitMQ.
- Creación de exchanges `direct`, `topic` y `dead`.
- Envío de eventos desde `campuslab-ms-bookings`.
- Consumo de mensajes por email, preparación y voucher.
- ACK manual.
- NACK sin requeue.
- Mensaje enviado correctamente a DLQ.
- Creación y eliminación de colas vía REST.
- Creación y eliminación de exchanges vía REST.
- Creación y eliminación de bindings vía REST.
- Validaciones `400 Bad Request`.
- Ejecución mediante Docker Compose.
- Pruebas mediante Postman.
- Verificación visual mediante RabbitMQ Management.

---

## Gestión del proyecto

El proyecto se gestiona mediante GitHub Projects y metodología Kanban.

Flujo utilizado:

```text
Issue
  ↓
Rama feature
  ↓
Commit
  ↓
Pull Request
  ↓
Revisión
  ↓
Merge a main
```

La rama `main` se mantiene protegida y los cambios se integran mediante Pull Request.

La guía del proyecto indica trabajar con una rama por tarea y evitar modificaciones directas sobre `main`. :chatgpt-content-reference{index="0"}

---

## Proyecto

Asignatura:

```text
DSY1107 - Desarrollo Cloud Native I
```

Proyecto:

```text
CampusLab
```