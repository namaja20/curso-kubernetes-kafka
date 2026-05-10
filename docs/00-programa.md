# Programa — Kubernetes + Kafka (Confluent)

**Metodología y entorno:** [01-metodologia.md](01-metodologia.md) (teoría breve → demo preconstruida → lab constructivo; formador sin código en vivo) · [02-codespace.md](02-codespace.md) (todo en GitHub Codespaces).

## Objetivos generales

- Comprender la arquitectura de Kafka (incluido **KRaft**).
- Operar un cluster **Kafka (Confluent)** y gestionar topics, particiones y consumidores.
- Diagnosticar lag, replicación y consumo.
- Usar **Schema Registry**, **Kafka Connect** e introducción a **ksqlDB**.
- Entender despliegue y gestión con **operadores** (visión **Confluent for Kubernetes / CFK**).
- Relacionar la operación de Kafka con **Kubernetes** (pods, logs, reinicios).

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

## Resultados esperados (alumnado)

Ver README raíz y cierre de cada LAB.
