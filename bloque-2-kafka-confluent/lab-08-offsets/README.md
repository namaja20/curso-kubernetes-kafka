# LAB 8 — Offsets y control de consumo

> **Entorno.** Pod **`kafka-cli`** (Lab 5). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Relacionado con el [Capítulo 3 — Consumer groups y offsets](../fundamentos/03-consumer-groups-offsets.md)** (2ª parte: commits, earliest/latest, reset).

[← Volver: Capítulo 3](../fundamentos/03-consumer-groups-offsets.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Gestionar **offsets** de un consumer group: lectura desde el inicio, desde el final, y **reset** explícito con `kafka-consumer-groups`.

## Prerrequisitos

- Labs 5 y 7 recomendados.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="offsets-lab08"
GROUP="grupo-offsets"
```

## Paso 1 — Cargar histórico en el topic

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 3 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- bash -c '
for i in $(seq 1 20); do echo "historico-$i"; done | \
  kafka-console-producer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic offsets-lab08
'
```

## Paso 2 — `earliest` (leer histórico)

Primer arranque del grupo con `--from-beginning`:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group "$GROUP" --from-beginning --timeout-ms 8000
```

Deberían aparecer los 20 mensajes. El offset queda registrado en `__consumer_offsets`.

## Paso 3 — `latest` (solo mensajes nuevos)

Crear un **grupo nuevo** sin `--from-beginning`:

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c '
echo "mensaje-nuevo-1" | kafka-console-producer \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 --topic offsets-lab08
'

kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic offsets-lab08 --group grupo-solo-nuevos --timeout-ms 5000
'
```

Solo se ve lo publicado **después** de que el grupo existe (comportamiento `auto.offset.reset=latest` por defecto en clientes sin offset previo).

## Paso 4 — Reset de offsets a `earliest`

Con el grupo `grupo-offsets` que ya consumió, forzar releer desde el inicio:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" \
  --group "$GROUP" --topic "$TOPIC" \
  --reset-offsets --to-earliest --dry-run

kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" \
  --group "$GROUP" --topic "$TOPIC" \
  --reset-offsets --to-earliest --execute
```

Volver a consumir:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group "$GROUP" --from-beginning --timeout-ms 8000
```

Los 20 mensajes vuelven a aparecer.

## Paso 5 — Reset a `latest`

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" \
  --group "$GROUP" --topic "$TOPIC" \
  --reset-offsets --to-latest --execute
```

El siguiente consumo del grupo no verá histórico pendiente.

## Comprobaciones

- [ ] `--from-beginning` lee los 20 mensajes iniciales.
- [ ] Grupo nuevo sin offset previo no ve histórico (solo nuevos).
- [ ] `--reset-offsets --to-earliest --execute` permite releer.
- [ ] `--to-latest` deja el grupo al día con el final del log.

## Retos opcionales

1. Reset por partición concreta: añadir `--partition 0` al comando de reset.
2. Inspeccionar `kafka-consumer-groups --describe --group "$GROUP" --state`.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| Reset rechazado | Parar todos los consumidores del grupo antes del reset. |
| `--dry-run` obligatorio en prod | En el lab usar siempre dry-run antes de `--execute`. |

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic offsets-lab08
```

---

[← Volver: Capítulo 3](../fundamentos/03-consumer-groups-offsets.md) · [Siguiente capítulo: 4 — Replicación e ISR →](../fundamentos/04-replicacion-isr.md)
