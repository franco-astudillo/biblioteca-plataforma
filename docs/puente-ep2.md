# Puente del L7 a VidalStore · EP2

Esta carpeta implementa Biblioteca. La transferencia al dominio del encargo requiere
repositorios y nombres propios, sin dejar biblioteca.* ni prestamo.* en VidalStore.

| Biblioteca | VidalStore |
|---|---|
| biblioteca.eventos/comandos/dlx | vidalstore.eventos/comandos/dlx |
| prestamo.creado | compra.realizada |
| prestamo.devuelto | licencia.revocada |
| libro.agotado manual | juego.publicado por catálogo |
| prestamos.mjs · esquema prestamos | Microservicio de licencias · esquema licencias |
| Tres tablas del worker | Mismas tres tablas del worker vidalstore-eventos |

La cola de avisos debe recibir licencia.revocada; copiar prestamo.* como compra.*
dejaría la revocación fuera. juego.publicado no trae usuarioSub, así que se validan
los campos obligatorios por routing key.

Licencias conserva usuario_sub del jugador, estado activa/revocada, adquirida_en,
revocada_en y revocada_por. El UNIQUE(usuario_sub,juego_id) debe ser parcial,
WHERE estado='activa', para permitir una compra posterior a la revocación.
El usuarioSub de licencia.revocada sale de la fila del jugador; revocadaPor del token
del administrador. El ER debe documentar ambos y la frontera entre dueños.

Los seeds y el catálogo JSON de EP1 siguen versionados. Licencias empieza vacía.
IE1–5 e IE12–16 quedan cubiertos por la topología, módulos, ACK, DLQ y persistencia;
Compose aporta parte de IE11. L8 agrega clúster de dos nodos, administrador,
métricas y políticas de retención. Este L7 no implementa esos componentes.

En la EP2 se trabaja en ramas feature/* desde dev; no se publican cambios directos en main.
