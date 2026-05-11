# LAB 4 — ConfigMaps y Secrets

[← Anterior: Lab 3 — Diagnóstico](../lab-03-diagnostico/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente bloque: Kafka + Confluent →](../../bloque-2-kafka-confluent/fundamentos/README.md)

---

## Objetivo

Separar **código** y **configuración**. Aprender a:

- Inyectar configuración como **variables de entorno** desde un ConfigMap y desde un Secret.
- Montar configuración como **archivo** dentro del contenedor (volumen).
- Modificar la configuración y propagar el cambio a los pods (`rollout restart`).

## Prerrequisitos

- Cluster `kind-curso` del [Lab 0](../lab-00-entorno/README.md) operativo.

## Conceptos rápidos

- **ConfigMap** — pares clave/valor (o ficheros) para configuración **no sensible**. Texto plano.
- **Secret** — igual, pero codificado en base64 y con tratamiento aparte (RBAC, *encryption at rest* si el cluster lo activa). **No** es cifrado por sí solo.
- **Tres formas de consumir** ambos:
  1. Como **variable de entorno** (`env.valueFrom.configMapKeyRef` / `secretKeyRef`).
  2. Como **bloque completo** de variables (`envFrom`).
  3. Como **fichero** dentro de un volumen (`volumes` + `volumeMounts`).
- **Propagación de cambios:** los *volúmenes* refrescan automáticamente (con un pequeño retardo); las *variables de entorno* **no** — necesitan reinicio del pod.

## Paso 1 — Crear ConfigMap y Secret

```bash
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifests/00-namespace.yaml
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifests/01-configmap.yaml
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifests/02-secret.yaml
```

Inspecciona ambos:

```bash
kubectl -n lab04 get cm app-config -o yaml
kubectl -n lab04 get secret app-secret -o yaml
```

Fíjate en el Secret: el valor de `API_TOKEN` aparece en **base64** (`echo -n s3cr3t-no-lo-cuentes | base64` lo confirma). **Base64 no es cifrado.**

Para verlo en claro:

```bash
kubectl -n lab04 get secret app-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d; echo
```

## Paso 2 — Desplegar la app que consume ambos

```bash
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifests/03-deployment.yaml
kubectl -n lab04 rollout status deploy/demo
```

La app es un script `busybox` que imprime las variables y el fichero montado, y luego entra en un bucle. Mira la salida:

```bash
kubectl -n lab04 logs deploy/demo --tail=40
```

Deberías ver, **al principio**:

```
==== ENV ====
GREETING=Hola desde ConfigMap
LOG_LEVEL=info
API_TOKEN=s3cr3t-no-lo-cuentes
==== /etc/app/app.properties ====
feature.flag.beta=false
cache.ttl.seconds=60
welcome.message=Bienvenido al curso
```

Y a continuación líneas periódicas tipo `[12:34:56] alive, GREETING=Hola desde ConfigMap`.

## Paso 3 — Cambiar el fichero del ConfigMap (propagación automática)

Edita el valor de `app.properties` dentro del ConfigMap:

```bash
kubectl -n lab04 patch cm app-config --type=merge \
  -p '{"data":{"app.properties":"feature.flag.beta=true\ncache.ttl.seconds=120\nwelcome.message=Hola, mundo actualizado\n"}}'
```

Espera ~30–60 segundos (el kubelet refresca los volúmenes proyectados con cierta latencia) y comprueba **desde dentro del pod**:

```bash
kubectl -n lab04 exec deploy/demo -- cat /etc/app/app.properties
```

Verás el contenido nuevo **sin reiniciar el pod**: los volúmenes desde ConfigMap se actualizan en vivo.

> El **bucle del script** sigue imprimiendo el `GREETING` viejo: las variables de entorno **se fijan al arrancar el contenedor** y no se refrescan.

## Paso 4 — Cambiar una variable de entorno (requiere reinicio)

Edita `GREETING`:

```bash
kubectl -n lab04 patch cm app-config --type=merge \
  -p '{"data":{"GREETING":"Hola tras rollout"}}'
```

Mira los logs: el bucle sigue diciendo `Hola desde ConfigMap`. Reinicia el Deployment:

```bash
kubectl -n lab04 rollout restart deploy/demo
kubectl -n lab04 rollout status deploy/demo
kubectl -n lab04 logs deploy/demo --tail=10
```

Ahora la línea del bucle dice `Hola tras rollout`. **Lección:** cuando cambies un ConfigMap/Secret consumido por `env`, **acuérdate del `rollout restart`**.

## Paso 5 — Rotar un Secret

Cambia el `API_TOKEN`:

```bash
kubectl -n lab04 patch secret app-secret --type=merge \
  -p '{"stringData":{"API_TOKEN":"nuevo-token"}}'
kubectl -n lab04 rollout restart deploy/demo
kubectl -n lab04 logs deploy/demo --tail=5
```

La app arranca y ahora imprime `API_TOKEN=nuevo-token`. Patrón estándar para rotar credenciales.

## Comprobaciones

- [ ] Has visto el mismo dato consumido por **env** y por **volumen**.
- [ ] Entiendes que un Secret en base64 **no es cifrado**.
- [ ] Confirmaste que **volumen sí, env no** se refrescan en caliente.
- [ ] Sabes ejecutar un `rollout restart` para propagar cambios sin tocar la imagen.

## Retos opcionales

1. **`envFrom`** — sustituye los tres `env` por:
   ```yaml
   envFrom:
     - configMapRef:
         name: app-config
     - secretRef:
         name: app-secret
   ```
   y comprueba que **todas** las claves del ConfigMap aparecen como variables.

2. **Marcar el ConfigMap como inmutable** (mejora rendimiento del kubelet):
   ```bash
   kubectl -n lab04 patch cm app-config --type=merge -p '{"immutable":true}'
   ```
   Intenta editarlo después y observa el error. Para cambiarlo habrá que **borrarlo y recrearlo** (con otro nombre o tras `delete`).

3. **Crear un ConfigMap desde fichero**:
   ```bash
   echo "from.file=true" > /tmp/extra.properties
   kubectl -n lab04 create cm extra --from-file=/tmp/extra.properties --dry-run=client -o yaml
   ```

## Troubleshooting

| Síntoma | Causa | Acción |
|---------|-------|--------|
| `MountVolume.SetUp failed for volume ... configmap "app-config" not found` | El ConfigMap aún no existe o está en otro namespace | Aplicar primero el ConfigMap; revisar `namespace`. |
| Cambio el ConfigMap y la variable no cambia | Es `env`, no volumen | `kubectl rollout restart`. |
| Cambio el ConfigMap (volumen) y no veo el cambio | Aún no ha pasado el ciclo de refresco | Espera 30–60 s, o `kubectl exec` y vuelve a leer el fichero. |
| `Secret` en `kubectl get secret -o yaml` muestra texto sin cifrar | Es base64, no cifrado | Para cifrado en reposo hay que activarlo en `kube-apiserver`. |

## Limpieza

```bash
kubectl delete namespace lab04
```

---

[← Anterior: Lab 3 — Diagnóstico](../lab-03-diagnostico/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente bloque: Kafka + Confluent →](../../bloque-2-kafka-confluent/fundamentos/README.md)
