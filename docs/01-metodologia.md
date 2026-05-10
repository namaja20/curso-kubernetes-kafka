# Metodología didáctica

Este documento fija **cómo se imparte** el curso y **cómo debe organizarse** el material en el repositorio. El entorno de ejecución para formadores y alumnado es **único: un GitHub Codespace** abierto sobre este repo (o un fork controlado del mismo).

---

## 1. Principios

| Principio | Implicación en el repo |
|-----------|-------------------------|
| **Teoría breve y segmentada** | Cada “micro-parte” de un módulo tiene explicación corta, enfocada a lo que se va a ver en la demo o en el lab. |
| **El formador no escribe código en vivo** | Todo lo ejecutable o revisable en demo está **ya entregado** en el repo; el formador abre archivos, ejecuta comandos preparados y **explica** sobre lo que hay. |
| **Demos funcionales con ejemplos preconstruidos** | Carpetas `demo/` (o equivalente por tema) con manifiestos, scripts y apps listas; guía `DEMO.md` indica orden, comandos y puntos de narración. |
| **Laboratorios constructivos** | Documento principal del lab (p. ej. `LAB.md`) con **bloques de código en Markdown**, secuencia **paso a paso**, comprobaciones, validaciones y **retos**. |
| **Todo ocurre en el Codespace** | Terminal, editor, acceso a `kubectl`/CLI y a los clusters o servicios del aula se asume **desde el contenedor del Codespace**; no se depende de instalaciones en el PC del alumno. |

---

## 2. Ciclo por unidad didáctica

Cada **módulo** (o sub-tema dentro de un bloque) sigue el mismo ritmo:

```text
Teoría breve (micro-partes) → Demo guiada (repo preconstruido) → Laboratorio constructivo (alumnado)
```

1. **Teoría breve**  
   - Objetivo: vocabulario y modelo mental mínimo para entender lo que viene.  
   - Formato sugerido: `teoria.md` en la carpeta del tema, con secciones numeradas y cortas (párrafos + listas + un diagrama opcional).  
   - Sin implementación en vivo por parte del formador.

2. **Demo funcional**  
   - Objetivo: mostrar el comportamiento real del sistema con **artefactos ya presentes** en el repo.  
   - Formato: carpeta `demo/` con lo necesario (YAML, scripts, datos de ejemplo) + `DEMO.md` para el formador: qué archivo abrir primero, qué comando ejecutar, qué salida comentar, qué riesgo o limitación mencionar.  
   - El formador **no improvisa código**; solo navega, ejecuta lo documentado y explica.

3. **Laboratorio constructivo**  
   - Objetivo: que el alumnado **construya** la solución siguiendo el guion, con sensación de progresión y cierre con validación.  
   - Formato: `LAB.md` (o nombre acordado por bloque) con la estructura de la sección 4.

Entre demo y lab puede haber una transición explícita en el guion: “En la demo vimos X ya montado; en el lab vais a obtener X aplicando estos pasos.”

---

## 3. Convención de carpetas (recomendada)

Dentro de cada laboratorio o unidad (`lab-XX-...` o subcarpeta de tema):

| Elemento | Contenido |
|----------|-----------|
| `teoria.md` | Micro-partes teóricas del tema asociado al lab (si el lab aglutina el tema). |
| `demo/` | Ejemplos listos + `DEMO.md` (guía del formador). |
| `LAB.md` | Laboratorio constructivo para el alumnado (bloques de código, pasos, validación, retos). |
| `solucion/` *(opcional)* | Referencia para formador o para autocorrección tardía; **no** sustituye el proceso constructivo en aula. |

Si un bloque comparte una demo entre varios labs, puede existir `bloque-X/.../demos-comunes/nombre-demo/` y los `LAB.md` enlazan a esa ruta.

Los **fundamentos** de cada bloque (`fundamentos/`) pueden usar solo `teoria.md` + `demo/` sin `LAB.md`, o enlazar al primer lab que aterriza el concepto.

---

## 4. Plantilla lógica de `LAB.md` (laboratorio constructivo)

Cada `LAB.md` debería incluir, en este orden:

1. **Objetivo** y **resultado observable** (qué debe “verse” al terminar).  
2. **Prerrequisitos** (namespace, cluster, topic base, variables de entorno del Codespace — lo que aplique).  
3. **Pasos numerados**, cada uno con:  
   - Contexto de una frase.  
   - **Bloque de código** copiable (YAML, shell, etc.).  
   - **Probar / validar**: comando o comprobación explícita (salida esperada, `kubectl get`, consumo de mensaje, etc.).  
4. **Retos** (opcional pero recomendado): 1–2 extensiones sin solución desplegada en el cuerpo del lab (o con solución al final en `<details>`).  
5. **Qué se aprende** (cierre operativo, alineado con el programa).

Los bloques de código deben ser **coherentes entre sí** (mismo nombre de deployment, mismo topic, etc.) para que copiar y pegar en orden funcione en el Codespace.

---

## 5. Rol del formador vs. alumnado

- **Formador:** narración sobre `teoria.md` y `DEMO.md`, ejecución de comandos ya especificados, respuesta a dudas, moderación de retos.  
- **Alumnado:** sigue `LAB.md` en el Codespace, pega/edita según el guion, valida en cada paso.  
- **Repo:** es la **fuente única de verdad** para demos y labs; evita desviaciones entre “lo que dijo el instructor” y “lo que hay que entregar”.

---

## 6. GitHub Codespace

- El **workspace** del alumno y del formador es el **repositorio del curso** (o plantilla) abierto en Codespaces.  
- Herramientas de línea de comandos, extensiones y **acceso al cluster Kafka/Kubernetes** del aula deben estar definidos en la configuración del Codespace (p. ej. `.devcontainer/`, secretos de GitHub, `KUBECONFIG`, endpoints). Esa definición técnica se completará al fijar el stack concreto del proveedor de laboratorio.  
- Cualquier script de aprovisionamiento “local” del repo (p. ej. bajo `entorno-local/`) se entiende como **invocable desde el Codespace**, no como entorno alternativo en máquina física, salvo decisión explícita contraria documentada en el README.

---

## 7. Relación con el programa (`docs/00-programa.md`)

Los bloques de horas y los LAB 1–14 del programa se mantienen. Esta metodología indica **cómo** rellenar cada carpeta: teoría corta, demo preconstruida, lab constructivo con Markdown ejecutable y retos, todo dentro del flujo Codespace.
