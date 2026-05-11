# Programa — Kubernetes + Kafka (Confluent)

## Objetivos generales

- Comprender la arquitectura de Kafka (incluido **KRaft**).
- Operar un cluster **Kafka (Confluent)** y gestionar topics, particiones y consumidores.
- Diagnosticar lag, replicación y consumo.
- Usar **Schema Registry**, **Kafka Connect** e introducción a **ksqlDB**.
- Entender despliegue y gestión con **operadores** (visión **Confluent for Kubernetes / CFK**).
- Relacionar la operación de Kafka con **Kubernetes** (pods, logs, reinicios).

## Calendario por sesiones (3 h)

Las **32 h** de contenido del programa encajan en **11 sesiones** de **3 h** de calendario cada una (≈2 h 45 min de tiempo efectivo si reservas **15 min de descanso** a media sesión). Los tiempos son orientativos: usa los *retos opcionales* de cada laboratorio como buffer si el grupo va justo.

| Sesión | Bloque | Contenido principal |
|:------:|--------|---------------------|
| 1 | B1 | Lab 0 + caps. 01–02 + Lab 1 |
| 2 | B1 | Cap. 03 + Lab 2 + cap. 04 (1ª parte) |
| 3 | B1 | Cap. 04 (2ª parte) + Lab 3 + cap. 05 (1ª parte) + buffer |
| 4 | B1 | Cap. 05 (2ª parte) + Lab 4 + cierre B1 |
| 5 | B2 | Caps. B2-01–02 + Lab 5 |
| 6 | B2 | Lab 6 + cap. B2-03 (1ª parte) + Lab 7 |
| 7 | B2 | Cap. B2-03 (2ª parte) + Lab 8 + cap. B2-04 + Lab 9 |
| 8 | B2 | Lab 10 + Lab 11 + cap. B2-05 |
| 9 | B2 | Caps. B2-06–07 + práctica SR + demo Connect |
| 10 | B2 | Lab 12 + cap. B2-08 + demo ksqlDB |
| 11 | B2–B3 | Cap. B2-09 + caps. B3-01–02 + Labs 13–14 |

### Sesión 1 — Arranque y primer despliegue (B1)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Bienvenida, objetivos, mecánica del curso (flow teoría ↔ laboratorio) |
| 0:15 | [Lab 0](../bloque-1-kubernetes/lab-00-entorno/README.md) — entorno (Codespaces + kind + kubectl) |
| 0:45 | [Cap. 01 — Contenedores vs VMs](../bloque-1-kubernetes/fundamentos/01-contenedores-vs-vms.md) |
| 1:10 | [Cap. 02 — Runtime y CRI](../bloque-1-kubernetes/fundamentos/02-runtime-y-cri.md) |
| 1:35 | *Descanso (15 min)* |
| 1:50 | [Lab 1](../bloque-1-kubernetes/lab-01-despliegue/README.md) — Deployment + Service |
| 2:45 | Cierre, dudas, vista a la sesión 2 |

### Sesión 2 — Orquestación y operación (B1)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Repaso breve |
| 0:10 | [Cap. 03 — Orquestación](../bloque-1-kubernetes/fundamentos/03-orquestacion.md) |
| 0:40 | [Lab 2](../bloque-1-kubernetes/lab-02-escalado-actualizaciones/README.md) — escalado, RollingUpdate, rollback |
| 1:40 | *Descanso (15 min)* |
| 1:55 | [Cap. 04 — Arquitectura de Kubernetes](../bloque-1-kubernetes/fundamentos/04-arquitectura-k8s.md) (1ª parte: control plane + nodos) |
| 2:35 | Discusión y vista al Lab 3 |

### Sesión 3 — Arquitectura y diagnóstico (B1)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Repaso breve |
| 0:10 | [Cap. 04](../bloque-1-kubernetes/fundamentos/04-arquitectura-k8s.md) (2ª parte: scheduling y ciclo de vida del pod) |
| 0:40 | [Lab 3](../bloque-1-kubernetes/lab-03-diagnostico/README.md) — diagnóstico (varios escenarios) |
| 1:40 | *Descanso (15 min)* |
| 1:55 | [Cap. 05 — Modelo de objetos y pods](../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) (1ª parte) |
| 2:30 | Buffer: exploración con `kubectl`, dudas |

