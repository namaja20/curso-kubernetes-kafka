# Entorno: GitHub Codespaces

Todo el curso se imparte y practica **dentro de un Codespace** asociado a este repositorio.

## Objetivos del entorno

- Mismo editor, terminal y herramientas para todo el aula.  
- Sin dependencia de SO o permisos en el portátil del alumno.  
- Acceso homogéneo a los recursos del laboratorio (Kubernetes, Kafka/Confluent, CLIs).

## Qué falta por concretar (siguiente capa de trabajo)

Cuando se cierre el diseño con el proveedor del lab:

- Imagen base del dev container (herramientas: `kubectl`, cliente Kafka/Confluent, etc.).  
- Cómo se inyecta **kubeconfig**, tokens o URLs (GitHub Secrets, Codespace secrets, script de inicio).  
- Si el cluster es **remoto** (compartido) o se **levanta** desde el Codespace (p. ej. Kind/K3d), y los scripts correspondientes en `entorno-local/` o `.devcontainer/`.

Hasta entonces, los documentos de metodología y los `LAB.md` asumen “terminal del Codespace + acceso al cluster según variables documentadas en cada lab”.

## Uso para el formador

- Abrir el repo en Codespace, seguir `docs/01-metodologia.md` y los `DEMO.md` de cada unidad.  
- No es necesario clonar en local salvo preferencia personal; el estándar del curso es Codespace.

## Uso para el alumnado

- Crear Codespace desde la rama o plantilla indicada por el instructor.  
- Trabajar cada `LAB.md` en el editor integrado; copiar bloques de código al terminal del Codespace.
