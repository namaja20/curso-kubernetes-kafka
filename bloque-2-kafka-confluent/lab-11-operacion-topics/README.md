# LAB 11 — Operación de topics

> **Entorno.** Pod **`kafka-cli`** (Lab 5). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Llegada desde el [Capítulo 5 — KRaft](../fundamentos/05-kraft.md).** Cada cambio de configuración es metadato gestionado por el quorum.

[← Volver: Capítulo 5 — KRaft](../fundamentos/05-kraft.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Operar topics existentes: consultar y **alterar retención**, aumentar particiones y verificar el efecto con las herramientas de línea de comandos (base para GitOps/Tekton en producción).

## Prerrequisitos

- Lab 5 operativo.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="ops-lab11"
```

## Paso 1 — Crear topic de trabajo

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 3 --replication-factor 3 \
  --config retention.ms=604800000

kubectl -n app-a exec deploy/kafka-cli -- kafka-configs \
  --bootstrap-server "$BS" \
  --entity-type topics --entity-name "$TOPIC" --describe
```

`retention.ms=604800000` son 7 días (valor por defecto habitual).

## Paso 2 — Reducir retención a 1 hora

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-configs \
  --bootstrap-server "$BS" \
  --entity-type topics --entity-name "$TOPIC" \
  --alter --add-config retention.ms=3600000

kubectl -n app-a exec deploy/kafka-cli -- kafka-configs \
  --bootstrap-server "$BS" \
  --entity-type topics --entity-name "$TOPIC" --describe
```

Los segmentos antiguos se purgarán según la política del broker (no es instantáneo en todos los casos).

## Paso 3 — Límite por tamaño

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-configs \
  --bootstrap-server "$BS" \
  --entity-type topics --entity-name "$TOPIC" \
  --alter --add-config retention.bytes=104857600
```

Con esto el topic no crecerá indefinidamente en disco más allá de ~100 MiB (suma de particiones según versión/config).

## Paso 4 — Aumentar particiones

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --alter --topic "$TOPIC" --partitions 6

kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC"
```

Solo se **aumentan** particiones; reducir no está soportado. Nuevas particiones solo reciben mensajes **sin clave** o con claves nuevas según hash.

## Paso 5 — Verificar metadatos (KRaft)

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-metadata-quorum \
  --bootstrap-server "$BS" describe --status
```

Cada `alter` de topic es un cambio en el log de metadatos del cluster.

## Comprobaciones

- [ ] `kafka-configs --describe` muestra `retention.ms` y `retention.bytes` aplicados.
- [ ] El topic pasa de 3 a 6 particiones.
- [ ] `kafka-metadata-quorum` responde sin error.

## Retos opcionales

1. Borrar una config: `--alter --delete-config retention.bytes`.
2. Exportar definición del topic (particiones, RF, configs) a un YAML en Git como borrador de futuro `KafkaTopic` (CFK) o pipeline Tekton.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| `UnknownTopicOrPartition` | Nombre de topic incorrecto. |
| No se aplican configs | Revisar `--entity-type topics` y nombre exacto. |

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic ops-lab11
```

---

[← Volver: Capítulo 5 — KRaft](../fundamentos/05-kraft.md) · [Siguiente capítulo: 6 — Schema Registry →](../fundamentos/06-schema-registry.md)
