# Kubernetes y Kafka (Confluent)

Formación orientada a **comprender, operar y diagnosticar** una plataforma de **Apache Kafka** en modo **Confluent**, desplegada y operada en entornos **Kubernetes**, con énfasis en el comportamiento real del sistema y la resolución de incidencias.

**Duración:** 32 horas.

---

## Objetivos

Al finalizar el curso serás capaz de:

- Explicar la arquitectura de Kafka, incluido el modo **KRaft** (sin ZooKeeper).
- Operar un clúster Kafka en entorno Confluent: topics, particiones, productores y consumidores.
- Diagnosticar problemas habituales: lag, replicación, consumo y disponibilidad.
- Utilizar componentes Confluent relevantes: **Schema Registry**, **Kafka Connect** y una introducción a **ksqlDB**.
- Relacionar el despliegue de Kafka con **Kubernetes** y con el papel de los **operadores** (visión **Confluent for Kubernetes / CFK**).
- Aplicar comandos y flujos de trabajo en **kubectl** para inspeccionar cargas de trabajo, configuración y registros.

---

## Contenidos

El programa se organiza en **tres bloques**:

1. **Kubernetes (12 h)** — Fundamentos de contenedores y orquestación; despliegue y exposición de aplicaciones; escalado y actualizaciones; diagnóstico; ConfigMaps y Secrets. Cuatro laboratorios prácticos.
2. **Kafka y Confluent (18 h)** — Modelo de event streaming; brokers, topics, particiones; grupos de consumo y offsets; replicación y tolerancia a fallos; lag y operación de topics; **Kafka Connect**. Ocho laboratorios prácticos.
3. **Integración (2 h)** — Kafka en Kubernetes y un recorrido extremo a extremo (productor → topic → consumidor). Dos laboratorios prácticos.

El detalle por temas, laboratorios y rutas en el repositorio está en **[Programa del curso](docs/00-programa.md)**.

---

## Metodología (cómo se imparte el curso)

- **Base conceptual:** antes de cada práctica se introduce el vocabulario y las ideas necesarias, de forma breve y enlazada al ejercicio.
- **Ejemplos en el repositorio:** dispondrás de manifiestos, scripts y aplicaciones de referencia ya preparados para ilustrar el comportamiento del sistema y servir de base común durante las explicaciones.
- **Laboratorios guiados:** el grueso del aprendizaje es práctico. Cada laboratorio incluye objetivos claros, pasos ordenados, comprobaciones de que el resultado es el esperado y, en muchos casos, pequeños retos para consolidar lo aprendido.
- **Trabajo en entorno unificado:** las prácticas se realizan en **GitHub Codespaces** a partir de **tu fork** del repositorio, con las herramientas de línea de comandos y el acceso al entorno de clúster que se indique. No es necesario instalar software en tu equipo personal para seguir el curso.

---

## Estructura del repositorio

| Ruta | Descripción |
|------|-------------|
| `docs/` | Programa detallado del curso |
| `bloque-1-kubernetes/` | Fundamentos y laboratorios 1–4 |
| `bloque-2-kafka-confluent/` | Fundamentos y laboratorios 5–12 |
| `bloque-3-integracion/` | Laboratorios 13–14 |
| `entorno-local/` | Scripts y definiciones de apoyo para el entorno de prácticas (según el diseño del laboratorio) |

En cada carpeta de laboratorio encontrarás la descripción del ejercicio y los archivos de guion paso a paso (típicamente, guías en Markdown dentro de la misma carpeta).

---

## Cómo empezar

1. Haz un **fork** de este repositorio en tu cuenta de GitHub.
2. En tu fork, abre un **Codespace**: botón *Code* → pestaña *Codespaces* → *Create codespace on main* (o la rama indicada).
3. Ejecuta el **[Lab 0 — Preparar el entorno (kind + kubectl)](bloque-1-kubernetes/lab-00-entorno/README.md)** para dejar el cluster local listo. **Una sola vez por Codespace.**
4. Sigue el orden de bloques y laboratorios del [programa](docs/00-programa.md).

---

## ▶ Recorrido del contenido

El material teórico está pensado para leerse como un libro, en orden, con navegación `← anterior` / `siguiente →` en cada documento. Al final de cada capítulo hay una **ficha de laboratorio** (descripción, objetivos y encaje con el tema) que enlaza con el laboratorio del programa correspondiente; en los índices de cada bloque está la lista completa de labs.

**[Comenzar: Bloque 1 — Fundamentos de Kubernetes →](bloque-1-kubernetes/fundamentos/README.md)**
