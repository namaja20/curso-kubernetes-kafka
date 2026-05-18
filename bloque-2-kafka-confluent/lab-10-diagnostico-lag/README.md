# LAB 10 — Diagnóstico de lag

> **Entorno.** Pod **`kafka-cli`** (Lab 5). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Relacionado con el [Capítulo 4 — Replicación e ISR](../fundamentos/04-replicacion-isr.md)** (indicador operativo: lag).

[← Volver: Capítulo 4](../fundamentos/04-replicacion-isr.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Generar **lag** en un consumer group, medirlo con `kafka-consumer-groups --describe` e interpretar si el consumo va al día.

## Prerrequisitos

- Lab 5 operativo.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="lag-lab10"
GROUP="procesadores-lag"
```

## Paso 1 — Acumular mensajes sin consumidor

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 3 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- bash -c '
for i in $(seq 1 200); do echo "carga-$i"; done | \
  kafka-console-producer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic lag-lab10
'
```

## Paso 2 — Consumir solo una parte (dejar lag)

Leer 50 mensajes y salir (el grupo hace commit si auto-commit está activo en el consumer):

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group "$GROUP" --max-messages 50 --timeout-ms 15000
```

## Paso 3 — Más producción con el grupo parado

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c '
for i in $(seq 201 350); do echo "carga-$i"; done | \
  kafka-console-producer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic lag-lab10
'
```

## Paso 4 — Medir lag

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" --describe --group "$GROUP"
```

Por cada partición:

| Columna | Significado |
|---------|-------------|
| `LOG-END-OFFSET` | Último offset escrito en la partición |
| `CURRENT-OFFSET` | Último offset confirmado por el grupo |
| `LAG` | Diferencia (mensajes pendientes) |

Lag **creciente** de forma sostenida = el consumo no alcanza la producción.

## Paso 5 — Reducir lag

Consumir hasta ponerse al día:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group "$GROUP" --timeout-ms 20000
```

Volver a ejecutar `--describe`: `LAG` debería ser **0** (o cercano) en todas las particiones.

## Comprobaciones

- [ ] Tras producción sin consumidor, al primer `--describe` el grupo puede no existir o tener lag alto tras consumo parcial.
- [ ] Tras el paso 4, `LAG` > 0 en al menos una partición.
- [ ] Tras el paso 5, `LAG` ≈ 0.

## Retos opcionales

1. Añadir un segundo consumidor al mismo grupo y observar cómo baja el lag más rápido (reparto de particiones).
2. `watch -n2 kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups ... --describe --group "$GROUP"` mientras consume.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| LAG siempre 0 | El consumer leyó todo; producir más sin consumir de nuevo. |
| Grupo no aparece | Arrancar al menos un consumer con ese `group.id` una vez. |

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic lag-lab10
```

---

[← Volver: Capítulo 4](../fundamentos/04-replicacion-isr.md) · [Siguiente capítulo: 5 — KRaft →](../fundamentos/05-kraft.md)
