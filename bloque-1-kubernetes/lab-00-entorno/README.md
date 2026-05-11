# LAB 0 — Preparar el entorno (kind + kubectl)

[← Índice del bloque](../fundamentos/README.md) · [Siguiente: Lab 1 — Despliegue de aplicaciones →](../lab-01-despliegue/README.md)

---

## Objetivo

Dejar el entorno listo para todos los laboratorios siguientes: instalar y verificar **Docker**, **kubectl** y **kind**, y arrancar un cluster local de Kubernetes de **3 nodos** que se reutilizará a lo largo del curso.

Al terminar este laboratorio tendrás:

- `docker`, `kubectl` y `kind` instalados y accesibles desde la línea de comandos.
- Un cluster Kubernetes en local llamado **`curso`** con un nodo de control y dos nodos worker.
- `kubectl` configurado y apuntando a ese cluster por defecto.

## Prerrequisitos

- Estar dentro de un **GitHub Codespace** del fork del repositorio (recomendado) o disponer de un entorno **Linux / macOS** con **Docker** ya funcionando.
- Acceso a Internet para descargar imágenes y binarios.
- ~4 GB de RAM disponibles para el cluster (3 nodos kind ligeros).

> En Codespaces, Docker viene preinstalado y los binarios se instalan **por usuario** (sin `sudo`).

## Conceptos rápidos

- **Docker** — engine de contenedores. kind ejecuta cada nodo del cluster como un contenedor Docker.
- **kubectl** — CLI oficial para hablar con la API de Kubernetes.
- **kind (Kubernetes-in-Docker)** — herramienta que crea clusters Kubernetes corriendo los nodos como contenedores Docker. Ideal para desarrollo, formación y CI; **no para producción**.

## Paso 1 — Verificar Docker

```bash
docker info | head -10
```

