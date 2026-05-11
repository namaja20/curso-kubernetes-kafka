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

## Organización de los manifiestos

Cada escenario es una **app distinta** con su propio Deployment. El namespace del lab cuelga aparte porque lo comparten todas:

```
lab-03-diagnostico/
└── manifest/
    ├── 00-namespace.yaml
    └── apps/
        ├── bad-image/        # Escenario 1: ImagePullBackOff
        │   └── deployment.yaml
        ├── crashloop/        # Escenario 2: CrashLoopBackOff
        │   └── deployment.yaml
        ├── too-big/          # Escenario 3: Pending por recursos
        │   └── deployment.yaml
        └── not-ready/        # Escenario 4: Running pero no Ready
            ├── deployment.yaml
            └── service.yaml
```

Aplica todo de golpe con `-R` (recursivo):

```bash
kubectl apply -R -f bloque-1-kubernetes/lab-03-diagnostico/manifest/
```

Y comprueba el panorama:

```bash
kubectl -n lab03 get deploy,pods
```

Verás 4 Deployments en distintos estados de "no felicidad". A partir de aquí, **el orden de los escenarios no importa**: cada bloque es independiente.

## Conceptos rápidos

- **`kubectl get`** — visión panorámica de objetos.
- **`kubectl describe`** — detalle de un objeto + sus `Events` recientes (suele ser donde está la respuesta).
- **`kubectl logs`** — stdout/stderr del contenedor. Con `-p` ves los logs del contenedor **anterior** si reinició.
- **`kubectl get events`** — eventos del namespace ordenados por tiempo.
- **Fases del pod** — `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`. **No confundir** con la columna `READY n/m`, que indica cuántos contenedores del pod están pasando su *readiness*.

## Escenario 1 — `ImagePullBackOff`

### El manifiesto

[`manifest/apps/bad-image/deployment.yaml`](manifest/apps/bad-image/deployment.yaml). La única "trampa" es la etiqueta de imagen:

```yaml
    spec:
      containers:
        - name: app
          image: nginx:this-tag-does-not-exist   # tag inventado
```

Sin probes ni recursos: queremos aislar exclusivamente el fallo de **descarga** de la imagen. El kubelet pide la imagen, falla, espera (back-off exponencial), reintenta… y el pod nunca llega a ejecutarse.

### Diagnóstico

```bash
kubectl -n lab03 get pod -l app=bad-image
kubectl -n lab03 describe pod -l app=bad-image
```

Mira el bloque `Events` al final del describe:

```
Failed to pull image "nginx:this-tag-does-not-exist": ... manifest unknown
Back-off pulling image "nginx:this-tag-does-not-exist"
```

**Lección:** los errores de pull aparecen en `Events`, no en `logs` (el contenedor nunca llegó a ejecutarse).

## Escenario 2 — `CrashLoopBackOff`

### El manifiesto

[`manifest/apps/crashloop/deployment.yaml`](manifest/apps/crashloop/deployment.yaml). Aquí la imagen **sí** baja, pero el `command` está pensado para morir:

```yaml
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - |
              echo "Iniciando..."
              sleep 2
              echo "Algo fue mal" >&2
              exit 1                     # ← salida con error
```

`exit 1` hace que el contenedor termine con un código distinto de cero. Como la `restartPolicy` por defecto del Pod es `Always`, el kubelet lo reinicia. Tras unos cuantos reintentos rápidos, entra en **CrashLoopBackOff** (espera creciente entre reinicios).

### Diagnóstico

```bash
kubectl -n lab03 get pod -l app=crashloop
```

Verás `STATUS` alternando entre `Running` y `CrashLoopBackOff` o `Error`, y la columna `RESTARTS` creciendo.

Logs en caliente:

```bash
kubectl -n lab03 logs -l app=crashloop --tail=20
```

Si el pod **ya reinició**, los logs que ves son de la instancia actual. Para los del contenedor anterior:

```bash
POD=$(kubectl -n lab03 get pod -l app=crashloop -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab03 logs "$POD" --previous
```

**Lección:** cuando el contenedor arranca aunque sea brevemente, los **logs son tu primera parada**. `events` complementa, no sustituye.

## Escenario 3 — `Pending` por falta de recursos

### El manifiesto

[`manifest/apps/too-big/deployment.yaml`](manifest/apps/too-big/deployment.yaml). El pod pide más de lo que ningún nodo puede ofrecer:

```yaml
          resources:
            requests:
              cpu: "32"          # 32 cores garantizados
              memory: 64Gi       # 64 GiB
            limits:
              cpu: "32"
              memory: 64Gi
```

`requests` participa en el **scheduling**: el scheduler busca un nodo con al menos esa capacidad libre. Como nuestros nodos kind tienen unos pocos GB y unas pocas vCPU, **ningún nodo cumple**.

### Diagnóstico

```bash
kubectl -n lab03 get pod -l app=too-big
kubectl -n lab03 describe pod -l app=too-big
```

En `Events`:

```
0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
```

**Lección:** `Pending` es problema *previo* a arrancar (scheduling, volumen sin enlazar, imagen aún no descargada). El nodo nunca llega a recibir el pod, así que no hay logs.

### Solución pedagógica

Bajar los `requests` con un `patch`:

```bash
kubectl -n lab03 patch deploy too-big --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"25m","memory":"32Mi"},"limits":{"cpu":"100m","memory":"64Mi"}}}]'
```

En segundos pasa a `Running`.

## Escenario 4 — `Running` pero `0/1 Ready`

### Los manifiestos

[`manifest/apps/not-ready/deployment.yaml`](manifest/apps/not-ready/deployment.yaml). La readiness apunta a un path que **no existe** en nginx:

```yaml
          readinessProbe:
            httpGet:
              path: /esto-no-existe   # devolverá 404
              port: 80
            initialDelaySeconds: 1
            periodSeconds: 3
            failureThreshold: 2
```

[`manifest/apps/not-ready/service.yaml`](manifest/apps/not-ready/service.yaml) es un ClusterIP estándar, **necesario** para visualizar la consecuencia: el Service mira la readiness antes de incluir un Pod en sus endpoints.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: not-ready
  namespace: lab03
spec:
  selector:
    app: not-ready
  ports:
    - port: 80
      targetPort: 80
```

### Diagnóstico

```bash
kubectl -n lab03 get pod -l app=not-ready
```

Columna `STATUS` = `Running`; columna `READY` = `0/1`. **El contenedor está vivo, pero el Service no lo considera apto.**

Comprueba que el Service no tiene endpoints:

```bash
kubectl -n lab03 get endpoints not-ready
```

(Te dará `<none>`.) El motivo:

```bash
kubectl -n lab03 describe pod -l app=not-ready | sed -n '/Events:/,$p'
```

```
Readiness probe failed: HTTP probe failed with statuscode: 404
```

**Lección:** un fallo de readiness **no reinicia el pod** (eso es trabajo de la *liveness probe*), pero lo saca del Service. Es uno de los errores más sutiles: la app "está corriendo" y aun así nadie la ve.

### Solución

Corregir el path con un patch:

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
