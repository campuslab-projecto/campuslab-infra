# Estado verificado de CampusLab - EP5

Fecha de revision: 20 de noviembre de 2026.

## Arquitectura AWS

EC2 Backend (172.31.20.121):
- BFF: puerto 8080.
- Reservas: puerto 8081.
- RabbitMQ Admin: puerto 8082.
- Notificaciones: puerto 8083.
- Dos instancias RabbitMQ.

EC2 Kafka (172.31.18.59):
- Un broker Kafka.
- Auditoria: puerto 8084.
- Reporteria: puerto 8085.
- Kafka Admin: puerto 8086.

## Kafka

Entorno local:
- Docker Compose con ZooKeeper, tres brokers y Kafka UI.
- Replicacion predeterminada configurada en 3.
- min.insync.replicas configurado en 2.
- Configuracion validada estaticamente.
- Existen evidencias anteriores de ejecucion.

AWS:
- Un broker con ID 1.
- Topicos campuslab.booking.events y campuslab.occupancy.events.
- Una particion por topico.
- Factor de replicacion 1.
- Retencion de 24 horas.
- cleanup.policy=delete.

## Integracion

- Reservas publica eventos Kafka.
- Auditoria consume eventos de reservas.
- Reporteria consume eventos de reservas y ocupacion.
- Grupos independientes: campuslab-audit-group y campuslab-report-group.
- LAG 0 registrado durante la inspeccion de AWS.

## Administracion Kafka

Se comprobaron operaciones de:
- Consulta y creacion de topicos.
- Modificacion de configuracion.
- Aumento de particiones.
- Eliminacion de topicos de prueba.
- Consulta de consumer groups.

## Manejo de errores y DLT

- Existen evidencias de pruebas aisladas de Auditoria y Reporteria.
- Se registraron reintentos ante eventos invalidos.
- Se documento el procesamiento posterior de eventos validos.
- Se incorporo una prueba automatizada Embedded Kafka en MS Audit.
- La prueba DLT de MS Audit fue aprobada localmente.
- La version DLT actualizada de MS Audit no se ha comprobado en el despliegue AWS actual.

## RabbitMQ

En AWS se verificaron:
- Dos instancias saludables.
- Tres colas principales con consumidores conectados.
- Tres DLQ.
- Exchanges direct, topic y dead letter.
- Bindings y configuracion x-dead-letter.
- Evidencia anterior de rechazo de mensaje hacia DLQ.

## Seguridad y persistencia

- El BFF respondio HTTP 401 para endpoints protegidos sin JWT.
- Reporteria dispone de una base H2 en volumen persistente.
- Reservas utiliza H2 persistente en el despliegue EP5 comprobado.
- PostgreSQL AWS RDS corresponde a una integracion anterior.

## Docker y GitHub

- BFF desplegado: campuslab-bff:report-v1.
- Reservas desplegado: campuslab-ms-bookings:kafka-aws.
- Las imagenes anteriores necesitan construccion o distribucion para reproducirse.
- La carpeta de Compose del backend AWS no es un repositorio Git.
- Los archivos aws-backend y aws-bookings locales no reproducen exactamente el AWS actual.

## Evidencias

El directorio evidencias-ep5 contiene 28 archivos versionados:
- 01-08: servicios, topicos, auditoria, reportes y Kafka Admin.
- 09-14: pruebas DLT, offsets y reportes.
- 15-17: operaciones Kafka Admin.
- 18-24: RabbitMQ y RabbitMQ Admin.
- 25-26: tres brokers Kafka, ZooKeeper y Kafka UI.
- 27: proteccion JWT del BFF.
- 28: persistencia H2 de Reporteria.

## Consideraciones

- Diferenciar entorno local de tres brokers y AWS de un broker.
- No publicar credenciales ni archivos .env.
- Revisar la red Docker externa antes del despliegue local.
- No ejecutar los Compose AWS alternativos sobre los servicios actuales.
- Mantener los cambios de codigo mediante IntelliJ, GitHub y Pull Request.
