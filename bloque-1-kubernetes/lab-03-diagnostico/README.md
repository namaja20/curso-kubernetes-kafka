# LAB 3 — Diagnóstico de incidencias

[← Anterior: Lab 2 — Escalado](../lab-02-escalado-actualizaciones/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 4 — ConfigMaps y Secrets →](../lab-04-configuracion/README.md)

---

## Objetivo

Familiarizarse con el **kit de diagnóstico básico** de Kubernetes (`get`, `describe`, `logs`, `events`) provocando a propósito los cuatro errores más típicos en producción:

1. **`ImagePullBackOff`** — el cluster no consigue descargar la imagen.
2. **`CrashLoopBackOff`** — el contenedor arranca y muere en bucle.
3. **`Pending`** por falta de recursos — el scheduler no encuentra dónde colocar el pod.
4. **Pod `Running` pero `0/1 Ready`** — la app está viva pero la *readiness probe* falla; el Service no le manda tráfico.

## Prerrequisitos

- Cluster `kind-curso` del [Lab 0](../lab-00-entorno/README.md) operativo.

Despliega los cuatro escenarios:

```bash
kubectl apply -f bloque-1-kubernetes/lab-03-diagnostico/manifests/
```

Comprueba el panorama:

```bash
kubectl -n lab03 get deploy,pods
```

Verás 4 Deployments en distintos estados de "no felicidad". A partir de aquí, **el orden de los pasos no importa**: cada bloque es independiente.

## Conceptos rápidos

- **`kubectl get`** — visión panorámica de objetos.
- **`kubectl describe`** — detalle de un objeto + sus `Events` recientes (suele ser donde está la respuesta).
- **`kubectl logs`** — stdout/stderr del contenedor. Con `-p` ves los logs del contenedor **anterior** si reinició.
- **`kubectl get events`** — eventos del namespace ordenados por tiempo.
- **Fases del pod** — `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown` (no confundir con la columna `READY n/m`).

## Escenario 1 — `ImagePullBackOff`

```bash
kubectl -n lab03 get pod -l app=bad-image
```

Pod en `ImagePullBackOff` (después de unos segundos de `ErrImagePull`).

Diagnóstico:

```bash
kubectl -n lab03 describe pod -l app=bad-image
```

Mira el bloque `Events` al final. Verás algo como:

```
Failed to pull image "nginx:this-tag-does-not-exist": ... manifest unknown
```

**Lección:** los errores de pull aparecen en `Events`, no en `logs` (el contenedor nunca llegó a ejecutarse).

## Escenario 2 — `CrashLoopBackOff`

```bash
kubectl -n lab03 get pod -l app=crashloop
```

El pod alterna `Running` → `Error`/`CrashLoopBackOff`. La columna `RESTARTS` no para de crecer.

Diagnóstico:

```bash
kubectl -n lab03 logs -l app=crashloop --tail=20
```

(Si el pod ya ha reiniciado, prueba `--previous` para ver los logs anteriores al último reinicio):

```bash
POD=$(kubectl -n lab03 get pod -l app=crashloop -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab03 logs "$POD" --previous
```

Verás los `echo` y el `exit 1`. **Lección:** cuando el contenedor sí arranca, los logs son tu primera parada.

## Escenario 3 — `Pending` por falta de recursos

```bash
kubectl -n lab03 get pod -l app=too-big
```

El pod queda en `Pending` indefinidamente.

Diagnóstico:

```bash
kubectl -n lab03 describe pod -l app=too-big
```

Sección `Events`:

```
0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
```

El **scheduler** ha rechazado todos los nodos. **Lección:** `Pending` es problema *previo* a arrancar (scheduling, volumen, imagen aún no descargada). El nodo nunca llega a recibir el pod.

Solución pedagógica — bajar los `requests`:

```bash
kubectl -n lab03 patch deploy too-big --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"25m","memory":"32Mi"},"limits":{"cpu":"100m","memory":"64Mi"}}}]'
```

En segundos pasa a `Running`.

## Escenario 4 — `Running` pero `0/1 Ready`

```bash
kubectl -n lab03 get pod -l app=not-ready
```

Columna `STATUS` = `Running`; columna `READY` = `0/1`. **El contenedor está vivo, pero el Service no lo considera apto.**

Comprueba que el Service correspondiente no tiene endpoints:

```bash
kubectl -n lab03 get endpoints not-ready
```

(Te dará `<none>`.)

Diagnóstico:

```bash
kubectl -n lab03 describe pod -l app=not-ready | sed -n '/Events:/,$p'
```

Verás `Readiness probe failed: HTTP probe failed with statuscode: 404`.

**Lección:** un fallo de readiness **no reinicia el pod** (eso lo hace la *liveness*), pero lo saca del Service. Es uno de los errores más sutiles de detectar: la app "está corriendo" y aun así nadie la ve.

Solución — corregir el path de la probe:

```bash
kubectl -n lab03 patch deploy not-ready --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"}]'
```

Tras unos segundos los pods quedan `1/1 Ready` y el Service obtiene endpoints.

## Comprobaciones

- [ ] Has identificado el motivo de cada escenario sólo con `describe` y `logs`.
- [ ] Entiendes la diferencia entre **fase del pod** (`Running`) y **readiness** (`0/1`).
- [ ] Sabes cuándo mirar `logs` y cuándo mirar `events`.

## Retos opcionales

1. **Watch global de eventos** mientras provocas un fallo:
   ```bash
   kubectl -n lab03 get events -w
   ```
2. **`kubectl debug`** — adjuntar un contenedor con `curl`/`netcat` a un pod problemático (sin reiniciarlo):
   ```bash
   POD=$(kubectl -n lab03 get pod -l app=not-ready -o jsonpath='{.items[0].metadata.name}')
   kubectl -n lab03 debug "$POD" -it --image=curlimages/curl:8.10.1 --target=nginx -- sh
   ```
3. **Forzar OOMKill**: añade `limits.memory: 8Mi` a un nginx y observa `OOMKilled` en `kubectl describe pod`.

## Troubleshooting (del Lab, no de la app)

| Síntoma | Causa | Acción |
|---------|-------|--------|
| `kubectl describe` no muestra `Events` | Han pasado más de ~1h y el cluster los purgó | Repite la acción y mira en caliente. |
| El `patch` falla con "the object has been modified" | Carrera con el controller | Reintenta. |
| Pods no terminan de borrarse al final | Finalizers pendientes | `kubectl -n lab03 get pod` y, si urge, `--force --grace-period=0`. |

## Limpieza

```bash
kubectl delete namespace lab03
```

---

[← Anterior: Lab 2 — Escalado](../lab-02-escalado-actualizaciones/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 4 — ConfigMaps y Secrets →](../lab-04-configuracion/README.md)
