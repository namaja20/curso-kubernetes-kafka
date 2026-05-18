# Cheatsheet — Comandos Kafka (operación contra cluster)

Referencia rápida para operar un cluster Kafka desde el entorno del curso. No sustituye la documentación oficial; recoge los comandos usados en los laboratorios y en los capítulos de fundamentos.

## Dónde se ejecutan los comandos

Todos los binarios `kafka-*` vienen en la imagen **`confluentinc/cp-kafka`**. En clase **no se usan en los pods broker** para las demos habituales, sino en el pod con rol **`kafka-cli`** (Deployment en `app-a`): misma imagen, pero el contenedor solo hace `sleep infinity` y sirve de shell para `kubectl exec`.

| Rol | Recurso | Namespace |
|-----|---------|-----------|
| Brokers | StatefulSet `kafka` | `kafka` |
| **CLI (demos)** | Deployment **`kafka-cli`** | `app-a` |

Detalle del patrón: [Entorno de práctica: pod kafka-cli](entorno-practica-kafka-cli.md). Despliegue inicial: [Lab 5](../lab-05-flujo-basico/README.md).

## Cómo ejecutar los comandos

Siempre desde el pod **`kafka-cli`** (salvo que el enunciado indique inspección en un broker concreto):

```bash
# Shell interactiva en el pod cliente (recomendado)
kubectl -n app-a exec -it deploy/kafka-cli -- bash

# Un solo comando sin entrar al pod
kubectl -n app-a exec deploy/kafka-cli -- <comando>
```

**Bootstrap** (punto de entrada del cluster en este curso):

```bash
BS="kafka.kafka.svc.cluster.local:9092"
```

Todas las órdenes siguientes asumen `--bootstrap-server "$BS"` salvo que se indique lo contrario.

Propiedades opcionales (archivo montado en el pod cliente del Lab 5):

```bash
# /etc/kafka-client/client.properties
# bootstrap.servers=kafka.kafka.svc.cluster.local:9092
# security.protocol=PLAINTEXT
```

---

## Resumen por área operativa

| Herramienta | Uso principal (operador) |
|-------------|--------------------------|
| `kafka-topics` | Crear, listar, describir y alterar topics |
| `kafka-console-producer` / `kafka-console-consumer` | Pruebas rápidas de flujo (no producción) |
| `kafka-consumer-groups` | Grupos, offsets, **lag** |
| `kafka-configs` | Retención, políticas, configs dinámicas |
| `kafka-metadata-quorum` | Estado del quorum **KRaft** |
| `kafka-reassign-partitions` | Mover réplicas entre brokers |
| `kafka-broker-api-versions` | Comprobar conectividad y APIs |
| `kafka-log-dirs` | Uso de disco por partición en brokers |
| `kafka-acls` | Permisos (clusters con seguridad) |

---

## Topics — `kafka-topics`

| Acción | Comando |
|--------|---------|
| Comprobar conectividad | `kafka-broker-api-versions --bootstrap-server "$BS"` |
| Listar topics | `kafka-topics --bootstrap-server "$BS" --list` |
| Describir un topic (líder, ISR, réplicas) | `kafka-topics --bootstrap-server "$BS" --describe --topic <nombre>` |
| Describir todos | `kafka-topics --bootstrap-server "$BS" --describe` |
| Crear topic | `kafka-topics --bootstrap-server "$BS" --create --topic <nombre> --partitions <N> --replication-factor <RF>` |
| Aumentar particiones | `kafka-topics --bootstrap-server "$BS" --alter --topic <nombre> --partitions <N>` |
| Borrar topic | `kafka-topics --bootstrap-server "$BS" --delete --topic <nombre>` |

Ejemplo (Lab 5):

```bash
kafka-topics --bootstrap-server "$BS" \
  --create --topic eventos --partitions 3 --replication-factor 3

kafka-topics --bootstrap-server "$BS" --describe --topic eventos
```

En la salida de `--describe`, por cada partición: **Leader**, **Replicas**, **Isr** (réplicas en sync). Base del diagnóstico de replicación (Labs 9–10).

