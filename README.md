# campuslab-infra

Repositorio de infraestructura y documentación global del sistema CampusLab.

---

## Descripción

Este repositorio contiene la configuración necesaria para levantar los servicios backend de CampusLab mediante Docker Compose, utilizando imágenes Docker publicadas para cada microservicio.

La infraestructura actual permite ejecutar localmente:

- BFF de CampusLab.
- Microservicio de reservas.
- Microservicio de notificaciones.
- Microservicio administrador de RabbitMQ.
- RabbitMQ con Management UI.

También contiene la documentación general de arquitectura, puertos, servicios, integración con AWS RDS y la matriz de funcionalidades del proyecto.

El objetivo es permitir la ejecución de la solución mediante:

```bash
docker compose up -d
```

y mantener la infraestructura preparada para despliegue en AWS.

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
campuslab-ms-bookings
       ├──────────────→ AWS RDS PostgreSQL
       │
       ↓
    RabbitMQ
       ↓
campuslab-ms-notify
```

Para el flujo de reservas y mensajería asíncrona:

```text
Frontend Angular
       ↓
BFF
       ↓
campuslab-ms-bookings
       │
       ├── Persistencia → AWS RDS PostgreSQL
       │
       ↓
     RabbitMQ
       ↓
campuslab-ms-notify
       ↓
Procesamiento de:
- Email
- Ticket de preparación
- Voucher
```

`campuslab-ms-bookings` administra las reservas, persiste la información y actúa como productor de eventos RabbitMQ.

`campuslab-ms-notify` concentra los consumidores responsables del procesamiento asíncrono de email, preparación y voucher.

`campuslab-ms-rabbit-admin` permite administrar colas, exchanges y bindings mediante una API REST.

Para desarrollo local, `campuslab-ms-bookings` mantiene H2 como configuración por defecto. Para la integración cloud se verificó conexión y persistencia con PostgreSQL en AWS RDS.

---

## Servicios locales

Actualmente la infraestructura local incluye:

| Servicio | Imagen | Puerto |
|---|---|---:|
| campuslab-bff | `lukmezac/campuslab-bff:latest` | 8080 |
| campuslab-ms-bookings | `lukmezac/campuslab-ms-bookings:latest` | 8081 |
| campuslab-ms-rabbit-admin | `lukmezac/campuslab-ms-rabbit-admin:latest` | 8082 |
| campuslab-ms-notify | `lukmezac/campuslab-ms-notify:latest` | 8083 |
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

  campuslab-ms-notify:
    image: lukmezac/campuslab-ms-notify:latest
    container_name: campuslab-ms-notify
    ports:
      - "8083:8083"
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

El Docker Compose local mantiene H2 como almacenamiento por defecto para `campuslab-ms-bookings`.

La conexión con AWS RDS se habilita mediante variables de entorno sin almacenar credenciales en el repositorio.

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

Variables RabbitMQ:

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

Variables de base de datos:

| Variable | Descripción |
|---|---|
| `DB_URL` | URL JDBC de la base de datos |
| `DB_USERNAME` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |
| `DB_DRIVER` | Driver JDBC |
| `DB_DIALECT` | Dialecto Hibernate |

El microservicio utiliza H2 en memoria como configuración local por defecto.

Ejemplo local:

```text
DB_URL=jdbc:h2:mem:testdb
DB_DRIVER=org.h2.Driver
DB_USERNAME=sa
DB_PASSWORD=password
DB_DIALECT=org.hibernate.dialect.H2Dialect
```

### Base de datos cloud - AWS RDS

El microservicio `campuslab-ms-bookings` fue integrado con una base de datos PostgreSQL administrada mediante AWS RDS.

Configuración utilizada:

```text
Motor: PostgreSQL
Base de datos: campuslab
Puerto: 5432
Servicio: AWS RDS
```

La conexión se configura mediante las mismas variables de entorno:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
DB_DRIVER
DB_DIALECT
```

Configuración PostgreSQL:

```text
DB_DRIVER=org.postgresql.Driver
DB_DIALECT=org.hibernate.dialect.PostgreSQLDialect
```

Formato de conexión:

```text
jdbc:postgresql://<RDS_ENDPOINT>:5432/campuslab
```

El usuario y la contraseña no se almacenan en GitHub.

Las credenciales son proporcionadas mediante variables de entorno al momento de ejecutar la aplicación.

