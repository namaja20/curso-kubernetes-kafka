# Formación: Kubernetes + Kafka (Confluent)

Capacitación para **comprender, operar y diagnosticar** una plataforma Kafka (Confluent) en entornos modernos, incluyendo **Kubernetes**.

- **Duración total:** 32 h  
- **Metodología:** ver **[Metodología didáctica](docs/01-metodologia.md)** — ciclo *teoría breve* → *demo con repo preconstruido* → *laboratorio constructivo*; **todo en [GitHub Codespaces](docs/02-codespace.md)**.

## Estructura del repositorio

| Ruta | Contenido |
|------|-----------|
| `docs/` | Programa (`00-programa.md`), metodología (`01-metodologia.md`), Codespace (`02-codespace.md`) |
| `bloque-1-kubernetes/` | Fundamentos K8s + LAB 1–4 |
| `bloque-2-kafka-confluent/` | Fundamentos Kafka/Confluent + LAB 5–12 |
| `bloque-3-integracion/` | LAB 13–14 (K8s + pipeline end-to-end) |
| `entorno-local/` | Scripts y definiciones invocables **desde el Codespace** (p. ej. cluster local o utilidades) |

### Convención por unidad / laboratorio

Según `docs/01-metodologia.md`, cada tema o `lab-XX` puede incluir:

- `teoria.md` — micro-partes teóricas breves  
- `demo/` + `DEMO.md` — ejemplo ya montado; el **formador no escribe código en vivo**, solo explica lo entregado  
- `LAB.md` — laboratorio **constructivo**: bloques de código en Markdown, pasos, validación y retos  

El `README.md` de cada carpeta de lab sigue siendo el índice / resumen; el guion detallado para alumnado irá en `LAB.md` cuando se redacte.

## Entorno de prácticas

**Estándar del curso:** [GitHub Codespaces](docs/02-codespace.md) sobre este repo. Terminal, herramientas y acceso al cluster Kafka/Kubernetes del aula se unifican ahí.

## Cómo usar este repo con Cursor (fuera del aula)

Abre la carpeta `curso-kubernetes-kafka` como **raíz del workspace** (`Archivo → Abrir carpeta`) para preparar material. En el aula oficial, el workspace de referencia es el **Codespace**.

## Estado del material

Estructura de bloques y labs lista; metodología y Codespace documentados. Pendiente: contenidos `teoria.md`, `demo/` y `LAB.md` por unidad, y definición técnica del dev container / credenciales del lab.
