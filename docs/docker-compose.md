# Ejecución completa con Docker Compose

Desde biblioteca-plataforma:
`docker compose up -d --build --wait`.

El archivo canónico es compose.yml, proyecto biblioteca, con ocho servicios.
Consulta el README para preparar los .env, ver los puertos y ejecutar los experimentos.

Los volúmenes datos-rabbit y datos-postgres tienen nombre explícito. El hostname rabbit1
conserva la identidad Erlang. RabbitMQ usa check_port_connectivity y Postgres pg_isready.
Worker y préstamos esperan a los dos con condition: service_healthy.
El healthcheck del worker consulta /salud, que comprueba broker y SELECT 1.
El healthcheck de préstamos también comprueba la base y su publicador.
BFF/gateway/libros no abren conexiones a dependencias al arrancar.

Después de editar un servicio, reconstruye su imagen:
`docker compose up -d --build --wait eventos`.
Para detener conservando datos: `docker compose down`.

compose.orden-sin-espera.yml reproduce ECONNREFUSED y compose.orden.yml lo corrige.
Son proyectos separados, sin los volúmenes persistentes del laboratorio.

.env y sus variantes están ignorados. .dockerignore impide incluir secretos y
node_modules de Windows en imágenes Linux. npm ci usa package-lock.json.
Angular compila con Node 24 y se sirve con nginx:1.29-alpine con fallback a index.html.