Se verificó:

- Resolución DNS del endpoint AWS RDS.
- Conectividad TCP al puerto `5432`.
- Conexión Spring Boot mediante PostgreSQL JDBC.
- Inicialización de HikariCP.
- Inicialización de JPA e Hibernate.
- Creación de la tabla `bookings` en PostgreSQL.
- Creación de una reserva mediante la API.
- Consulta posterior de la reserva.
- Persistencia después de detener y reiniciar completamente `campuslab-ms-bookings`.

Esto demuestra que la información permanece almacenada en AWS RDS y no depende de H2 en memoria.

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

### campuslab-ms-notify

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

`campuslab-ms-bookings` publica eventos y `campuslab-ms-notify` procesa los mensajes de forma desacoplada.

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
| `q.cmd.email` | Notificaciones relacionadas con reservas |
| `q.cmd.prep` | Ticket de preparación |
| `q.cmd.voucher` | Voucher o acta de devolución |

### Dead Letter Queues

```text
q.cmd.email.dlq
q.cmd.prep.dlq
q.cmd.voucher.dlq
```

Las DLQ permiten almacenar mensajes que no pudieron ser procesados correctamente.

Durante las pruebas se verificó un flujo controlado donde un mensaje rechazado mediante `NACK` llegó a:

```text
q.cmd.voucher.dlq
```

con:

```text
Ready = 1
```

Posteriormente se eliminó del código final el fallo intencional utilizado para realizar esta prueba.

### Routing Keys

Routing keys principales:

```text
email.send
prep.ticket
voucher.gen
```

Bindings `topic` definidos para la arquitectura:

```text
email.*
prep.#
voucher.*
```

---

## Flujo de mensajería

### Creación de reserva

```text
SOLICITADA
     ↓
BookingMessageProducer
     ↓
cmd.direct
     ↓
email.send
     ↓
q.cmd.email
     ↓
campuslab-ms-notify
     ↓
EmailConsumer
```

### Reserva aprobada

```text
APROBADA
    ↓
BookingMessageProducer
    ↓
cmd.direct
   ↙        ↘
email.send  prep.ticket
   ↓           ↓
q.cmd.email  q.cmd.prep
   ↓           ↓
campuslab-ms-notify
   ↓           ↓
EmailConsumer PrepConsumer
```

### Reserva devuelta

```text
DEVUELTA
    ↓
BookingMessageProducer
    ↓
cmd.direct
   ↙             ↘
email.send      voucher.gen
   ↓               ↓
q.cmd.email     q.cmd.voucher
   ↓               ↓
campuslab-ms-notify
   ↓               ↓
EmailConsumer   VoucherConsumer
```

Los consumidores se encuentran en el microservicio:

```text
campuslab-ms-notify
```

De esta forma, `campuslab-ms-bookings` mantiene la responsabilidad de gestionar las reservas y producir eventos, mientras que `campuslab-ms-notify` procesa las tareas asíncronas.

---

## Confirmación de mensajes

Los consumidores de `campuslab-ms-notify` utilizan confirmación manual.

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

Al utilizar `requeue = false`, el mensaje no vuelve indefinidamente a la cola principal y RabbitMQ puede derivarlo hacia la Dead Letter Queue correspondiente.

Flujo de error:

```text
Consumer
   ↓
Error de procesamiento
   ↓
basicNack(..., false, false)
   ↓
cmd.dead.dlx
   ↓
DLQ correspondiente
```

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

- Queues.
- Exchanges.
- Bindings.
- Dead Letter Queues.
- Cantidad de mensajes.
- Consumers.

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

La lógica de administración se encuentra encapsulada en `RabbitAdminService`, mientras que el controller expone operaciones de alto nivel.

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

Ejemplo direct:

```json
{
  "name": "ex.direct.campuslab",
  "type": "direct"
}
```

Ejemplo topic:

```json
{
  "name": "ex.topic.campuslab",
  "type": "topic"
}
```

Ejemplo fanout:

```json
{
  "name": "ex.fanout.campuslab",
  "type": "fanout"
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
DELETE http://localhost:8082/api/rabbit/exchanges/ex.direct.campuslab
```

### Crear binding direct

```http
POST http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.direct.campuslab",
  "exchangeType": "direct",
  "routingKey": "prueba.route"
}
```

### Crear binding topic

