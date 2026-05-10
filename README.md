# Formación: Kubernetes + Kafka (Confluent)

Capacitación para **comprender, operar y diagnosticar** una plataforma Kafka (Confluent) en entornos modernos, incluyendo **Kubernetes**.

- **Duración total:** 32 h  
- **Metodología:** laboratorios prácticos guiados (objetivo → contexto → ejecución → observación → conclusiones)

## Estructura del repositorio

| Ruta | Contenido |
|------|-----------|
| `docs/` | Programa, referencias y material de apoyo |
| `bloque-1-kubernetes/` | Fundamentos K8s + LAB 1–4 |
| `bloque-2-kafka-confluent/` | Fundamentos Kafka/Confluent + LAB 5–12 |
| `bloque-3-integracion/` | LAB 13–14 (K8s + pipeline end-to-end) |
| `entorno-local/` | Alternativa reproducible en local (contenedores) cuando no haya laboratorio en navegador |

Cada laboratorio tiene su carpeta con un `README.md` que resume objetivo, ejecución prevista y aprendizajes.

## Entorno de prácticas

1. **Recomendado en aula:** entorno vía navegador (K8s + Kafka KRaft Confluent + herramientas), homogéneo y sin instalación en el puesto.
2. **Alternativa:** seguir `entorno-local/README.md` para levantar un entorno equivalente con contenedores desde este repo.

## Cómo usar este repo con Cursor

Abre la carpeta `curso-kubernetes-kafka` como **raíz del workspace** (`Archivo → Abrir carpeta`). Así el agente y las búsquedas se centran en el curso, no en otros proyectos.

## Estado del material

La estructura y los README por laboratorio están preparados; el contenido guiado paso a paso, manifiestos y scripts se irán completando por capas (K8s → Kafka → integración).
