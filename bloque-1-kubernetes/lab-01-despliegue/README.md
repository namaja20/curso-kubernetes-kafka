# LAB 1 — Despliegue de aplicaciones

> **Has entrado aquí desde el [Capítulo 2 — Runtime y CRI](../fundamentos/02-runtime-y-cri.md).** Al terminar, vuelve a ese capítulo y continúa con la teoría hasta el siguiente laboratorio.

[← Volver: Capítulo 2 — Runtime y CRI](../fundamentos/02-runtime-y-cri.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Desplegar una aplicación contenedorizada en Kubernetes, exponerla con un Service y observar que **los pods son efímeros** mientras el Service mantiene un nombre estable.

Al terminar tendrás:

- Un **namespace** propio (`lab01`) con un Deployment de **3 réplicas** de nginx.
- Un **Service** `ClusterIP` que reparte tráfico entre esas réplicas.
- Verificada la **auto-reparación**: al borrar un pod, Kubernetes lo recrea.

## Prerrequisitos

- Haber completado el [Lab 0](../lab-00-entorno/README.md): cluster `kind-curso` activo.
- `kubectl config current-context` devuelve `kind-curso`.

## Conceptos rápidos

- **Namespace** — partición lógica del cluster; aísla nombres de objetos.
- **Deployment** — controlador que mantiene N réplicas idénticas de un Pod.
- **ReplicaSet** — recurso intermedio que crea/destruye Pods; lo gestiona el Deployment.
- **Service (ClusterIP)** — DNS estable e IP virtual interna; reparte tráfico entre los pods cuyas labels coinciden con su `selector`.

## Organización de los manifiestos

Todos los YAML viven bajo `manifest/`. Los recursos *infra* del laboratorio (como el namespace) cuelgan directamente, y cada aplicación tiene su propia subcarpeta dentro de `apps/`:

```
lab-01-despliegue/
└── manifest/
    ├── 00-namespace.yaml          # infra del lab
    └── apps/
        └── web/                   # nuestra app de ejemplo
            ├── deployment.yaml
            └── service.yaml
```

Aplicaremos los archivos en este mismo orden, porque un Deployment no puede crearse si su namespace aún no existe.

## Paso 1 — Crear el namespace

### El manifiesto

[`manifest/00-namespace.yaml`](manifest/00-namespace.yaml):

```yaml
apiVersion: v1          # API "core/v1" para los objetos básicos
kind: Namespace         # tipo de recurso
metadata:
  name: lab01           # nombre lógico; aparecerá en kubectl -n lab01
```

Un **Namespace** es lo mínimo que necesita Kubernetes para *segregar* los objetos del lab del resto del cluster. No tiene `spec`: su única información relevante es el nombre.

### Aplicarlo

```bash
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifest/00-namespace.yaml
kubectl get ns lab01
```

## Paso 2 — Desplegar la aplicación `web`

### El manifiesto

[`manifest/apps/web/deployment.yaml`](manifest/apps/web/deployment.yaml):

```yaml
apiVersion: apps/v1               # los Deployments viven en el grupo "apps"
kind: Deployment
metadata:
  name: web                       # nombre del Deployment
  namespace: lab01                # se crea dentro del namespace del lab
  labels:
    app: web                      # label del propio Deployment (útil para listar/filtrar)
spec:
  replicas: 3                     # número deseado de réplicas
  selector:
    matchLabels:
      app: web                    # el Deployment "adopta" Pods con esta label
  template:                       # plantilla a partir de la cual se crean los Pods
    metadata:
      labels:
        app: web                  # debe coincidir con el selector de arriba
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80   # informativo: documenta el puerto del contenedor
          readinessProbe:         # cuándo el pod debe considerarse "apto" para tráfico
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            periodSeconds: 5
          resources:              # límites de uso para que kind no se ahogue
            requests:
              cpu: 25m            # 0.025 cores garantizados
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
```

Tres ideas a destacar:

- **`selector.matchLabels` debe casar con `template.metadata.labels`.** Si no, el Deployment crearía Pods que él mismo no reconoce y entraría en bucle. Es el error nº1 de los recién llegados.
- **`readinessProbe`** es lo que diferencia *"el contenedor está corriendo"* de *"está listo para recibir tráfico"*. El Service usa esta señal: si la probe falla, el pod queda fuera de los endpoints (lo veremos a fondo en Lab 3).
- **`resources.requests`** condicionan el **scheduling** (el nodo debe tener al menos eso libre). **`resources.limits`** condicionan el runtime (si se pasa, lo mata el kernel via cgroups).

### Aplicarlo y observar

```bash
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifest/apps/web/deployment.yaml
kubectl -n lab01 get deploy,rs,pods -o wide
```

Verás:

- **1 Deployment** `web` (lo declarado).
- **1 ReplicaSet** `web-<hash>` (creado por el Deployment).
- **3 Pods** `web-<hash>-<rand>`, repartidos entre nodos.

## Paso 3 — Exponer la aplicación con un Service

### El manifiesto

[`manifest/apps/web/service.yaml`](manifest/apps/web/service.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web                       # DNS interno: web.lab01.svc.cluster.local
  namespace: lab01
spec:
  type: ClusterIP                 # virtual IP, accesible SOLO dentro del cluster
  selector:
    app: web                      # apuntará a TODOS los Pods con label app=web
  ports:
    - name: http
      port: 80                    # puerto del Service (lo que ve el cliente)
      targetPort: 80              # puerto del Pod (containerPort)
```

Clave de funcionamiento:

- El Service **no conoce a los Pods uno por uno**. El controlador de Endpoints calcula la lista en tiempo real a partir del **`selector`**.
- `port` y `targetPort` permiten exponer un puerto distinto del que escucha el contenedor (aquí coinciden por simplicidad).
- `type: ClusterIP` es el tipo más restrictivo y el más usado dentro del cluster. Para exponer fuera existen `NodePort`, `LoadBalancer` e `Ingress`, que no necesitamos aquí.

### Aplicarlo

```bash
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifest/apps/web/service.yaml
kubectl -n lab01 get svc,endpoints web
```

Observa que **los Endpoints son las IPs reales de los pods**. Compáralas con:

```bash
kubectl -n lab01 get pods -o wide
```

## Paso 4 — Probar la conectividad interna

El Service `ClusterIP` **no es accesible desde fuera del cluster**. Para probarlo, levanta un pod cliente efímero dentro del propio cluster:

```bash
kubectl -n lab01 run curlbox --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s -o /dev/null -w "%{http_code}\n" http://web/; done'
```

Debe responder `200` cinco veces. Las peticiones se reparten internamente entre los pods (round-robin a nivel de conexión).

> Aquí estamos resolviendo el nombre `web` porque el pod cliente vive en el namespace `lab01`. Desde otro namespace habría que usar el FQDN `web.lab01.svc.cluster.local`.

## Paso 5 — Comprobar la auto-reparación

Mata un pod a mano y observa la reconciliación:

```bash
POD=$(kubectl -n lab01 get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab01 delete pod "$POD"
kubectl -n lab01 get pods -w
```

- El pod borrado pasa a `Terminating`.
- El **ReplicaSet** detecta que faltan réplicas y crea uno nuevo (`Pending` → `ContainerCreating` → `Running`).
- El pod nuevo **tiene otro nombre y otra IP**, pero el Service sigue resolviendo a `web`. El cliente no necesita saber nada.

Pulsa `Ctrl+C` cuando el cluster vuelva a tener 3 pods `Running`.

## Comprobaciones

- [ ] `kubectl -n lab01 get deploy web` muestra `3/3` ready.
- [ ] `kubectl -n lab01 get endpoints web` lista **3 IPs**.
- [ ] El `curl` desde `curlbox` devuelve `200` repetidamente.
- [ ] Tras borrar un pod, el conteo vuelve a 3 sin intervención.

## Retos opcionales

1. **Cambia el selector del Service** a algo que no exista (`app: weeb`) con `kubectl -n lab01 edit svc web`. Observa que `endpoints` queda vacío y los `curl` fallan. Restáuralo. Aprendizaje: el Service descubre pods por **labels**, no por nombres.
2. **Escala manualmente** a 5 réplicas:
   ```bash
   kubectl -n lab01 scale deploy/web --replicas=5
   kubectl -n lab01 get pods -w
   ```
   Sin tocar el Service, los nuevos pods reciben tráfico automáticamente.
3. **Inspecciona los Events** del namespace:
   ```bash
   kubectl -n lab01 get events --sort-by=.lastTimestamp
   ```

## Troubleshooting

| Síntoma | Causa habitual | Solución |
|---------|----------------|----------|
| Pods en `ImagePullBackOff` | Sin acceso a Docker Hub | `kubectl -n lab01 describe pod <pod>`; revisar conectividad del Codespace. |
| Pods en `Pending` | Falta de CPU/RAM | Reducir réplicas o redimensionar el cluster. |
| `curl: (6) Could not resolve host: web` | No estás dentro del namespace `lab01` o el Service no está creado | Verifica `kubectl -n lab01 get svc web`. |
| Service sin endpoints | El selector no casa con las labels del Deployment | `kubectl -n lab01 get pods --show-labels` y `kubectl -n lab01 describe svc web`. |

## Limpieza

Conserva el namespace `lab01` para el siguiente laboratorio del bloque, que lo reutiliza. Si quieres borrarlo:

```bash
kubectl delete namespace lab01
```

## Vuelve a la teoría

Has completado el laboratorio. Regresa al capítulo desde el que llegaste y continúa la lectura:

[← Volver al **Capítulo 2 — Runtime y CRI**](../fundamentos/02-runtime-y-cri.md)

Cuando lo termines, el siguiente capítulo de teoría es:

[Capítulo 3 — Orquestación →](../fundamentos/03-orquestacion.md)

---

[← Volver: Capítulo 2 — Runtime y CRI](../fundamentos/02-runtime-y-cri.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente capítulo: 3 — Orquestación →](../fundamentos/03-orquestacion.md)