```http
POST http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.topic.campuslab",
  "exchangeType": "topic",
  "routingKey": "prueba.*"
}
```

### Crear binding fanout

```http
POST http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.fanout.campuslab",
  "exchangeType": "fanout"
}
```

Para `fanout` no es necesario informar `routingKey`.

### Eliminar binding direct

```http
DELETE http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.direct.campuslab",
  "exchangeType": "direct",
  "routingKey": "prueba.route"
}
```

### Eliminar binding topic

```http
DELETE http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.topic.campuslab",
  "exchangeType": "topic",
  "routingKey": "prueba.*"
}
```

### Eliminar binding fanout

```http
DELETE http://localhost:8082/api/rabbit/bindings
```

Body:

```json
{
  "queueName": "q.prueba.campuslab",
  "exchangeName": "ex.fanout.campuslab",
  "exchangeType": "fanout"
}
```

---

## Validaciones RabbitMQ Admin

Se utilizan DTOs y Jakarta Validation para validar los datos recibidos por la API.

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

Respuesta esperada:

```json
{
  "error": "Bad Request",
  "message": "Tipo de exchange no soportado. Use direct, topic o fanout",
  "status": 400
}
```

Para la creación y eliminación de bindings se valida:

- `queueName` obligatorio.
- `exchangeName` obligatorio.
- `exchangeType` obligatorio.
- `routingKey` obligatorio para exchanges `direct`.
- `routingKey` obligatorio para exchanges `topic`.
- `routingKey` no requerido para exchanges `fanout`.

---

## Manejo global de errores

El microservicio administrador utiliza:

```text
GlobalExceptionHandler
```

para transformar errores internos en respuestas HTTP claras.

Flujo:

```text
IllegalArgumentException
       ↓
GlobalExceptionHandler
       ↓
400 Bad Request
```

También se controlan errores de validación producidos por Jakarta Validation.

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
campuslab-ms-notify
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

### Notify

El procesamiento de `campuslab-ms-notify` puede verificarse mediante los logs del contenedor:

```bash
docker logs campuslab-ms-notify
```

En un procesamiento correcto se pueden observar mensajes como:

```text
EMAIL recibido
Email procesado correctamente. ACK enviado.
```

Los logs de `campuslab-ms-bookings` no deben mostrar los consumidores, ya que este microservicio queda como productor de eventos.

### AWS RDS

La conexión con PostgreSQL en AWS RDS puede verificarse en los logs de `campuslab-ms-bookings`.

Ejemplo:

```text
HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection
```

También se verificó:

```text
Started BookingsApplication
```

con la aplicación utilizando PostgreSQL como datasource.

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
    │   ├── Crear exchange direct
    │   ├── Crear exchange topic
    │   ├── Crear exchange fanout
    │   ├── Eliminar exchange direct
    │   ├── Eliminar exchange topic
    │   ├── Eliminar exchange fanout
    │   └── Validar exchange inválido
    │
    ├── Bindings
    │   ├── Crear binding direct
    │   ├── Crear binding topic
    │   ├── Crear binding fanout
    │   ├── Eliminar binding direct
    │   ├── Eliminar binding topic
    │   └── Eliminar binding fanout
    │
    └── Validaciones
        ├── Crear cola con nombre vacío
        └── Crear binding sin exchangeType
```

Los requests de bindings utilizan el campo:

```text
exchangeType
```

para identificar si el binding corresponde a un exchange `direct`, `topic` o `fanout`.

---

## Secuencia recomendada de demostración

### Reservas y mensajería

```text
Crear reserva
      ↓
SOLICITADA
      ↓
EmailConsumer
      ↓
APROBADA
      ↓
EmailConsumer + PrepConsumer
      ↓
DEVUELTA
      ↓
EmailConsumer + VoucherConsumer
```

Los consumers anteriores pertenecen a:

```text
campuslab-ms-notify
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

### Persistencia AWS RDS

