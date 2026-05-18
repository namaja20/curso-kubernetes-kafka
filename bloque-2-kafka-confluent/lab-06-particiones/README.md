# LAB 6 — Particiones y distribución

> **Entorno.** Reutiliza el cluster del [Lab 5](../lab-05-flujo-basico/README.md). Las demos se ejecutan desde el pod **`kafka-cli`** (`kubectl -n app-a exec deploy/kafka-cli -- …`). [Entorno de práctica: pod kafka-cli](../docs/entorno-practica-kafka-cli.md)

## Objetivo

Entender cómo Kafka escala y distribuye carga entre particiones.

## Ejecución (guía)

- Crear topic con múltiples particiones.
- Enviar mensajes con y sin clave.

## Observación

- Distribución entre particiones.
- Orden garantizado por partición.

## Qué se aprende

Escalabilidad y semántica de orden.

## Material

*(Pendiente de la capa Kafka.)*
