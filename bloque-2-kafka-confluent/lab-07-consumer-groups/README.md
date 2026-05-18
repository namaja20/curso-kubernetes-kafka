# LAB 7 — Consumer groups y rebalanceos

> **Entorno.** Pod **`kafka-cli`** (Lab 5). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Llegada desde el [Capítulo 3 — Consumer groups y offsets](../fundamentos/03-consumer-groups-offsets.md).**

[← Volver: Capítulo 3](../fundamentos/03-consumer-groups-offsets.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Materializar la regla de los **consumer groups**: una partición la lee un solo consumidor del grupo; grupos distintos leen en paralelo (pub/sub); al cambiar miembros ocurre **rebalanceo**.

## Prerrequisitos

- Lab 5 operativo.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="notif-lab07"
```

## Paso 1 — Preparar topic y datos

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 3 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- bash -c '
BS="kafka.kafka.svc.cluster.local:9092"
for i in $(seq 1 9); do echo "evt-$i"; done | \
  kafka-console-producer --bootstrap-server "$BS" --topic notif-lab07
'
```

## Paso 2 — Dos grupos distintos (broadcast entre grupos)

**Grupo `equipo-a`** — lee todo desde el inicio:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group equipo-a --from-beginning --timeout-ms 5000
```

**Grupo `equipo-b`** — misma operación:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group equipo-b --from-beginning --timeout-ms 5000
```

Ambos grupos leen los **9 mensajes**: distintos `group.id` = distintos punteros (modelo pub/sub del capítulo 1).

## Paso 3 — Dos consumidores en el mismo grupo (reparto)

Abrir **dos terminales** con shell en `kafka-cli`:

Terminal A:

```bash
kubectl -n app-a exec -it deploy/kafka-cli -- bash
kafka-console-consumer --bootstrap-server "$BS" --topic "$TOPIC" \
  --group procesadores --from-beginning
```

Terminal B (otra sesión):

```bash
kubectl -n app-a exec -it deploy/kafka-cli -- bash
kafka-console-consumer --bootstrap-server "$BS" --topic "$TOPIC" \
  --group procesadores --from-beginning
```

Cada terminal recibe un subconjunto de particiones; entre las dos se reparten los 9 mensajes (no duplicados dentro del grupo).

En una tercera terminal:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" --describe --group procesadores
```

Columna `CONSUMER-ID` muestra qué consumidor tiene cada partición.

## Paso 4 — Provocar rebalanceo

Con los dos consumidores del paso 3 activos, **cerrar uno** (`Ctrl+C` en una terminal).

Vuelve a ejecutar:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" --describe --group procesadores
```

El consumidor superviviente **hereda** las particiones del que salió. Eso es un **rebalanceo**.

Añadir un tercer consumidor al mismo grupo y observar de nuevo el `--describe`: las particiones se redistribuyen (3 particiones, 3 consumidores → uno por partición si hay equilibrio).

## Comprobaciones

- [ ] `equipo-a` y `equipo-b` leen los 9 mensajes cada uno.
- [ ] `procesadores` reparte mensajes entre dos consumidores sin duplicar dentro del grupo.
- [ ] Tras matar un consumidor, `--describe` muestra reassignment.

## Retos opcionales

1. Lanzar un cuarto consumidor con el mismo `group.id`: uno quedará inactivo (más consumidores que particiones).
2. `kafka-consumer-groups --bootstrap-server "$BS" --describe --group procesadores --state`.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| Un consumidor lee todo solo | El otro no comparte `group.id` o no arrancó a tiempo. |
| Rebalanceo continuo | Demasiados consumidores para las particiones; revisar heartbeats. |

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic notif-lab07
```

---

[← Volver: Capítulo 3](../fundamentos/03-consumer-groups-offsets.md) · [Siguiente capítulo: 4 — Replicación e ISR →](../fundamentos/04-replicacion-isr.md)