```text
Iniciar campuslab-ms-bookings con variables DB_*
      ↓
Conexión PostgreSQL AWS RDS
      ↓
Crear reserva
      ↓
GET /api/bookings
      ↓
Detener microservicio
      ↓
Iniciar nuevamente
      ↓
GET /api/bookings
      ↓
La reserva continúa disponible
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

Frontend:

```text
https://main.d35pphk36jv08s.amplifyapp.com
```

API Gateway:

```text
https://ok83lxm1s3.execute-api.us-east-1.amazonaws.com/api/bookings
```

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

La EP3 también incorpora persistencia cloud para reservas mediante PostgreSQL en AWS RDS.

---

## Despliegue en EC2

En una instancia EC2 con Docker y Git instalados:

```bash
git clone https://github.com/campuslab-projecto/campuslab-infra.git
cd campuslab-infra
docker compose up -d
```

Para un despliegue productivo se deben configurar los Security Groups correspondientes y evitar exponer directamente servicios internos que no requieran acceso público.

El despliegue cloud previo de los componentes principales se realizó durante la evaluación anterior.

La validación de RabbitMQ de EP3 fue realizada mediante Docker Compose local.

La base PostgreSQL de reservas fue validada mediante AWS RDS.

---

## Matriz de funcionalidades

La siguiente matriz sistematiza el estado actual del proyecto CampusLab.

| Funcionalidad | Estado | Componente responsable | Evidencia |
|---|---|---|---|
| Login con Azure AD / MSAL | Implementada y verificada | `campuslab-frontend` | `MsalGuard`, `loginRedirect`, `MsalInterceptor` |
| Envío automático de Bearer Token | Implementada y verificada | `campuslab-frontend` | `MsalInterceptor` + `protectedResourceMap` |
| Frontend Angular compila | Implementada y verificada | `campuslab-frontend` | `npm run build` completado correctamente |
| Frontend AWS Amplify | Implementada previamente | `campuslab-frontend` | Deploy AWS Amplify |
| Validación JWT | Implementada y verificada | `campuslab-bff` | `SecurityFilterChain`, OAuth2 Resource Server y `JwtDecoder` |
| Validación de issuer JWT | Implementada | `campuslab-bff` | `JwtValidators.createDefaultWithIssuer()` |
| Validación de audience JWT | Implementada | `campuslab-bff` | `AudienceValidator` |
| Conversión de roles Azure | Implementada | `campuslab-bff` | `AzureRolesConverter` |
| BFF compila | Implementada y verificada | `campuslab-bff` | Maven `BUILD SUCCESS` |
| Crear reserva | Implementada | `campuslab-ms-bookings` | `POST /api/bookings` |
| Consultar reservas | Implementada | `campuslab-ms-bookings` | `GET /api/bookings` |
| Consultar reserva por ID | Implementada | `campuslab-ms-bookings` | `GET /api/bookings/{id}` |
| Cambiar estado de reserva | Implementada | `campuslab-ms-bookings` | `PUT /api/bookings/{id}/status` |
| Entidad JPA Booking | Implementada | `campuslab-ms-bookings` | `@Entity` |
| Repositorio JPA | Implementada | `campuslab-ms-bookings` | `BookingRepository extends JpaRepository` |
| Integración con base de datos cloud | Implementada y probada | `campuslab-ms-bookings` / AWS RDS | PostgreSQL + JPA + conexión cloud verificada |
| Persistencia después de reinicio | Implementada y probada | `campuslab-ms-bookings` / AWS RDS | Reserva ID 1 permaneció después de reiniciar |
| Driver PostgreSQL | Implementada | `campuslab-ms-bookings` | Dependencia `org.postgresql:postgresql` |
| Configuración DB parametrizada | Implementada | `campuslab-ms-bookings` | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `DB_DRIVER`, `DB_DIALECT` |
| H2 para desarrollo local | Implementada | `campuslab-ms-bookings` | Fallback en `application.yml` |
| ms-bookings compila | Implementada y verificada | `campuslab-ms-bookings` | Maven `BUILD SUCCESS` |
| Evento `BOOKING_CREATED` | Implementada | `campuslab-ms-bookings` | `BookingMessageProducer` |
| Evento `BOOKING_STATUS_CHANGED` | Implementada | `campuslab-ms-bookings` | `BookingMessageProducer` |
| Productor RabbitMQ desacoplado | Implementada | `campuslab-ms-bookings` | Produce eventos sin contener consumers |
| Notificación email asíncrona | Implementada | `campuslab-ms-notify` | `q.cmd.email` + `EmailConsumer` |
| Ticket de preparación | Implementada | `campuslab-ms-notify` | `q.cmd.prep` + `PrepConsumer` |
| Voucher / acta de devolución | Implementada | `campuslab-ms-notify` | `q.cmd.voucher` + `VoucherConsumer` |
| ms-notify compila | Implementada y verificada | `campuslab-ms-notify` | Maven `BUILD SUCCESS` |
| Exchange direct | Implementada | RabbitMQ | `cmd.direct` |
| Exchange topic | Implementada | RabbitMQ | `cmd.topic` |
| Dead Letter Exchange | Implementada | RabbitMQ | `cmd.dead.dlx` |
| DLQ email | Implementada | RabbitMQ | `q.cmd.email.dlq` |
| DLQ preparación | Implementada | RabbitMQ | `q.cmd.prep.dlq` |
| DLQ voucher | Implementada | RabbitMQ | `q.cmd.voucher.dlq` |
| ACK manual | Implementada | `campuslab-ms-notify` | `basicAck()` |
| NACK sin requeue | Implementada | `campuslab-ms-notify` | `basicNack(..., false, false)` |
| Envío a DLQ | Implementada y probada | RabbitMQ / `campuslab-ms-notify` | `q.cmd.voucher.dlq Ready = 1` |
| Nombres RabbitMQ centralizados | Implementada | bookings / notify | Configuración en `application.yml` |
| Beans de Queue | Implementada | Configuración RabbitMQ | Beans explícitos |
| Beans de Exchange | Implementada | `campuslab-ms-bookings` | `DirectExchange` y `TopicExchange` |
| Beans de Binding | Implementada | `campuslab-ms-bookings` | Bindings direct, topic y DLQ |
| Consumidores `@RabbitListener` | Implementada | `campuslab-ms-notify` | Email, Prep y Voucher |
| Crear cola vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/queues` |
| Eliminar cola vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/queues/{name}` |
| Crear exchange vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/exchanges` |
| Eliminar exchange vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/exchanges/{name}` |
| Crear binding vía REST | Implementada | `campuslab-ms-rabbit-admin` | `POST /api/rabbit/bindings` |
| Eliminar binding vía REST | Implementada | `campuslab-ms-rabbit-admin` | `DELETE /api/rabbit/bindings` |
| Binding direct | Implementada | `campuslab-ms-rabbit-admin` | `exchangeType: direct` |
| Binding topic | Implementada | `campuslab-ms-rabbit-admin` | `exchangeType: topic` |
| Binding fanout | Implementada | `campuslab-ms-rabbit-admin` | `exchangeType: fanout` |
| Lógica administrativa encapsulada | Implementada | `campuslab-ms-rabbit-admin` | `RabbitAdminService` |
| Validación de nombre vacío | Implementada | `campuslab-ms-rabbit-admin` | `400 Bad Request` |
| Validación de tipo de exchange | Implementada | `campuslab-ms-rabbit-admin` | direct / topic / fanout |
| Validación de bindings | Implementada | `campuslab-ms-rabbit-admin` | DTO + Jakarta Validation |
| Manejo global de errores | Implementada | `campuslab-ms-rabbit-admin` | `GlobalExceptionHandler` |
| rabbit-admin compila | Implementada y verificada | `campuslab-ms-rabbit-admin` | Gradle `BUILD SUCCESSFUL` |
| Docker Compose local | Implementada | `campuslab-infra` | RabbitMQ, bookings, rabbit-admin, notify y BFF |
| API Gateway | Implementada previamente | AWS / BFF | Endpoint evaluación anterior |
| AWS RDS PostgreSQL | Implementada y probada | AWS / `campuslab-ms-bookings` | Conexión JDBC y persistencia real |
| Despliegue RabbitMQ en AWS | No requerido para la validación local realizada | `campuslab-infra` | RabbitMQ probado mediante Docker Compose |
| Catálogo | Fuera del alcance funcional actual | `campuslab-ms-catalog` | Repositorio reservado sin implementación funcional |
| Servicio notify independiente | Implementada y probada | `campuslab-ms-notify` | Consumers + ACK/NACK + Docker |
| Reportería | Fuera del alcance funcional actual | `campuslab-ms-report` | Repositorio reservado sin implementación funcional |
| Auditoría | Fuera del alcance funcional actual | `campuslab-ms-audit` | Repositorio reservado sin implementación funcional |

---

## Evidencias de EP3

Durante el desarrollo se verificó:

- Creación de las seis colas RabbitMQ.
- Creación de exchanges `direct`, `topic` y `dead`.
- Configuración de bindings direct.
- Configuración de bindings topic.
- Configuración de Dead Letter Exchange.
- Envío de eventos desde `campuslab-ms-bookings`.
- Separación de productores y consumidores.
- `campuslab-ms-bookings` quedó encargado de producir eventos.
- `campuslab-ms-notify` quedó encargado de consumir mensajes.
- Consumo de mensajes de email.
- Consumo de mensajes de preparación.
- Consumo de mensajes de voucher.
- ACK manual.
- NACK sin requeue.
- Prueba controlada de mensaje enviado a DLQ.
- Verificación de `q.cmd.voucher.dlq` con un mensaje disponible.
- Eliminación posterior del fallo controlado utilizado para probar la DLQ.
- Creación y eliminación de colas vía REST.
- Creación y eliminación de exchanges vía REST.
- Soporte de exchanges `direct`, `topic` y `fanout`.
- Creación y eliminación de bindings vía REST.
- Validación del campo `exchangeType`.
- Validaciones `400 Bad Request`.
- Manejo global de errores.
- Lógica administrativa encapsulada en `RabbitAdminService`.
- Ejecución de RabbitMQ mediante Docker.
- Ejecución de `campuslab-ms-bookings` mediante Docker.
- Ejecución de `campuslab-ms-rabbit-admin` mediante Docker.
- Ejecución de `campuslab-ms-notify` mediante Docker.
- Ejecución del BFF mediante Docker.
- Integración de los cinco contenedores mediante Docker Compose.
- Pruebas mediante Postman.
- Verificación visual mediante RabbitMQ Management.
- Compilación correcta de `campuslab-ms-bookings`.
- Compilación correcta de `campuslab-ms-notify`.
- Compilación correcta de `campuslab-ms-rabbit-admin`.
- Compilación correcta de `campuslab-bff`.
- Compilación correcta del frontend Angular.
- Validación de MSAL en frontend.
- Validación de JWT en BFF.
- Conexión real de `campuslab-ms-bookings` con PostgreSQL en AWS RDS.
- Creación de la tabla `bookings` mediante Hibernate.
- Persistencia de una reserva en AWS RDS.
- Persistencia verificada después de reiniciar el microservicio.

### Evidencia del desacoplamiento

Arquitectura final:

```text
campuslab-ms-bookings
        ↓
   Producer RabbitMQ
        ↓
      RabbitMQ
        ↓
