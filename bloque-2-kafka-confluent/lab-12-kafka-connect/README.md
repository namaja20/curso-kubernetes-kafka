# LAB 12 — Integración con Kafka Connect (JDBC → Kafka)

> **Entorno.** Cluster Kafka del Lab 5 + **Kafka Connect** y **PostgreSQL** en `lab12`. Las CLIs `kafka-*` siguen en pod **`kafka-cli`** (`app-a`). [Entorno de práctica](../docs/entorno-practica-kafka-cli.md)

> **Llegada desde el [Capítulo 7 — Kafka Connect](../fundamentos/07-kafka-connect.md).**

[← Volver: Capítulo 7 — Kafka Connect](../fundamentos/07-kafka-connect.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Desplegar un **worker de Kafka Connect**, registrar un conector **JDBC Source** y verificar que filas de PostgreSQL aparecen en un topic de Kafka. Operación vía **API REST** (no `kafka-topics`).

## Prerrequisitos

- [Lab 5](../lab-05-flujo-basico/README.md) operativo.
- ~1 GiB RAM adicional para Connect + PostgreSQL.

## Organización de los manifiestos

```
lab-12-kafka-connect/
└── manifest/
    ├── 00-namespace.yaml
    └── apps/
        ├── postgres/
        │   ├── configmap-init.yaml
        │   └── deployment.yaml      # Service + Deployment
        ├── connect/
        │   ├── deployment.yaml      # Connect worker + Service :8083
        │   └── connector-jdbc-source.json
```

Connect usa imagen **`confluentinc/cp-kafka-connect`** (incluye conector JDBC). Kafka sigue en namespace `kafka`.

## Paso 1 — Desplegar PostgreSQL y Connect

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-12-kafka-connect/manifest/00-namespace.yaml
kubectl apply -R -f bloque-2-kafka-confluent/lab-12-kafka-connect/manifest/apps/postgres/
kubectl apply -f bloque-2-kafka-confluent/lab-12-kafka-connect/manifest/apps/connect/deployment.yaml
```

Esperar Ready:

```bash
kubectl -n lab12 get pods
kubectl -n lab12 wait --for=condition=ready pod -l app=connect --timeout=180s
```

Comprobar API REST:

```bash
kubectl -n lab12 exec deploy/connect -- curl -s http://localhost:8083/connectors
```

Debe responder `[]` (sin conectores aún).

## Paso 2 — Registrar el conector JDBC Source

El manifiesto JSON de referencia:

[`manifest/apps/connect/connector-jdbc-source.json`](manifest/apps/connect/connector-jdbc-source.json)

Registro vía REST (el JSON se envía desde el host por stdin):

```bash
kubectl -n lab12 exec -i deploy/connect -- curl -s -X POST \
  -H "Content-Type: application/json" \
  -d @- http://localhost:8083/connectors \
  < bloque-2-kafka-confluent/lab-12-kafka-connect/manifest/apps/connect/connector-jdbc-source.json
```

Estado del conector:

```bash
kubectl -n lab12 exec deploy/connect -- \
  curl -s http://localhost:8083/connectors/jdbc-source-pedidos/status | head -c 500
```

## Paso 3 — Verificar topic en Kafka

El conector crea el topic `jdbc-pedidos` (prefijo `jdbc-` + nombre de tabla):

```bash
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 --list | grep jdbc

kubectl -n app-a exec deploy/kafka-cli -- kafka-console-consumer \
  --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic jdbc-pedidos --from-beginning --timeout-ms 10000
```

Deberían aparecer registros JSON con `id`, `cliente`, `importe`.

## Paso 4 — Incremental (nueva fila en BD)

Insertar fila en PostgreSQL:

```bash
kubectl -n lab12 exec deploy/postgres -- psql -U demo -d demo -c \
  "INSERT INTO pedidos (cliente, importe) VALUES ('Umbrella', 999.00);"
```

Tras unos segundos, volver a consumir o usar `--partition 0 --offset latest` para ver el mensaje nuevo.

## Comprobaciones

- [ ] Pod `connect` Ready y API `/connectors` responde.
- [ ] Conector `jdbc-source-pedidos` en estado `RUNNING`.
- [ ] Topic `jdbc-pedidos` con al menos 3 mensajes iniciales.
- [ ] Nueva fila en SQL se refleja en Kafka (modo incrementing).

## Retos opcionales

1. `GET http://localhost:8083/connectors/jdbc-source-pedidos/config` — revisar configuración efectiva.
2. Pausar/reanudar: `PUT .../connectors/jdbc-source-pedidos/pause` y `.../resume`.
3. Borrar conector: `DELETE .../connectors/jdbc-source-pedidos`.

## Troubleshooting

| Síntoma | Acción |
|---------|--------|
| Connect no arranca | `kubectl -n lab12 logs deploy/connect`; revisar topics internos `connect-*` en Kafka. |
| `Failed to connect to postgres` | Esperar pod postgres Ready; probar DNS `postgres.lab12.svc.cluster.local`. |
| Conector FAILED | `curl .../status`; revisar credenciales JDBC y que la tabla `pedidos` existe. |
| Topic vacío | Repetir POST del conector; revisar logs del task. |

## Limpieza

```bash
kubectl -n lab12 exec deploy/connect -- \
  curl -s -X DELETE http://localhost:8083/connectors/jdbc-source-pedidos
kubectl delete namespace lab12
```

El cluster Kafka en `kafka` se conserva.

---

[← Volver: Capítulo 7 — Kafka Connect](../fundamentos/07-kafka-connect.md) · [Siguiente capítulo: 8 — ksqlDB →](../fundamentos/08-ksqldb.md) · [Bloque 3 — Integración →](../../bloque-3-integracion/fundamentos/README.md)
