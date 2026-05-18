# LAB 6 — Particiones y distribución

> **Entorno.** Cluster y pod **`kafka-cli`** del [Lab 5](../lab-05-flujo-basico/README.md). [Entorno de práctica: pod kafka-cli](../docs/entorno-practica-kafka-cli.md) · [Cheatsheet](../docs/cheatsheet-comandos-kafka.md)

> **Llegada desde el [Capítulo 2 — Brokers, topics y particiones](../fundamentos/02-brokers-topics-particiones.md).** Al terminar, vuelve a ese capítulo.

[← Volver: Capítulo 2](../fundamentos/02-brokers-topics-particiones.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Observar cómo Kafka reparte mensajes entre **particiones** y cómo la **clave** fija el destino (mismo orden por clave).

## Prerrequisitos

- [Lab 5](../lab-05-flujo-basico/README.md) completado (`kafka-0..2` Ready, `kafka-cli` en `app-a`).
- Pod `kafka-cli` accesible.

Variables de trabajo (dentro del pod o en cada `exec`):

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="pedidos-lab06"
```

## Paso 1 — Topic con varias particiones

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" \
  --create --topic "$TOPIC" --partitions 6 --replication-factor 3

kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server "$BS" --describe --topic "$TOPIC"
```

Anotar en qué **broker** vive el líder de cada partición (`Leader: 0`, `1` o `2`).

## Paso 2 — Mensajes sin clave (reparto round-robin)

En el pod `kafka-cli`:

```bash
kubectl -n app-a exec -it deploy/kafka-cli -- bash
```

Dentro del pod:

```bash
BS="kafka.kafka.svc.cluster.local:9092"
TOPIC="pedidos-lab06"

for i in $(seq 1 12); do echo "sin-clave-$i"; done | \
  kafka-console-producer --bootstrap-server "$BS" --topic "$TOPIC"
```

Comprobar en qué particiones hay datos:

```bash
kafka-get-offsets --bootstrap-server "$BS" --topic "$TOPIC"
```

Varias particiones deberían tener offset > 0 (reparto aproximado entre particiones).

## Paso 3 — Mensajes con clave (misma clave → misma partición)

Formato `clave:valor` con propiedades de parseo:

```bash
cat <<'EOF' | kafka-console-producer --bootstrap-server "$BS" --topic "$TOPIC" \
  --property parse.key=true --property key.separator=:
cliente-A:pedido-1
cliente-A:pedido-2
cliente-B:pedido-3
cliente-A:pedido-4
cliente-B:pedido-5
EOF
```

Vuelve a ejecutar `kafka-get-offsets` y observa que los mensajes de `cliente-A` comparten partición (no necesariamente la de `cliente-B`).

## Paso 4 — Orden dentro de una partición

Consumir solo una partición concreta (ejemplo partición `0`):

```bash
kafka-console-consumer --bootstrap-server "$BS" --topic "$TOPIC" \
  --partition 0 --from-beginning --timeout-ms 5000
```

El orden de lectura es el orden de escritura **en esa partición**. Entre particiones no hay orden global.

## Comprobaciones

- [ ] El topic tiene 6 particiones con RF 3 e ISR completo.
- [ ] Sin clave, varias particiones reciben mensajes.
- [ ] Con clave, mensajes con la misma clave van a la misma partición.
- [ ] La lectura por `--partition N` respeta el orden local.

## Retos opcionales

1. Aumentar particiones: `kafka-topics --alter --topic "$TOPIC" --partitions 8` y repetir producción con clave `cliente-A`: el hash puede cambiar de partición (limitación al redimensionar).
2. `kafka-log-dirs --bootstrap-server "$BS" --describe --topic-list "$TOPIC"` y comparar con `kubectl -n kafka exec kafka-0 -- ls /var/lib/kafka/data | grep pedidos`.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| Topic ya existe | Usar otro nombre o `--delete` (solo en lab). |
| `LEADER_NOT_AVAILABLE` | Esperar brokers Ready. |
| Todas las claves en una partición | Normal si pocas claves distintas; probar más claves. |

## Limpieza

```bash
kafka-topics --bootstrap-server "$BS" --delete --topic pedidos-lab06
```

---

[← Volver: Capítulo 2](../fundamentos/02-brokers-topics-particiones.md) · [Siguiente capítulo: 3 — Consumer groups →](../fundamentos/03-consumer-groups-offsets.md)