campuslab-ms-notify
        ↓
EmailConsumer
PrepConsumer
VoucherConsumer
```

Los consumidores fueron eliminados de `campuslab-ms-bookings` y trasladados a `campuslab-ms-notify`.

### Evidencia de flujo SOLICITADA

Al crear una reserva:

```text
EMAIL recibido:
Reserva ID: 1
Estado: SOLICITADA
Tipo: BOOKING_CREATED
Mensaje: Reserva creada correctamente
Email procesado correctamente. ACK enviado.
```

### Evidencia de flujo APROBADA

Al cambiar una reserva a `APROBADA`:

```text
EMAIL recibido:
Estado: APROBADA
Email procesado correctamente. ACK enviado.
```

También se procesó:

```text
TICKET DE PREPARACIÓN recibido:
Estado: APROBADA
Ticket procesado correctamente. ACK enviado.
```

### Evidencia de flujo DEVUELTA

Al cambiar una reserva a `DEVUELTA`:

```text
EMAIL recibido:
Estado: DEVUELTA
Email procesado correctamente. ACK enviado.
```

También se procesó:

```text
VOUCHER recibido:
Estado: DEVUELTA
Voucher procesado correctamente. ACK enviado.
```

### Evidencia de ACK

Ejemplo:

```text
Email procesado correctamente. ACK enviado.
```

```text
Ticket procesado correctamente. ACK enviado.
```

```text
Voucher procesado correctamente. ACK enviado.
```

### Evidencia de NACK y DLQ

Durante una prueba controlada:

```text
Error procesando voucher: Fallo controlado para probar DLQ
NACK enviado. Mensaje derivado a DLQ.
```

Posteriormente:

```text
q.cmd.voucher.dlq    Ready = 1
```

El fallo intencional fue eliminado después de comprobar el funcionamiento de la DLQ.

### Evidencia de integración con AWS RDS

Se configuró `campuslab-ms-bookings` para trabajar con PostgreSQL en AWS RDS mediante variables de entorno.

La conectividad se verificó mediante:

```text
TcpTestSucceeded : True
```

Durante el inicio de Spring Boot se obtuvo:

```text
HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection
```

La aplicación utilizó:

```text
jdbc:postgresql://<RDS_ENDPOINT>:5432/campuslab
```

Hibernate inicializó la persistencia y creó la tabla:

```text
Hibernate: create table bookings (...)
```

Spring Boot inició correctamente:

```text
Tomcat started on port 8081
Started BookingsApplication
```

Posteriormente se creó una reserva:

```text
id          : 1
userEmail   : estudiante@campuslab.cl
resourceId  : LAB-101
status      : SOLICITADA
```

La reserva fue consultada mediante:

```text
GET /api/bookings
```

Luego se detuvo completamente `campuslab-ms-bookings` y se volvió a iniciar utilizando la misma conexión AWS RDS.

Después del reinicio, la misma reserva continuó disponible:

```text
id          : 1
userEmail   : estudiante@campuslab.cl
resourceId  : LAB-101
status      : SOLICITADA
```

Esto demuestra persistencia real de datos en PostgreSQL AWS RDS y confirma que la información no dependía de H2 en memoria.

### Evidencia de frontend y MSAL

El frontend fue compilado mediante:

```text
npm run build
```

Resultado:

```text
Application bundle generation complete.
```

Se verificó la utilización de:

```text
PublicClientApplication
MsalGuard
MsalInterceptor
MsalService
loginRedirect
protectedResourceMap
```

`MsalInterceptor` adjunta el Bearer Token a las llamadas protegidas hacia la API.

### Evidencia de validación JWT

En `campuslab-bff` se verificó:

```text
SecurityFilterChain
oauth2ResourceServer
JwtDecoder
NimbusJwtDecoder
BearerTokenAuthenticationEntryPoint
AudienceValidator
AzureRolesConverter
```

El backend valida:

- issuer del token.
- audience del token.
- Bearer Token.
- roles provenientes de Azure AD.

---

## Gestión del proyecto

El proyecto se gestiona mediante GitHub Projects y metodología Kanban.

Tablero:

```text
https://github.com/orgs/campuslab-projecto/projects/1/views/1
```

Flujo utilizado:

```text
Issue
  ↓
