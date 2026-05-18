# LAB 9 — Replicación y tolerancia a fallos

> **Entorno.** Pod **`kafka-cli`** en `app-a` para CLIs; `kubectl` en namespace `kafka` para simular fallos de broker. [Entorno de práctica: pod kafka-cli](../docs/entorno-practica-kafka-cli.md)

## Objetivo

Comprender alta disponibilidad: réplicas, ISR y conmutación de líder.

## Ejecución (guía)

- Simular caída de broker (en entorno multi-broker).
- Analizar ISR y elección de líder.

## Observación

- Cambio de líder.
- Continuidad del sistema bajo fallo.

## Qué se aprende

Resiliencia: replicación, ISR, failover operativo.

## Material

*(Pendiente de la capa Kafka; depende del entorno multi-broker.)*