---

## Prueba de flujo — productor y consumidor de consola

Herramientas para **verificar** que el cluster responde; las aplicaciones reales usan clientes (Java, Python, etc.), no estas CLIs.

| Acción | Comando |
|--------|---------|
| Escribir mensajes (stdin) | `kafka-console-producer --bootstrap-server "$BS" --topic <nombre>` |
| Escribir con clave | `kafka-console-producer --bootstrap-server "$BS" --topic <nombre> --property parse.key=true --property key.separator=:` |
| Leer desde el inicio | `kafka-console-consumer --bootstrap-server "$BS" --topic <nombre> --from-beginning` |
| Leer solo mensajes nuevos | `kafka-console-consumer --bootstrap-server "$BS" --topic <nombre>` |
| Leer con grupo (prueba CG) | `kafka-console-consumer --bootstrap-server "$BS" --topic <nombre> --group <grupo> --from-beginning` |
| Salir del consumer con timeout | añadir `--timeout-ms 5000` |

---

## Consumer groups y lag — `kafka-consumer-groups`

Comando central del operador (equivalente operativo a `kubectl get pods` en Kubernetes).

| Acción | Comando |
|--------|---------|
| Listar grupos | `kafka-consumer-groups --bootstrap-server "$BS" --list` |
| Detalle, offsets y **lag** | `kafka-consumer-groups --bootstrap-server "$BS" --describe --group <grupo>` |
| Detalle de todos los grupos | `kafka-consumer-groups --bootstrap-server "$BS" --describe --all-groups` |
| Estado del grupo | `kafka-consumer-groups --bootstrap-server "$BS" --describe --group <grupo> --state` |
| Reset offsets → inicio del log | `kafka-consumer-groups --bootstrap-server "$BS" --group <grupo> --topic <nombre> --reset-offsets --to-earliest --execute` |
| Reset offsets → fin del log | `kafka-consumer-groups --bootstrap-server "$BS" --group <grupo> --topic <nombre> --reset-offsets --to-latest --execute` |
| Reset a timestamp | `kafka-consumer-groups --bootstrap-server "$BS" --group <grupo> --topic <nombre> --reset-offsets --to-datetime 2026-05-11T10:00:00.000 --execute` |
| Borrar offsets de un grupo | `kafka-consumer-groups --bootstrap-server "$BS" --group <grupo> --delete` |

Columnas habituales en `--describe`: **CURRENT-OFFSET**, **LOG-END-OFFSET**, **LAG**. Lag creciente de forma sostenida indica que el consumo no alcanza la producción.

---

## Configuración dinámica — `kafka-configs`

| Acción | Comando |
|--------|---------|
| Ver config de un topic | `kafka-configs --bootstrap-server "$BS" --entity-type topics --entity-name <nombre> --describe` |
| Alterar retención por tiempo | `kafka-configs --bootstrap-server "$BS" --entity-type topics --entity-name <nombre> --alter --add-config retention.ms=86400000` |
| Alterar retención por tamaño | `kafka-configs --bootstrap-server "$BS" --entity-type topics --entity-name <nombre> --alter --add-config retention.bytes=1073741824` |
| Borrar una config | `kafka-configs --bootstrap-server "$BS" --entity-type topics --entity-name <nombre> --alter --delete-config retention.ms` |
| Config de broker | `kafka-configs --bootstrap-server "$BS" --entity-type brokers --entity-name <broker-id> --describe` |

---

## KRaft — `kafka-metadata-quorum`

Sustituye las consultas antiguas a ZooKeeper en clusters modernos.

| Acción | Comando |
|--------|---------|
| Estado del quorum | `kafka-metadata-quorum --bootstrap-server "$BS" describe --status` |
| Réplicas del log de metadatos | `kafka-metadata-quorum --bootstrap-server "$BS" describe --replication` |

---

## Reasignación de particiones — `kafka-reassign-partitions`

Operación de mantenimiento (escalar brokers, equilibrar carga, DR). Flujo en tres pasos:

