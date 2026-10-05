# Ejecución completa con Docker Compose

Este archivo orquesta RabbitMQ, PostgreSQL, los microservicios de libros y préstamos,
el worker de eventos, el BFF, el gateway y la aplicación Angular.

## Estructura requerida

Los cinco repositorios deben estar clonados como carpetas hermanas:

```text
Laboratorio 6/
├── L1-gateway/
├── biblioteca-bff/
├── biblioteca-eventos/
├── biblioteca-plataforma/
└── biblioteca-web/
```

## Configuración local

Desde `biblioteca-plataforma`, copia `.env.example` como `.env` y completa los
valores reales de Cognito. El archivo `.env` está ignorado por Git y no debe
subirse al repositorio.

## Arranque y comprobación

```powershell
cd "D:\Laboratorio 6\biblioteca-plataforma"
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs -f eventos
```

Servicios expuestos:

- Angular: `http://localhost:4200`
- Gateway: `http://localhost:8080`
- BFF: `http://localhost:3000`
- Libros: `http://localhost:3001`
- Préstamos: `http://localhost:3002`
- Worker: `http://localhost:3010/salud`
- RabbitMQ Management: `http://localhost:15672`
- PostgreSQL: `localhost:5432`

Para detener los contenedores conservando sus volúmenes:

```powershell
docker compose down
```

`docker compose down -v` también elimina los datos persistentes y solo debe
usarse cuando se quiera reiniciar completamente el laboratorio.
