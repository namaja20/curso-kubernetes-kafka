# Bloque 2 — Fundamentos de Kafka y Confluent

[← Anterior: Tema 5 — Modelo de objetos y pods](../../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) · [Siguiente: Tema 1 — Event streaming y modelo Kafka →](01-event-streaming.md)

---

Material de soporte para la parte teórica del bloque 2 (antes y entre los laboratorios 5–12). Cada tema está pensado para una explicación que **alterna teoría corta y demos**; los laboratorios trabajan estos mismos conceptos con las manos.

## Itinerario sugerido

| Orden | Tema | Archivo | Tiempo orientativo | Conexión con laboratorios |
|------:|------|---------|--------------------|---------------------------|
| 1 | Event streaming y modelo Kafka | [`01-event-streaming.md`](01-event-streaming.md) | 25–30 min | LAB 5 |
| 2 | Brokers, topics y particiones | [`02-brokers-topics-particiones.md`](02-brokers-topics-particiones.md) | 30–40 min | LAB 5, 6 |
| 3 | Consumer groups y offsets | [`03-consumer-groups-offsets.md`](03-consumer-groups-offsets.md) | 35–45 min | LAB 7, 8 |
| 4 | Replicación, ISR y tolerancia a fallos | [`04-replicacion-isr.md`](04-replicacion-isr.md) | 30–40 min | LAB 9 |
| 5 | KRaft: controlador y quorum de metadatos | [`05-kraft.md`](05-kraft.md) | 25–30 min | LAB 9, 13 |
| 6 | Schema Registry | [`06-schema-registry.md`](06-schema-registry.md) | 25–30 min | LAB 12 |
| 7 | Kafka Connect | [`07-kafka-connect.md`](07-kafka-connect.md) | 30–35 min | LAB 12 |
| 8 | Introducción a ksqlDB | [`08-ksqldb.md`](08-ksqldb.md) | 20–25 min | (contextual) |
| 9 | Operadores y CFK (Confluent for Kubernetes) | [`09-operadores-cfk.md`](09-operadores-cfk.md) | 25–30 min | LAB 13 |

## Cómo usar este material

- Los temas 1–4 son **troncales**: sin ellos los laboratorios pierden sentido.
- Los temas 5 (KRaft) y 9 (CFK) son **operativos**: importan al comprender la versión de Kafka que usaremos en Kubernetes.
- Los temas 6–8 son **ecosistema Confluent**: contextualizan funcionalidades que se ven en laboratorio (Schema Registry y Connect) o que se introducen sin práctica obligatoria (ksqlDB).
- La carga teórica está pensada para **no aplastar al grupo**: bloques de 25–45 min separados por laboratorios.

---

[← Anterior: Tema 5 — Modelo de objetos y pods](../../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) · [Siguiente: Tema 1 — Event streaming y modelo Kafka →](01-event-streaming.md)
