# Biblioteca · Laboratorio 7

Ocho servicios, tres colas de trabajo con su DLQ y cuatro tablas de PostgreSQL.
El productor guarda el préstamo antes de publicar; el worker confirma después de persistir.

## Levantar desde un clon limpio

Los cinco repositorios deben ser carpetas hermanas:
`L1-gateway`, `biblioteca-bff`, `biblioteca-web`, `biblioteca-eventos` y `biblioteca-plataforma`.

Copia los ejemplos de entorno en cada repositorio, completa Cognito en el gateway,
BFF y servicios, y usa las mismas credenciales de RabbitMQ y PostgreSQL en plataforma,
worker y servicios. El frontend conserva la configuración pública de Cognito de L6.

```powershell
cd "D:\Laboratorio 6\biblioteca-plataforma"
Copy-Item .env.example .env
Copy-Item ../biblioteca-eventos/.env.example ../biblioteca-eventos/.env
Copy-Item ../L1-gateway/servicios/.env.example ../L1-gateway/servicios/.env
Copy-Item ../L1-gateway/gateway/.env.example ../L1-gateway/gateway/.env
Copy-Item ../biblioteca-bff/.env.example ../biblioteca-bff/.env
# Completar los archivos .env antes de continuar.
docker compose config --quiet
docker compose up -d --build --wait
docker compose ps
```

| Servicio | Acceso desde el PC |
|---|---|
| Angular | http://localhost:4200 |
| Gateway | http://localhost:8080 |
| BFF | http://localhost:3000 |
| Libros | http://localhost:3001 |
| Préstamos | http://localhost:3002 |
| Worker y sus dependencias | http://localhost:3010/salud |
| RabbitMQ Management | http://localhost:15672 |
| PostgreSQL | localhost:5432 |

Dentro de la red, el broker se llama `rabbit1` y la base `postgres`.
`environment` de Compose reemplaza los hosts localhost de `env_file`.
Solo worker y préstamos abren conexiones al arrancar y esperan `service_healthy`.
`--wait` espera los healthchecks antes de devolver el control al usuario.
Si una dependencia cae después, `/salud` responde 503 y las escrituras tienen reintentos.

## Datos del L6 y credenciales

Se conservan `datos-rabbit` y `datos-postgres`, con nombre explícito.
Antes de cambiar desde el Compose anterior revisa `docker volume ls` y los montajes
con `docker inspect`: si usabas los volúmenes `biblioteca-l6_*`, respáldalos y cambia
los campos `volumes.*.name` para reutilizarlos. No borres un volumen para arreglar un arranque.

Los valores POSTGRES_PASSWORD y RABBITMQ_DEFAULT_PASS inicializan un volumen vacío;
editar .env no cambia las credenciales dentro de un volumen existente.
RabbitMQ mantiene la identidad `rabbit@rabbit1` gracias a `hostname: rabbit1`.
La tabla de práctica `mensajes_taller` es histórica del L6, ajena a las entidades del worker.

El catálogo se lee del JSON versionado `L1-gateway/datos/catalogo.json`.
El script `L1-gateway/sembrar.mts` documenta su origen externo.
Los préstamos del JSON del L6 no se importan: la tabla nueva empieza vacía.

## Mensaje envenenado

Desde `biblioteca-plataforma`:

```powershell
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 3
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1 --roto
docker compose logs --tail 60 eventos
docker compose exec postgres psql -U biblioteca -d biblioteca -c "select routing_key, count(*) from eventos_auditoria group by routing_key;" -c "select cola_origen, motivo, intentos, left(payload,30) from mensajes_muertos;" -c "select count(*) from notificaciones;"
```

En una base nueva: 3 eventos, 3 notificaciones y 2 cartas muertas con motivo rejected,
una de auditoría y una de notificaciones. El payload roto coincide con los dos bindings.
Las DLQ pueden estar vacías porque CartasMuertasConsumidor ya las guardó y confirmó.

Para inspeccionar `x-death` antes de persistir, pausa solo el consumidor de DLQ:

```powershell
$env:CONSUMIR_DLQ = "false"
docker compose up -d --wait eventos
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1 --roto
docker compose run --rm --no-deps eventos node herramientas/ver-dlq.mjs auditoria.dlq
Remove-Item Env:CONSUMIR_DLQ
docker compose up -d --wait eventos
```

`ver-dlq.mjs` lee con ack manual y devuelve el mensaje: no lo borra.

## ACK, prefetch y recuperación

En una cola sin mensajes previos, detén el worker y publica cinco:

```powershell
docker compose stop eventos
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 5
docker compose run --rm --no-deps eventos node herramientas/consumidor-lento.mjs
```

Desde otra terminal:
`docker compose exec rabbit1 rabbitmqctl list_queues name messages messages_ready messages_unacknowledged`.
Con prefetch 1: auditoria = 5/4/1; al cortar el consumidor: 5/5/0.
Con `consumidor-lento.mjs 5`: 5/0/5. Al cortar vuelven los cinco.
Luego `docker compose up -d --wait eventos`.

Exchanges y colas durables más mensajes persistent conservan estructura y contenido.
Los publicadores usan confirmaciones del broker antes de terminar o confirmar la entrada.

## Idempotencia y fallos de base

```powershell
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1 --repetido
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1 --repetido
docker compose exec postgres psql -U biblioteca -d biblioteca -c "select count(*) from eventos_auditoria where evento_id = '11111111-2222-3333-4444-555555555555';"
docker compose stop postgres
curl.exe -i http://localhost:3010/salud
docker compose run --rm --no-deps eventos node herramientas/emitir.mjs 1
docker compose logs --tail 40 eventos
docker compose up -d --wait postgres
```

El duplicado deja una fila de auditoría y recibe ack.
La base caída produce dos WARN, espera 2 y 4 segundos y rechaza al tercer fallo.
Las cartas muertas permanecen en su DLQ con reintento cada 5 segundos hasta recuperar la base.
`notificaciones.evento_id` tiene índice no único: puede haber varias notificaciones por evento.

## Experimento del orden

```powershell
docker compose -f compose.orden-sin-espera.yml up -d
docker compose -f compose.orden-sin-espera.yml logs cliente
docker compose -f compose.orden-sin-espera.yml down -v
docker compose -f compose.orden.yml up -d
docker compose -f compose.orden.yml logs cliente
docker compose -f compose.orden.yml down -v
```

El primero resuelve postgres pero recibe connection refused; el segundo imprime
«la base contesto». Estos dos proyectos son aislados y no montan los volúmenes del laboratorio.

## Migrar la topología de L6

`assertQueue` no modifica argumentos de una cola existente.
Si aparece 406 PRECONDITION_FAILED, detén los productores/worker y comprueba primero
mensajes ready y unacked. En el laboratorio, solo después de conservar cualquier dato
necesario, puedes borrar las tres colas de trabajo vacías y dejar que el worker las declare.
Las DLQ se declaran antes que las colas de trabajo.
En producción se migra a colas nuevas o se aplican políticas revisadas, sin borrar mensajes.

## Cambios, parada y entregables

```powershell
docker compose up -d --build --wait eventos prestamos
docker compose logs -f eventos
docker compose down
```

`down` conserva datos. `down -v` elimina los datos persistentes; no es la parada normal.
No hay recarga automática: reconstruye la imagen del servicio que cambiaste.

- [Modelo de datos y dueños](docs/modelo-de-datos.md)
- [Topología y contratos](docs/topologia.md)
- [Guía de ejecución](docs/docker-compose.md)
- [Puente a EP2](docs/puente-ep2.md)

TypeORM usa synchronize únicamente para este laboratorio, con advertencia en el código.
Préstamos y eventos no forman una transacción distribuida: los publisher confirms comprueban
aceptación del broker, pero una caída entre INSERT y publish requiere un outbox para recuperación
automática en producción. No se implementa outbox en L7.

Referencias de implementación: [Nest TypeORM](https://docs.nestjs.com/techniques/database),
[RabbitMQ DLX](https://www.rabbitmq.com/docs/dlx),
[índices parciales de PostgreSQL](https://www.postgresql.org/docs/current/indexes-partial.html).
