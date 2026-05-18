# LAB 14 — Pipeline completo (Producer → Topic → Consumer)

> **Entorno.** Stack del Lab 5 (+ Lab 12 opcional si Connect sigue desplegado). Pod **`kafka-cli`** para el pipeline Kafka.

> **Llegada desde el [Capítulo 2 — Pipeline extremo a extremo](../fundamentos/02-pipeline-extremo-a-extremo.md).**

[← Volver: Capítulo 2 — Pipeline](../fundamentos/02-pipeline-extremo-a-extremo.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Inicio del curso ↺](../../README.md)

---

## Objetivo

Recorrer la **checklist mental** del pipeline integrado: produce → topic (partición, ISR) → consume → offset/lag, observando **Kubernetes** y **Kafka** en cada salto.

## Prerrequisitos

- Labs 5 y 13 completados.
- Brokers `Ready`, `kafka-cli` operativo.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="pipeline-lab14"
GROUP="pipeline-demo"
```

## Paso 1 — Salud del cluster (K8s + Kafka)

```bash
kubectl -n kafka get pods -l app=kafka
kubectl -n app-a exec deploy/kafka-cli -- kafka-broker-api-versions \
  --bootstrap-server "$BS" | head -3
kubectl -n app-a exec deploy/kafka-cli -- kafka-metadata-quorum \
  --bootstrap-server "$BS" describe --status
```

## Paso 2 — Crear topic y localizar líderes

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 3 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC"
```

Anotar qué broker es **Leader** de la partición 0.

## Paso 3 — Produce (con clave)

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c '
cat <<EOF | kafka-console-producer --bootstrap-server '"$BS"' --topic '"$TOPIC"' \
  --property parse.key=true --property key.separator=:
orden-1:{"evento":"creado","importe":10}
orden-1:{"evento":"pagado","importe":10}
orden-2:{"evento":"creado","importe":25}
EOF
'
```

Verificar offsets:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-get-offsets \
  --bootstrap-server "$BS" --topic "$TOPIC"
```

## Paso 4 — Consume con consumer group

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server "$BS" --topic "$TOPIC" \
  --group "$GROUP" --from-beginning --timeout-ms 10000
```

## Paso 5 — Offsets y lag

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-consumer-groups \
  --bootstrap-server "$BS" --describe --group "$GROUP"
```

Lag debería ser **0** tras consumo completo.

## Paso 6 — Reinicio del consumidor (misma app, nuevo pod)

Simular redeploy de la app consumidora: el grupo retoma donde quedó.

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c \
  'echo orden-3:{"evento":"creado","importe":99} | kafka-console-producer \
  --bootstrap-server '"$BS"' --topic '"$TOPIC"' \
  --property parse.key=true --property key.separator=:'

kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server '"$BS"' --topic '"$TOPIC"' \
  --group '"$GROUP"' --timeout-ms 5000
'
```

Solo debe aparecer el mensaje nuevo (offset ya commitado para los anteriores).

## Paso 7 — (Opcional) Extremo Connect

Si el [Lab 12](../../bloque-2-kafka-confluent/lab-12-kafka-connect/README.md) sigue activo:

```bash
kubectl -n lab12 exec deploy/connect -- curl -s http://localhost:8083/connectors
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --list | grep jdbc
```

Pipeline ampliado: **PostgreSQL → Connect → Topic → Consumer**.

## Comprobaciones

- [ ] Topic con RF 3 e ISR completo antes de producir.
- [ ] Mensajes con misma clave `orden-1` en la misma partición (verificar con consumo por `--partition` si se desea).
- [ ] Consumer group registra offsets; segundo consume no relee histórico ya commitado.
- [ ] Se ha pasado por checklist K8s (pods) y Kafka (describe, groups, quorum).

## Retos opcionales

1. Durante el consume, `kubectl -n kafka delete pod` del broker líder de la partición 0 y observar recuperación sin perder el ejercicio.
2. Documentar en una tabla los cuatro saltos del capítulo 2 (descubrimiento, partición, ISR, commit) con el comando usado en cada uno.

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic pipeline-lab14
```

---

[← Volver: Capítulo 2 — Pipeline](../fundamentos/02-pipeline-extremo-a-extremo.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Inicio del curso ↺](../../README.md)
