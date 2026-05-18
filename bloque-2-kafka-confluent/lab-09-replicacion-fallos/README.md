# LAB 9 — Replicación y tolerancia a fallos

> **Entorno.** Pod **`kafka-cli`** + brokers en `kafka` (Lab 5). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Llegada desde el [Capítulo 4 — Replicación e ISR](../fundamentos/04-replicacion-isr.md).**

[← Volver: Capítulo 4](../fundamentos/04-replicacion-isr.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Simular la **caída de un broker**, observar **ISR**, particiones *under-replicated* y **recuperación** automática sin intervención manual (patrón DR operativo básico).

## Prerrequisitos

- Cluster de 3 brokers del Lab 5 en `Ready`.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="ha-lab09"
```

## Paso 1 — Topic con replicación 3

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 6 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC"
```

Cada línea debe mostrar `Replicas: 3` e `Isr: 3` (tres IDs de broker).

Publicar datos de prueba:

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c '
for i in $(seq 1 30); do echo "ha-$i"; done | \
  kafka-console-producer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic ha-lab09
'
```

## Paso 2 — Estado sano (línea base)

```bash
kubectl -n kafka get pods -l app=kafka -o wide
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC" | grep -E 'Partition|Leader|Isr'
```

## Paso 3 — Simular fallo de broker

Eliminar el pod del broker 1 (Kubernetes lo recrea; el volumen persiste):

```bash
kubectl -n kafka delete pod kafka-1
```

Inmediatamente:

```bash
kubectl -n kafka get pods -l app=kafka -w
```

En otra terminal, mientras `kafka-1` no está Ready:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC"
```

Observaciones esperadas:

- Particiones cuyo **líder** era el broker 1: nuevo líder elegido entre los supervivientes.
- **ISR** reducido en algunas particiones (`Isr: 2` en lugar de `3`).
- Posible mención *under-replicated* en herramientas de monitorización.

## Paso 4 — El cluster sigue sirviendo tráfico

Con un broker caído:

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c '
echo "durante-fallo" | kafka-console-producer \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 --topic ha-lab09
'

kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic ha-lab09 --from-beginning --timeout-ms 5000 | tail -5
'
```

## Paso 5 — Recuperación

Cuando `kafka-1` vuelva a `1/1 Ready`:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC" | grep Isr
```

El ISR debería volver a **3 réplicas** en todas las particiones tras la sincronización.

Comprobar quorum KRaft:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-metadata-quorum \
  --bootstrap-server "$BS" describe --status
```

## Comprobaciones

- [ ] Antes del fallo, todas las particiones tienen ISR completo.
- [ ] Durante el fallo, el topic sigue aceptando produce/consume.
- [ ] Tras la recuperación del pod, ISR vuelve a 3.
- [ ] Se ha usado `kubectl delete pod` (capa K8s) y `kafka-topics --describe` (capa Kafka).

## Retos opcionales

1. Repetir con `kafka-0` y comparar tiempos de recuperación.
2. `kubectl -n kafka describe pod kafka-1` y revisar Events durante el reinicio.
3. Leer logs: `kubectl -n kafka logs kafka-2 --tail=30` buscando *leader election*.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| Produce falla durante fallo | Esperar elección de líder; reintentar en 30 s. |
| ISR no se recupera | `kubectl -n kafka logs kafka-1`; revisar disco/PVC. |
| Pod Pending tras delete | Recursos del cluster kind insuficientes. |

## Limpieza

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --delete --topic ha-lab09
```

---

[← Volver: Capítulo 4](../fundamentos/04-replicacion-isr.md) · [Siguiente capítulo: 5 — KRaft →](../fundamentos/05-kraft.md)
