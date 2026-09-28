# Modelo de datos · biblioteca

```text
┌──────────────────────────────────────────┐
│ mensajes_taller                          │
├──────────────────────────────────────────┤
│ PK  id           uuid                    │
│     routing_key  text         NOT NULL   │
│     payload      jsonb        NOT NULL   │
│     recibido_en  timestamptz  NOT NULL   │
└──────────────────────────────────────────┘
```