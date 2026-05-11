# LAB 1 — Despliegue de aplicaciones

[← Anterior: Lab 0 — Entorno](../lab-00-entorno/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 2 — Escalado y actualizaciones →](../lab-02-escalado-actualizaciones/README.md)

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

## Paso 1 — Crear el namespace y el Deployment

Aplica el namespace y el Deployment con 3 réplicas:

```bash
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifests/namespace.yaml
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifests/deployment.yaml
```

Observa lo que ha creado el Deployment:

```bash
kubectl -n lab01 get deploy,rs,pods -o wide
```

Verás:

- **1 Deployment** `web` (lo declarado).
- **1 ReplicaSet** `web-<hash>` (creado por el Deployment).
- **3 Pods** `web-<hash>-<rand>` (creados por el ReplicaSet), en distintos nodos.

## Paso 2 — Exponer el Deployment con un Service

```bash
kubectl apply -f bloque-1-kubernetes/lab-01-despliegue/manifests/service.yaml
kubectl -n lab01 get svc,endpoints web
```

Lo importante:

- El Service `web` tiene una **ClusterIP** (`10.96.x.x` o similar) y un **nombre DNS** dentro del cluster: `web.lab01.svc.cluster.local`.
- Sus **Endpoints** son las **IPs de los pods** que casan con el selector `app=web`. Comprueba que coinciden con `kubectl -n lab01 get pods -o wide`.

## Paso 3 — Probar la conectividad interna

El Service `ClusterIP` **no es accesible desde fuera del cluster**. Para probarlo, levanta un pod cliente efímero dentro del propio cluster:

```bash
kubectl -n lab01 run curlbox --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s -o /dev/null -w "%{http_code}\n" http://web/; done'
```

Debería responder `200` cinco veces. Las peticiones se reparten internamente entre los pods (round-robin a nivel de conexión).

> Si la imagen `curlimages/curl` tarda en descargarse, ten paciencia; sólo ocurre la primera vez por nodo.

## Paso 4 — Comprobar la auto-reparación

Mata uno de los pods a mano y observa la reconciliación:

```bash
POD=$(kubectl -n lab01 get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab01 delete pod "$POD"
kubectl -n lab01 get pods -w
```

- El pod borrado pasa a `Terminating`.
- El **ReplicaSet** detecta que faltan réplicas y crea un pod nuevo (`Pending` → `ContainerCreating` → `Running`).
- El pod nuevo **tiene otro nombre y otra IP**, pero el Service sigue resolviendo a `web`. El cliente no necesita saber nada.

Pulsa `Ctrl+C` cuando el cluster vuelva a tener 3 pods `Running`.

## Comprobaciones

- [ ] `kubectl -n lab01 get deploy web` muestra `3/3` ready.
- [ ] `kubectl -n lab01 get endpoints web` lista **3 IPs**.
- [ ] El `curl` desde `curlbox` devuelve `200` repetidamente.
- [ ] Tras borrar un pod, el conteo vuelve a 3 sin intervención.

## Retos opcionales

1. **Cambia el selector del Service** a algo que no exista (`app: weeb`) con `kubectl -n lab01 edit svc web`. Observa que `endpoints` queda vacío y los `curl` fallan. Restáuralo. Aprendizaje: el Service descubre pods por **labels**.
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

Conserva el namespace `lab01` para el [Lab 2](../lab-02-escalado-actualizaciones/README.md), que lo reutiliza. Si quieres borrarlo:

```bash
kubectl delete namespace lab01
```

---

[← Anterior: Lab 0 — Entorno](../lab-00-entorno/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 2 — Escalado y actualizaciones →](../lab-02-escalado-actualizaciones/README.md)
