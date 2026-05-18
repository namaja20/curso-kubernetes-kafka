# LAB 13 — Operación de Kafka en Kubernetes

> **Entorno.** Cluster Kafka del [Lab 5](../../bloque-2-kafka-confluent/lab-05-flujo-basico/README.md) en namespace `kafka` + pod **`kafka-cli`** en `app-a`.

> **Llegada desde el [Capítulo 1 — Kafka en Kubernetes](../fundamentos/01-kafka-en-k8s.md).**

[← Volver: Capítulo 1 — Kafka en Kubernetes](../fundamentos/01-kafka-en-k8s.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Practicar el diagnóstico **a dos capas**: Kubernetes (pods, logs, eventos, reinicios) y Kafka (ISR, describe, quorum KRaft).

## Prerrequisitos

- Lab 5 completado.
- Topic de prueba existente (p. ej. `eventos`) o crear uno nuevo.

```bash
BS="kafka.kafka.svc.cluster.local:9092"
```

## Paso 1 — Inventario plano Kubernetes

```bash
kubectl -n kafka get sts,svc,pvc,pods -o wide
kubectl -n app-a get deploy,pods
```

Anotar:

- Nombre de cada broker (`kafka-0` …).
- PVC asociado a cada pod (`data-kafka-N`).
- Service `kafka` (bootstrap) y `kafka-headless` (DNS por pod).

## Paso 2 — Inspeccionar un broker

```bash
kubectl -n kafka describe pod kafka-0
kubectl -n kafka logs kafka-0 --tail=40
```

Revisar sección **Events** del describe y buscar en logs referencias a *leader*, *partition* o *KRaft*.

## Paso 3 — Plano Kafka (mismo momento)

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe

kubectl -n app-a exec deploy/kafka-cli -- kafka-metadata-quorum \
  --bootstrap-server "$BS" describe --status
```

Correlacionar: un pod `Running` en K8s puede coexistir con particiones *under-replicated* en Kafka.

## Paso 4 — Reinicio controlado de un broker

```bash
kubectl -n kafka delete pod kafka-2
kubectl -n kafka get pods -w
```

Mientras reinicia, en otra terminal:

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe | grep -E 'Topic:|Leader|Isr'
```

Probar produce/consume desde `kafka-cli`:

```bash
kubectl -n app-a exec deploy/kafka-cli -- bash -c \
  'echo test-k8s | kafka-console-producer --bootstrap-server '"$BS"' --topic eventos'
```

## Paso 5 — Eventos del namespace

```bash
kubectl -n kafka get events --sort-by=.lastTimestamp | tail -15
```

## Comprobaciones

- [ ] Se identifican STS, headless, bootstrap y PVCs.
- [ ] Tras `delete pod`, el StatefulSet recrea el broker con el mismo nombre.
- [ ] `kafka-topics --describe` vuelve a ISR completo tras la recuperación.
- [ ] Se han usado herramientas de **ambas** capas.

## Retos opcionales

1. `kubectl -n kafka top pods` (si metrics-server disponible) durante produce de carga.
2. Comparar `kubectl -n kafka exec kafka-0 -- df -h` con `kafka-log-dirs --describe`.

## Troubleshooting

| Síntoma | Capa | Acción |
|---------|------|--------|
| Pod Pending | K8s | `describe pod`; PVC o CPU. |
| ISR incompleto persistente | Kafka | Logs del broker; tiempo de sync. |
| No resuelve DNS headless | K8s | `kubectl -n kafka get svc kafka-headless`. |

## Limpieza

No requiere borrado; el cluster se conserva para el Lab 14.

---

[← Volver: Capítulo 1](../fundamentos/01-kafka-en-k8s.md) · [Siguiente capítulo: 2 — Pipeline extremo a extremo →](../fundamentos/02-pipeline-extremo-a-extremo.md)