### Sesión 4 — Configuración y cierre B1

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Repaso breve |
| 0:10 | [Cap. 05](../bloque-1-kubernetes/fundamentos/05-objetos-y-pods.md) (2ª parte: probes, namespaces) |
| 0:35 | [Lab 4](../bloque-1-kubernetes/lab-04-configuracion/README.md) — ConfigMaps y Secrets |
| 1:45 | *Descanso (15 min)* |
| 2:00 | Cierre B1: mapa mental de Kubernetes |
| 2:30 | Puente al Bloque 2 — por qué Kubernetes + Kafka |

### Sesión 5 — Idea de log y topología (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | [Event streaming y modelo Kafka](../bloque-2-kafka-confluent/fundamentos/01-event-streaming.md) |
| 0:45 | [Brokers, topics y particiones](../bloque-2-kafka-confluent/fundamentos/02-brokers-topics-particiones.md) |
| 1:30 | *Descanso (15 min)* |
| 1:45 | [Lab 5](../bloque-2-kafka-confluent/lab-05-flujo-basico/README.md) — flujo básico (topic, produce/consume) |
| 2:45 | Cierre y dudas |

### Sesión 6 — Particiones, claves y grupos (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Repaso breve |
| 0:10 | [Lab 6](../bloque-2-kafka-confluent/lab-06-particiones/README.md) — particiones y clave |
| 1:10 | [Consumer groups y offsets](../bloque-2-kafka-confluent/fundamentos/03-consumer-groups-offsets.md) (1ª parte) |
| 1:55 | *Descanso (15 min)* |
| 2:10 | [Lab 7](../bloque-2-kafka-confluent/lab-07-consumer-groups/README.md) — consumer groups y rebalanceos |

### Sesión 7 — Offsets y replicación (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | [Consumer groups y offsets](../bloque-2-kafka-confluent/fundamentos/03-consumer-groups-offsets.md) (2ª parte: commits, earliest/latest, reset) |
| 0:30 | [Lab 8](../bloque-2-kafka-confluent/lab-08-offsets/README.md) — offsets |
| 1:25 | *Descanso (15 min)* |
| 1:40 | [Replicación e ISR](../bloque-2-kafka-confluent/fundamentos/04-replicacion-isr.md) |
| 2:20 | [Lab 9](../bloque-2-kafka-confluent/lab-09-replicacion-fallos/README.md) — replicación e ISR, fallo de broker |

### Sesión 8 — Operación y KRaft (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | Repaso breve (replicación) |
| 0:10 | [Lab 10](../bloque-2-kafka-confluent/lab-10-diagnostico-lag/README.md) — diagnóstico de lag |
| 1:25 | *Descanso (15 min)* |
| 1:40 | [Lab 11](../bloque-2-kafka-confluent/lab-11-operacion-topics/README.md) — operación de topics (retención, etc.) |
| 2:20 | [KRaft](../bloque-2-kafka-confluent/fundamentos/05-kraft.md) |
| 2:50 | Cierre breve |

### Sesión 9 — Schema Registry y Kafka Connect (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | [Schema Registry](../bloque-2-kafka-confluent/fundamentos/06-schema-registry.md) |
| 0:45 | Práctica guiada sobre Schema Registry (topics de sesiones anteriores) |
| 1:40 | *Descanso (15 min)* |
| 1:55 | [Kafka Connect](../bloque-2-kafka-confluent/fundamentos/07-kafka-connect.md) |
| 2:40 | Demo en vivo Connect → vista al Lab 12 |

### Sesión 10 — Connect en práctica y ksqlDB (B2)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | [Lab 12](../bloque-2-kafka-confluent/lab-12-kafka-connect/README.md) — Kafka Connect (JDBC → Kafka) |
| 1:30 | *Descanso (15 min)* |
| 1:45 | [ksqlDB](../bloque-2-kafka-confluent/fundamentos/08-ksqldb.md) |
| 2:35 | Demo ksqlDB sobre topics del Lab 12 |

### Sesión 11 — CFK, integración y cierre (B2–B3)