```bash
# 1) Generar plan JSON (topics-to-move.json con topics y particiones deseadas)
kafka-reassign-partitions --bootstrap-server "$BS" \
  --generate --topics-to-move-json-file topics-to-move.json \
  --broker-list 0,1,2 --bootstrap-server "$BS"

# 2) Ejecutar el plan generado (reassignment.json)
kafka-reassign-partitions --bootstrap-server "$BS" \
  --execute --reassignment-json-file reassignment.json

# 3) Verificar progreso
kafka-reassign-partitions --bootstrap-server "$BS" \
  --verify --reassignment-json-file reassignment.json
```

---

## Disco y brokers — `kafka-log-dirs`

| Acción | Comando |
|--------|---------|
| Uso de disco por directorio de log | `kafka-log-dirs --bootstrap-server "$BS" --describe --topic-list <nombre>` |
| Todos los topics en un broker | `kafka-log-dirs --bootstrap-server "$BS" --broker-list <id> --describe` |

Útil junto con `kubectl exec` en pods `kafka-N` para inspeccionar `/var/lib/kafka/data` (Lab 5, reto opcional).

---

## Seguridad — `kafka-acls` (si el cluster usa SASL/SSL)

| Acción | Comando |
|--------|---------|
| Listar ACLs | `kafka-acls --bootstrap-server "$BS" --list` |
| Conceder produce en topic | `kafka-acls --bootstrap-server "$BS" --add --allow-principal User:<app> --operation Write --topic <nombre>` |
| Conceder consume en topic | `kafka-acls --bootstrap-server "$BS" --add --allow-principal User:<app> --operation Read --topic <nombre> --group <grupo>` |

En el Lab 5 el cluster usa `PLAINTEXT`; estos comandos aplican cuando se active seguridad en entornos reales.

---

## Relación con las cuatro APIs del ecosistema

La diapositiva de **Componentes / APIs** del material describe *tipos de cliente*, no herramientas CLI:

| API (concepto) | Herramienta operativa en este curso |
|----------------|-------------------------------------|
| **Producer** | Apps + `kafka-console-producer` (solo prueba) |
| **Consumer** | Apps + `kafka-console-consumer` (solo prueba) |
| **Consumer groups** | `kafka-consumer-groups` |
| **Stream Processor** | Kafka Streams / ksqlDB (Lab 12, cap. ksqlDB) |
| **Connector** | Kafka Connect REST API + CRs (Lab 12) |

El operador de plataforma combina **kubectl** (pods, logs, reinicios) con **`kafka-topics --describe`** y **`kafka-consumer-groups --describe`** para el plano Kafka.

---

## Comandos Kubernetes frecuentes (mismo escenario Lab 5)

| Acción | Comando |
|--------|---------|
| Estado de brokers | `kubectl -n kafka get pods -l app=kafka` |
| Logs de un broker | `kubectl -n kafka logs kafka-0 -f` |
| Reiniciar un broker (DR simulado) | `kubectl -n kafka delete pod kafka-1` |
| PVCs del cluster | `kubectl -n kafka get pvc` |
| Cliente del inquilino | `kubectl -n app-a get deploy kafka-cli` |

---

## Parámetros que suelen aparecer en operación

| Parámetro / flag | Significado breve |
|------------------|-------------------|
| `--bootstrap-server` | Brokers de arranque para descubrir metadatos |
| `--partitions` | Paralelismo del topic |
| `--replication-factor` | Copias por partición (tolerancia a fallos) |
| `--from-beginning` | Consumer lee desde el offset más antiguo disponible |
| `--group` | Consumer group (reparto y offsets) |
| `retention.ms` | Tiempo que se conservan los mensajes |
| `min.insync.replicas` | Mínimo de réplicas en ISR para aceptar writes “seguros” |
| `unclean.leader.election.enable` | Si `false`, no se elige líder fuera del ISR (más consistencia) |

---

[← Entorno: pod kafka-cli](entorno-practica-kafka-cli.md) · [Índice del bloque](../fundamentos/README.md) · [Lab 5 — Flujo básico](../lab-05-flujo-basico/README.md)