Rama feature / refactor / docs
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

Durante EP3 se utilizaron ramas separadas para:

- Implementar integración RabbitMQ.
- Implementar `campuslab-ms-notify`.
- Implementar RabbitMQ Admin.
- Refactorizar consumers.
- Incorporar PostgreSQL para AWS RDS.
- Actualizar infraestructura.
- Actualizar documentación.

---

## Compilación verificada

Durante la revisión final se verificó la compilación de los componentes desarrollados.

### campuslab-ms-bookings

```text
BUILD SUCCESS
```

### campuslab-ms-notify

```text
BUILD SUCCESS
```

### campuslab-ms-rabbit-admin

```text
BUILD SUCCESSFUL
```

### campuslab-bff

```text
BUILD SUCCESS
```

### campuslab-frontend

```text
Application bundle generation complete.
```

El warning de tamaño del bundle Angular no impide la compilación del proyecto.

---

## Estado final EP3

La solución CampusLab incluye:

- Frontend Angular.
- Autenticación Azure AD mediante MSAL.
- Bearer Token automático desde frontend.
- BFF Spring Boot.
- Validación JWT.
- Validación de issuer y audience.
- Microservicio de reservas.
- JPA y repositorios.
- Persistencia cloud mediante PostgreSQL AWS RDS.
- RabbitMQ.
- Exchanges direct, topic y dead letter.
- Colas de email, preparación y voucher.
- Dead Letter Queues.
- Productores RabbitMQ.
- Consumers desacoplados en `campuslab-ms-notify`.
- `@RabbitListener`.
- ACK manual.
- NACK sin requeue.
- Manejo explícito de errores.
- Flujo a DLQ probado.
- RabbitMQ Admin REST.
- Administración de colas.
- Administración de exchanges.
- Administración de bindings.
- Exchanges direct, topic y fanout mediante la API administrativa.
- Jakarta Validation.
- Manejo global de errores.
- Docker Compose.
- Postman.
- RabbitMQ Management.
- Matriz de funcionalidades.
- Evidencias de pruebas.
- Gestión mediante ramas y Pull Requests.

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

Integrantes:

```text
Lukas Meza Cofré
Christian Sandoval
```