| Tiempo | Actividad |
|--------|-----------|
| 0:00 | [Operadores y CFK](../bloque-2-kafka-confluent/fundamentos/09-operadores-cfk.md) |
| 0:40 | [Kafka en Kubernetes](../bloque-3-integracion/fundamentos/01-kafka-en-k8s.md) |
| 1:10 | [Pipeline extremo a extremo](../bloque-3-integracion/fundamentos/02-pipeline-extremo-a-extremo.md) |
| 1:30 | *Descanso (15 min)* |
| 1:45 | [Lab 13](../bloque-3-integracion/lab-13-kafka-en-kubernetes/README.md) — Kafka en K8s (pods, logs, reinicio) |
| 2:30 | [Lab 14](../bloque-3-integracion/lab-14-pipeline-completo/README.md) — pipeline Producer → Topic → Consumer |
| 2:55 | Cierre del curso y preguntas |

### Notas de impartición

- **Labs densos** (2, 3 y 12): acorta con los pasos opcionales del README si el grupo va justo de tiempo.
- **Buffer B1**: la sesión 3 incluye tiempo libre al final; la sesión 4 cierra con margen — úsalos si B1 se alarga.
- **Stack de integración** (Java, Python, JS, etc.): solo condiciona los ejemplos de productor/consumidor en los labs de Kafka; no altera este reparto horario.

## Bloque 1 — Kubernetes (12 h)

### Fundamentos iniciales (sin LAB de instalación)

- Contenedores vs VMs; rol de Docker en imagen/distribución.
- Ejecución en K8s: runtime (containerd/CRI), evolución respecto a Docker como runtime.
- Orquestación: problemas que resuelve K8s, múltiples instancias, HA, automatización.
- Arquitectura: control plane y nodos, scheduler, API server, kubelet, ciclo de vida del pod.
- Modelo de objetos: pods, introducción a objetos declarativos.

### Laboratorios

| LAB | Tema | Carpeta |
|-----|------|---------|
| 0 | Preparar el entorno (kind + kubectl) | `bloque-1-kubernetes/lab-00-entorno/` |
| 1 | Despliegue de aplicaciones (Deployment, Service) | `bloque-1-kubernetes/lab-01-despliegue/` |
| 2 | Escalado y actualizaciones (rollback) | `bloque-1-kubernetes/lab-02-escalado-actualizaciones/` |
| 3 | Diagnóstico (logs, eventos, pods en error) | `bloque-1-kubernetes/lab-03-diagnostico/` |
| 4 | ConfigMaps y Secrets | `bloque-1-kubernetes/lab-04-configuracion/` |

## Bloque 2 — Kafka + Confluent (18 h)

### Contenidos teóricos contextuales

- Kafka como plataforma de event streaming: brokers, topics, particiones, consumer groups, offsets.
- **KRaft:** controlador, quorum de metadatos, HA.
- Componentes Confluent: **Schema Registry**, **Kafka Connect**, introducción **ksqlDB**.
- Kafka en K8s: qué es un operator, introducción **CFK**, uso enterprise.

### Laboratorios

| LAB | Tema | Carpeta |
|-----|------|---------|
| 5 | Flujo básico (topic, produce/consume) | `bloque-2-kafka-confluent/lab-05-flujo-basico/` |
| 6 | Particiones y clave | `bloque-2-kafka-confluent/lab-06-particiones/` |
| 7 | Consumer groups y rebalanceos | `bloque-2-kafka-confluent/lab-07-consumer-groups/` |
| 8 | Offsets (earliest/latest, reset) | `bloque-2-kafka-confluent/lab-08-offsets/` |
| 9 | Replicación, ISR, fallo de broker | `bloque-2-kafka-confluent/lab-09-replicacion-fallos/` |
| 10 | Diagnóstico de lag | `bloque-2-kafka-confluent/lab-10-diagnostico-lag/` |
| 11 | Operación de topics (p. ej. retención) | `bloque-2-kafka-confluent/lab-11-operacion-topics/` |
| 12 | Kafka Connect (JDBC → Kafka) | `bloque-2-kafka-confluent/lab-12-kafka-connect/` |

## Bloque 3 — Escenarios e integración (2 h)

| LAB | Tema | Carpeta |
|-----|------|---------|
| 13 | Kafka en Kubernetes (pods, logs, reinicio) | `bloque-3-integracion/lab-13-kafka-en-kubernetes/` |
| 14 | Pipeline completo Producer → Topic → Consumer | `bloque-3-integracion/lab-14-pipeline-completo/` |

## Resultados esperados

Los resultados de aprendizaje del curso se recogen en el README principal del repositorio; cada laboratorio concreta los logros de su sesión.
