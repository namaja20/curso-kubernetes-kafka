# Bloque 2 — Fundamentos de Kafka y Confluent

[← Anterior: Modelo de objetos y pods](../../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) · [Siguiente: Event streaming y modelo Kafka →](01-event-streaming.md)

---

Este bloque desarrolla el modelo conceptual de Apache Kafka y los componentes del ecosistema Confluent. Empieza por la idea más básica —el log distribuido— y avanza hasta los componentes de mayor nivel (Schema Registry, Connect, ksqlDB) y al modo en que se opera todo ello en Kubernetes a través de un operador.

## Entorno de práctica (pod `kafka-cli`)

Las demos operativas del bloque (topics, productores/consumidores de consola, consumer groups, lag, etc.) se ejecutan desde un **pod dedicado `kafka-cli`** en el namespace de inquilino `app-a`, usando la imagen **`confluentinc/cp-kafka`** solo como contenedor de herramientas — **no como broker**. El cluster Kafka (brokers en el namespace `kafka`) se despliega una vez en el Lab 5 y se reutiliza en los Labs 6–12.

➡️ [Entorno de práctica: pod kafka-cli](../docs/entorno-practica-kafka-cli.md) · [Cheatsheet de comandos](../docs/cheatsheet-comandos-kafka.md)

## Laboratorios del bloque

Ruta práctica del programa, enlazada también al final de cada capítulo:

| # | Tema | Enlace |
|--:|------|--------|
| 5 | Flujo básico de datos | [`lab-05-flujo-basico`](../lab-05-flujo-basico/README.md) |
| 6 | Particiones y distribución | [`lab-06-particiones`](../lab-06-particiones/README.md) |
| 7 | Consumer groups | [`lab-07-consumer-groups`](../lab-07-consumer-groups/README.md) |
| 8 | Offsets y control de consumo | [`lab-08-offsets`](../lab-08-offsets/README.md) |
| 9 | Replicación y tolerancia a fallos | [`lab-09-replicacion-fallos`](../lab-09-replicacion-fallos/README.md) |
| 10 | Diagnóstico de lag | [`lab-10-diagnostico-lag`](../lab-10-diagnostico-lag/README.md) |
| 11 | Operación de topics | [`lab-11-operacion-topics`](../lab-11-operacion-topics/README.md) |
| 12 | Integración con Kafka Connect | [`lab-12-kafka-connect`](../lab-12-kafka-connect/README.md) |

## Capítulos

| # | Título | Archivo |
|--:|--------|---------|
| 1 | Event streaming y modelo Kafka | [`01-event-streaming.md`](01-event-streaming.md) |
| 2 | Brokers, topics y particiones | [`02-brokers-topics-particiones.md`](02-brokers-topics-particiones.md) |
| 3 | Consumer groups y offsets | [`03-consumer-groups-offsets.md`](03-consumer-groups-offsets.md) |
| 4 | Replicación, ISR y tolerancia a fallos | [`04-replicacion-isr.md`](04-replicacion-isr.md) |
| 5 | KRaft: controlador y quorum de metadatos | [`05-kraft.md`](05-kraft.md) |
| 6 | Schema Registry | [`06-schema-registry.md`](06-schema-registry.md) |
| 7 | Kafka Connect | [`07-kafka-connect.md`](07-kafka-connect.md) |
| 8 | Introducción a ksqlDB | [`08-ksqldb.md`](08-ksqldb.md) |
| 9 | Operadores y CFK (Confluent for Kubernetes) | [`09-operadores-cfk.md`](09-operadores-cfk.md) |

---

[← Anterior: Modelo de objetos y pods](../../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) · [Siguiente: Event streaming y modelo Kafka →](01-event-streaming.md)
