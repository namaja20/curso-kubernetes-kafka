# LAB 4 — ConfigMaps y Secrets

> **Has entrado aquí desde el [Capítulo 5 — Modelo de objetos y pods](../fundamentos/05-objetos-y-pods.md).** Al terminar, vuelve a ese capítulo. Es el último capítulo del bloque; tras él se cierra el Bloque 1 y arranca el Bloque 2 — Kafka + Confluent.

[← Volver: Capítulo 5 — Modelo de objetos y pods](../fundamentos/05-objetos-y-pods.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Separar **código** y **configuración**. Aprender a:

- Inyectar configuración como **variables de entorno** desde un ConfigMap y desde un Secret.
- Montar configuración como **archivo** dentro del contenedor (volumen).
- Modificar la configuración y propagar el cambio a los pods (`rollout restart`).

## Prerrequisitos

- Cluster `kind-curso` del [Lab 0](../lab-00-entorno/README.md) operativo.

## Organización de los manifiestos

```
lab-04-configuracion/
└── manifest/
    ├── 00-namespace.yaml
    └── apps/
        └── demo/                # nuestra app de ejemplo, "demo"
            ├── configmap.yaml
            ├── secret.yaml
            └── deployment.yaml  # consume configmap y secret
```

Una sola app (`demo`) que muestra **las tres formas** de consumir ConfigMaps y Secrets en un mismo Deployment.

## Conceptos rápidos

- **ConfigMap** — pares clave/valor (o ficheros) para configuración **no sensible**. Texto plano.
- **Secret** — igual, pero codificado en base64 y con tratamiento aparte (RBAC, *encryption at rest* si el cluster lo activa). **No** es cifrado por sí solo.
- **Tres formas de consumir** ambos:
  1. Como **variable de entorno** (`env.valueFrom.configMapKeyRef` / `secretKeyRef`).
  2. Como **bloque completo** de variables (`envFrom`).
  3. Como **fichero** dentro de un volumen (`volumes` + `volumeMounts`).
- **Propagación de cambios:** los *volúmenes* refrescan automáticamente (con un pequeño retardo); las *variables de entorno* **no** — necesitan reinicio del pod.

## Paso 1 — Crear el namespace, el ConfigMap y el Secret

### Los manifiestos

[`manifest/00-namespace.yaml`](manifest/00-namespace.yaml) — namespace `lab04`, sin sorpresas.

[`manifest/apps/demo/configmap.yaml`](manifest/apps/demo/configmap.yaml):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab04
data:
  GREETING: "Hola desde ConfigMap"     # clave simple → se consume como env
  LOG_LEVEL: "info"                    # idem
  app.properties: |                    # clave con extensión → se montará como fichero
    feature.flag.beta=false
    cache.ttl.seconds=60
    welcome.message=Bienvenido al curso
```

Detalles:

- Todas las claves de `data` son **strings**. Si necesitas binarios, usa `binaryData`.
- El nombre de la clave es el que se usa **luego** como variable de entorno o como nombre de fichero. `app.properties` se montará como `/etc/app/app.properties`.
- Un mismo ConfigMap puede contener variables y ficheros mezclados, como aquí.

[`manifest/apps/demo/secret.yaml`](manifest/apps/demo/secret.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: lab04
type: Opaque
stringData:                            # texto claro: Kubernetes lo codifica a base64 al guardarlo
  API_TOKEN: "s3cr3t-no-lo-cuentes"
  DB_PASSWORD: "p@ssw0rd-demo"
```

Detalles:

- **`stringData`** es la forma cómoda de escribir: lo declaras en claro y la API lo guarda en `data` codificado en base64. Usarlo en `kubectl apply` también funciona.
- **`type: Opaque`** es el genérico. Existen tipos especializados (`kubernetes.io/dockerconfigjson`, `kubernetes.io/tls`, etc.) con validaciones propias.
- **Base64 ≠ cifrado.** Cualquiera con permisos `get secret` lo decodifica trivialmente. El cifrado en reposo se activa aparte en `kube-apiserver`.

### Aplicarlos

```bash
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifest/00-namespace.yaml
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifest/apps/demo/configmap.yaml
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifest/apps/demo/secret.yaml
```

Inspecciónalo:

```bash
kubectl -n lab04 get cm app-config -o yaml
kubectl -n lab04 get secret app-secret -o yaml
```

Decodificar el secret en claro (sirve para verificar y para incidencias):

```bash
kubectl -n lab04 get secret app-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d; echo
```

## Paso 2 — Desplegar la app que consume ambos

### El manifiesto

[`manifest/apps/demo/deployment.yaml`](manifest/apps/demo/deployment.yaml). El contenedor es un `busybox` que ejecuta un script que **imprime** el estado actual y luego entra en un bucle, para poder observar fácilmente los cambios.

Bloque por bloque:

```yaml
    spec:
      containers:
        - name: demo
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - |
              echo "==== ENV ===="
              echo "GREETING=$GREETING"
              echo "LOG_LEVEL=$LOG_LEVEL"
              echo "API_TOKEN=$API_TOKEN"
              echo "==== /etc/app/app.properties ===="
              cat /etc/app/app.properties
              echo "==== loop ===="
              while true; do
                echo "[$(date +%T)] alive, GREETING=$GREETING"
                sleep 10
              done
```

El script imprime una vez **el estado inicial** y luego repite cada 10 s el valor de `$GREETING`. Como las variables de entorno **se fijan al arrancar el contenedor**, el bucle nos servirá para detectar si un cambio en el ConfigMap llega o no sin reinicio.

```yaml
          env:                               # forma 1: variables sueltas, una a una
            - name: GREETING
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: GREETING
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: API_TOKEN
```

Cada `env` toma **una clave concreta** del ConfigMap o Secret. Ventaja: control fino y nombres distintos a las claves originales si los necesitas (`name` puede diferir de `key`).

```yaml
          volumeMounts:                      # forma 3: como fichero dentro del contenedor
            - name: app-config
              mountPath: /etc/app
              readOnly: true
...
      volumes:
        - name: app-config
          configMap:
            name: app-config
            items:                           # proyecta SOLO la clave app.properties
              - key: app.properties
                path: app.properties         # → /etc/app/app.properties
```

Aquí montamos un *volumen proyectado* desde el ConfigMap. La cláusula `items` evita montar todas las claves: solo nos interesa el fichero `app.properties`. El kubelet refrescará este volumen automáticamente cuando el ConfigMap cambie.

> El manifiesto no usa **`envFrom`** (forma 2), que cargaría de golpe todas las claves de un ConfigMap o Secret como variables. La verás en los retos opcionales.

### Aplicarlo

```bash
kubectl apply -f bloque-1-kubernetes/lab-04-configuracion/manifest/apps/demo/deployment.yaml
kubectl -n lab04 rollout status deploy/demo
```

Mira la salida inicial:

```bash
kubectl -n lab04 logs deploy/demo --tail=40
```

Deberías ver:

```
==== ENV ====
GREETING=Hola desde ConfigMap
LOG_LEVEL=info
API_TOKEN=s3cr3t-no-lo-cuentes
==== /etc/app/app.properties ====
feature.flag.beta=false
cache.ttl.seconds=60
welcome.message=Bienvenido al curso
==== loop ====
[12:34:56] alive, GREETING=Hola desde ConfigMap
```

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

> El **bucle del script** sigue imprimiendo el `GREETING` viejo: las variables de entorno **se fijan al arrancar el contenedor** y no se refrescan. Ese es justamente el punto del Paso 4.

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

Ahora la línea del bucle dice `Hola tras rollout`.

**Lección:** cuando cambies un ConfigMap/Secret consumido por `env`, **acuérdate del `rollout restart`**.

## Paso 5 — Rotar un Secret

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
   y comprueba que **todas** las claves del ConfigMap aparecen como variables (incluido `app.properties`, que como variable no tiene sentido — sirve de aviso).

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
| `Secret` en `kubectl get secret -o yaml` muestra texto "casi en claro" | Es base64, no cifrado | Para cifrado en reposo hay que activarlo en `kube-apiserver`. |

## Limpieza

```bash
kubectl delete namespace lab04
```

## Vuelve a la teoría

Has completado el laboratorio. Regresa al capítulo desde el que llegaste y cierra el bloque:

[← Volver al **Capítulo 5 — Modelo de objetos y pods**](../fundamentos/05-objetos-y-pods.md)

Al terminarlo, has completado el Bloque 1. El siguiente paso es:

[Bloque 2 — Kafka + Confluent →](../../bloque-2-kafka-confluent/fundamentos/README.md)

---

[← Volver: Capítulo 5 — Modelo de objetos y pods](../fundamentos/05-objetos-y-pods.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente bloque: Kafka + Confluent →](../../bloque-2-kafka-confluent/fundamentos/README.md)