Debe mostrar la versión del *Server* y *Containers* sin errores. Si falla, no continúes hasta resolverlo (ver [Troubleshooting](#troubleshooting)).

## Paso 2 — Instalar `kubectl`

Descarga la última versión estable y déjala en `~/.local/bin`, que está en el `PATH` de Codespaces:

```bash
mkdir -p "$HOME/.local/bin"
curl -L "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  -o "$HOME/.local/bin/kubectl"
chmod +x "$HOME/.local/bin/kubectl"
```

Verifica:

```bash
kubectl version --client
```

> En **macOS** (entorno local) sustituye `linux/amd64` por `darwin/arm64` (Apple Silicon) o `darwin/amd64` (Intel). En Linux ARM, usa `linux/arm64`.

## Paso 3 — Instalar `kind`

```bash
curl -L "https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64" \
  -o "$HOME/.local/bin/kind"
chmod +x "$HOME/.local/bin/kind"
kind version
```

## Paso 4 — Crear el cluster

### El manifiesto del cluster

Este laboratorio incluye una configuración mínima en [`manifest/cluster/kind-cluster.yaml`](manifest/cluster/kind-cluster.yaml). Es un fichero que pertenece **a kind**, no a la API de Kubernetes: define *cómo se construye* el cluster, no objetos que vivan dentro de él.

```yaml
kind: Cluster                       # tipo de recurso de kind (no de Kubernetes)
apiVersion: kind.x-k8s.io/v1alpha4  # esquema de configuración de kind
name: curso                         # nombre del cluster -> contexto kubectl: kind-curso
nodes:                              # cada elemento se materializa como un contenedor Docker
  - role: control-plane             # 1 nodo de control (apiserver, etcd, scheduler, controllers)
  - role: worker                    # 1 worker (ejecuta pods de usuario)
  - role: worker                    # 2 workers para poder ver scheduling entre nodos
```

Puntos a destacar:

- **`name: curso`** se convierte en el contexto `kind-curso` de `kubectl`. Cambiarlo aquí cambia el contexto.
- Tener **2 workers** (y no 1) permite observar más adelante cómo Kubernetes reparte pods entre nodos y cómo se comporta el `kube-proxy` al enrutar entre IPs de pods distintas.
- La organización del repo agrupa todos los YAML del lab bajo `manifest/`. Para infraestructura del propio cluster usamos `manifest/cluster/`; para aplicaciones, `manifest/apps/<nombre-de-la-app>/`. Verás este patrón en todos los labs.

### Crearlo

```bash
kind create cluster --config bloque-1-kubernetes/lab-00-entorno/manifest/cluster/kind-cluster.yaml
```

> La primera vez kind descargará la imagen de nodo (`kindest/node:vX.YY.Z`); puede tardar varios minutos.

Verifica que `kubectl` apunta automáticamente al cluster recién creado:

```bash
kubectl config current-context
```

Debe responder `kind-curso`.

## Paso 5 — Inspeccionar el cluster

Recorrer una vez los objetos del sistema ayuda a fijar el vocabulario que se usará en los siguientes labs:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

Cosas a observar:

- **3 nodos** en estado `Ready` (`curso-control-plane`, `curso-worker`, `curso-worker2`).
- En el namespace `kube-system` aparecen pods de **CoreDNS**, **kube-proxy**, **kindnet** (CNI), `etcd`, `kube-apiserver`, `kube-scheduler` y `kube-controller-manager`. Son justo las piezas vistas en el capítulo [Arquitectura de Kubernetes](../fundamentos/04-arquitectura-k8s.md).
- `kubectl cluster-info` muestra la URL del API server al que se está conectando.

## Comprobaciones

El entorno está listo cuando todo lo siguiente es cierto:

- [ ] `kubectl version --client` muestra una versión, sin errores de conexión.
- [ ] `kind version` muestra una versión.
- [ ] `kubectl config current-context` devuelve `kind-curso`.
- [ ] `kubectl get nodes` lista **3 nodos** en estado `Ready`.
- [ ] `kubectl get pods -A` no muestra pods en `CrashLoopBackOff` ni en `Error`.

## Retos opcionales

1. Borra el cluster y recréalo con un nombre distinto:
   ```bash
   kind delete cluster --name curso
   kind create cluster --name pruebas
   kind get clusters
   kubectl config get-contexts
   ```
   Verás cómo `kind` añade un *kubeconfig context* nuevo por cada cluster (`kind-pruebas`).

2. Lista los contenedores Docker que componen el cluster:
   ```bash
   docker ps --filter "label=io.x-k8s.kind.cluster=curso"
   ```
   Cada nodo Kubernetes del cluster es **un contenedor Docker** del host. Esto materializa la idea del capítulo [Contenedores vs VMs](../fundamentos/01-contenedores-vs-vms.md).

3. Recupera el `kubeconfig` exportado por kind y léelo:
   ```bash
   kind get kubeconfig --name curso | less
   ```

## Troubleshooting

| Síntoma | Causa habitual | Solución |
|---------|----------------|----------|
| `Cannot connect to the Docker daemon` | Docker no está corriendo | En local: `sudo systemctl start docker`. En Codespaces: reabrir el Codespace. |
| `kubectl: command not found` tras instalar | `~/.local/bin` no está en el PATH | `export PATH="$HOME/.local/bin:$PATH"` y reabrir la terminal. |
| `kind create cluster` se cuelga en *ensuring node image* | Primera descarga lenta | Espera; si tarda demasiado, cancela (`Ctrl+C`), `kind delete cluster --name curso` y reintenta. |
| Pods en `Pending` por falta de CPU/RAM | Codespace pequeño | Edita `kind-cluster.yaml` y deja **un único worker**, o reduce el tipo de Codespace. |
| `kubectl get nodes` no responde | Contexto incorrecto | `kubectl config use-context kind-curso`. |

## Limpieza

Por defecto **se recomienda dejar el cluster levantado** entre sesiones: los siguientes laboratorios lo reutilizan. Si necesitas liberar memoria:

```bash
kind delete cluster --name curso
```

Para volver a empezar, repite el **Paso 4**.

## Lo que viene a continuación

Entorno preparado. El primer laboratorio práctico de Kubernetes despliega y expone una aplicación sobre este cluster.

---

[← Índice del bloque](../fundamentos/README.md) · [Siguiente: Lab 1 — Despliegue de aplicaciones →](../lab-01-despliegue/README.md)
