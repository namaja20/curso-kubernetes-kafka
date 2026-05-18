# LAB 5 — Flujo básico de datos en Kafka

> **Llegada desde el [Capítulo 1 — Event streaming y modelo Kafka](../fundamentos/01-event-streaming.md).** Al terminar, regresa a ese capítulo y continúa la lectura hasta el siguiente laboratorio.

[← Volver: Capítulo 1 — Event streaming](../fundamentos/01-event-streaming.md) · [Índice del bloque ↑](../fundamentos/README.md)

---

## Objetivo

Levantar un **cluster Kafka de 3 brokers en modo KRaft** sobre el cluster Kubernetes del [Lab 0](../../bloque-1-kubernetes/lab-00-entorno/README.md), desplegar el pod **`kafka-cli`** (rol CLI) y validar el flujo completo de mensajes desde ese pod.

Al terminar este laboratorio existe en el cluster:

- Un **namespace de plataforma** (`kafka`) con un StatefulSet de **3 brokers** (`kafka-0` … `kafka-2`) y sus Services.
- Un **namespace de inquilino** (`app-a`) con el Deployment **`kafka-cli`**: pod con rol **solo CLI** (imagen `confluentinc/cp-kafka`, sin proceso broker; `sleep infinity` + `kubectl exec`).
- Un primer **topic** (`eventos`) operativo y persistido, con mensajes leídos y releídos para comprobar que el log no se vacía al consumir.

➡️ Patrón documentado: [Entorno de práctica: pod kafka-cli](../docs/entorno-practica-kafka-cli.md)

Este cluster y el pod **`kafka-cli`** son la **base de los laboratorios 6 a 12**: no se vuelven a desplegar; en cada lab las demos se lanzan con `kubectl -n app-a exec deploy/kafka-cli -- …`.

## Perfil de quien hace este lab

Los laboratorios del bloque 2 están pensados para el **rol de operador de plataforma**: quien despliega y mantiene Kafka en Kubernetes para que los equipos de aplicación lo consuman desde sus propios namespaces. No se programa: se opera con `kubectl` y, para las demos de Kafka, con comandos `kafka-*` ejecutados **en el pod `kafka-cli`** (imagen `confluentinc/cp-kafka` usada solo como contenedor de herramientas).

## Referencia de comandos

Hoja de comandos CLI contra el cluster: [Cheatsheet — Comandos Kafka](../docs/cheatsheet-comandos-kafka.md).

## Prerrequisitos

- Cluster `kind-curso` activo (Lab 0). Comprueba: `kubectl get nodes` debe listar 3 nodos en `Ready`.
- ~3 GiB de RAM libres en el host del cluster (Codespace o equipo local). Los 3 brokers consumen aproximadamente 750 MiB cada uno.
- Conocimientos previos del Bloque 1: **Deployments**, **StatefulSets** (introducidos por encima en el cap 05), **Services**, **Namespaces**, **PVCs**.

## Conceptos clave para esta operación

- **Cluster Kafka en KRaft**. Cada broker actúa además como **controller** (modo combinado). El quorum de controllers reemplaza a ZooKeeper para almacenar los metadatos del cluster.
- **StatefulSet**, no Deployment. Cada broker necesita una **identidad estable** (nombre DNS y volumen propio). Eso solo lo garantiza un StatefulSet.
- **Headless Service** (`kafka-headless`) vs **Service ClusterIP** (`kafka`). El primero da **una IP por pod** (DNS `kafka-0.kafka-headless.kafka.svc.cluster.local`), imprescindible para que los brokers se vean entre sí. El segundo es el **punto de entrada único** para los clientes (`bootstrap.servers`).
- **Multi-tenant por namespace**. La plataforma vive en `kafka`; los equipos de aplicación viven en sus namespaces (`app-a`, `app-b`, …). El operador de plataforma gestiona el cluster; los inquilinos solo consumen.
- **`bootstrap.servers`** no es una dirección de un broker concreto: es un punto de entrada que el cliente usa una sola vez para descubrir el resto.

## Organización de los manifiestos

```
lab-05-flujo-basico/
└── manifest/
    ├── 00-namespaces.yaml              # plataforma + inquilino
    └── apps/
        ├── kafka-cluster/              # responsabilidad del operador de plataforma
        │   ├── service-headless.yaml
        │   ├── service-bootstrap.yaml
        │   └── statefulset.yaml
        └── client/                     # pod kafka-cli (rol CLI, no broker)
            └── deployment.yaml         # Deployment kafka-cli en app-a
```

## Paso 1 — Namespaces (plataforma + inquilino)

### El manifiesto

[`manifest/00-namespaces.yaml`](manifest/00-namespaces.yaml):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka                       # namespace de la plataforma Kafka
  labels:
    role: platform                  # útil para políticas y para distinguir cluster vs tenants
---
apiVersion: v1
kind: Namespace
metadata:
  name: app-a                       # namespace del equipo "app-a" (inquilino de ejemplo)
  labels:
    role: tenant
```

La etiqueta `role` no la usa Kubernetes por sí misma; sirve para que más adelante se puedan aplicar **NetworkPolicies** y **RBAC** distintos a *plataforma* vs *inquilinos*. En este lab solo simboliza la separación.

### Aplicarlo

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/00-namespaces.yaml
kubectl get ns kafka app-a
```

## Paso 2 — Headless Service del cluster Kafka

### El manifiesto

[`manifest/apps/kafka-cluster/service-headless.yaml`](manifest/apps/kafka-cluster/service-headless.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: kafka
spec:
  clusterIP: None                   # SIN ClusterIP → "headless"
  selector:
    app: kafka
  ports:
    - { name: client,     port: 9092, targetPort: 9092 }
    - { name: controller, port: 9093, targetPort: 9093 }
  publishNotReadyAddresses: true    # imprescindible al arrancar el quorum (todos a la vez)
```

Puntos clave:

- **`clusterIP: None`** convierte el Service en *headless*. En vez de un único IP virtual, el DNS interno publica **un registro A por pod**: `kafka-0.kafka-headless.kafka.svc.cluster.local`, `kafka-1...`, `kafka-2...`. Ese nombre fijo es lo que cada broker necesita para que los demás lo encuentren.
- **`publishNotReadyAddresses: true`**. Al arrancar el cluster por primera vez, los tres brokers están "no ready" mientras se eligen entre sí como controllers. Sin esta opción, ningún broker resolvería al resto y nadie alcanzaría jamás `Ready` (deadlock).

### Aplicarlo

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/kafka-cluster/service-headless.yaml
```

## Paso 3 — Service de bootstrap

### El manifiesto

[`manifest/apps/kafka-cluster/service-bootstrap.yaml`](manifest/apps/kafka-cluster/service-bootstrap.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka                       # FQDN: kafka.kafka.svc.cluster.local
  namespace: kafka
spec:
  type: ClusterIP                   # IP virtual + load balancing entre brokers
  selector:
    app: kafka
  ports:
    - { name: client, port: 9092, targetPort: 9092 }
```

Diferencias respecto al headless:

- **Tiene ClusterIP**, así que actúa de punto de entrada único. El cliente solo necesita conocer `kafka.kafka.svc.cluster.local:9092` para arrancar; el Service lo reparte hacia cualquiera de los 3 brokers.
- **No es para el tráfico de datos a una partición concreta**, solo para el primer contacto (`bootstrap`). Tras la respuesta inicial, el cliente recibe los `advertised.listeners` reales de cada broker y se conecta directamente a quien corresponda.

### Aplicarlo

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/kafka-cluster/service-bootstrap.yaml
```

## Paso 4 — StatefulSet de 3 brokers (KRaft, modo combinado)

### El manifiesto

[`manifest/apps/kafka-cluster/statefulset.yaml`](manifest/apps/kafka-cluster/statefulset.yaml). Lo más relevante, bloque a bloque:

```yaml
spec:
  serviceName: kafka-headless       # nombre del headless service → DNS estable
  replicas: 3
  podManagementPolicy: Parallel     # crea los 3 pods a la vez (necesario para que se elijan)
```

`podManagementPolicy: Parallel` es deliberado: con la política por defecto (`OrderedReady`), Kubernetes esperaría a que `kafka-0` esté `Ready` para crear `kafka-1`. Pero `kafka-0` no estará `Ready` hasta que vea al menos a otro controller votar. Resultado: deadlock. Con `Parallel`, los tres arrancan a la vez y se eligen entre sí.

```yaml
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.7.1
          command:
            - sh
            - -c
            - |
              ORDINAL="${POD_NAME##*-}"           # kafka-0 -> 0, kafka-1 -> 1, kafka-2 -> 2
              export KAFKA_NODE_ID="${ORDINAL}"
              export KAFKA_ADVERTISED_LISTENERS="PLAINTEXT://${POD_NAME}.kafka-headless.kafka.svc.cluster.local:9092"
              exec /etc/confluent/docker/run
```

El `command` calcula dos cosas que **no pueden ser estáticas** porque dependen del pod concreto:

- **`KAFKA_NODE_ID`** — id único por broker, derivado del ordinal del pod.
- **`KAFKA_ADVERTISED_LISTENERS`** — el nombre con el que el broker se anuncia a los clientes. Tiene que ser el FQDN de **este pod** en el headless service para que los clientes lleguen al líder de cada partición.

```yaml
          env:
            - name: CLUSTER_ID                            # mismo en los 3 brokers
              value: "MkU3OEVBNTcwNTJENDM2Qk"             # cualquier UUID base64 de 22 chars
            - name: KAFKA_PROCESS_ROLES
              value: "broker,controller"                  # modo combinado: cada pod hace ambas funciones
            - name: KAFKA_CONTROLLER_QUORUM_VOTERS
              value: "0@kafka-0.kafka-headless.kafka.svc.cluster.local:9093,\
                      1@kafka-1.kafka-headless.kafka.svc.cluster.local:9093,\
                      2@kafka-2.kafka-headless.kafka.svc.cluster.local:9093"
```

Tres ideas importantes para el operador:

- **`CLUSTER_ID`** es el identificador del cluster. Una vez formateado el almacenamiento con ese ID, no se puede cambiar. Si cambia, los brokers no arrancan.
- **`KAFKA_PROCESS_ROLES: "broker,controller"`** es el **modo combinado**. En producción seria se separa (3 controllers dedicados + N brokers); para el curso es suficiente. KRaft sustituye totalmente a ZooKeeper.
- **`KAFKA_CONTROLLER_QUORUM_VOTERS`** es estático y conocido de antemano. Por eso el lab usa los nombres del headless service: son DNS estables que no cambian aunque el pod se reinicie.

```yaml
            - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
              value: "3"
            - name: KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR
              value: "3"
            - name: KAFKA_TRANSACTION_STATE_LOG_MIN_ISR
              value: "2"
            - name: KAFKA_AUTO_CREATE_TOPICS_ENABLE
              value: "false"
```

- Los **topics internos** (`__consumer_offsets`, `__transaction_state`) se replican con factor 3. Si el operador no fija esto, Kafka usa 1 por defecto y el cluster pierde HA al primer reinicio.
- **`auto.create.topics.enable: false`** — política estándar de operador: los topics se crean **explícitamente** (después con CI/CD, en este lab a mano). Evita que cualquier productor genere topics fantasma con cualquier configuración.

```yaml
          volumeMounts:
            - { name: data, mountPath: /var/lib/kafka/data }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 1Gi } }
```

Cada broker pide su **propio PVC** de 1 GiB. En kind se atiende con el `local-path-provisioner` por defecto. En producción esto correspondería a un `StorageClass` con disco rápido (NVMe/SSD locales o EBS gp3).

### Aplicarlo y observar el arranque

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/kafka-cluster/statefulset.yaml

kubectl -n kafka get pods -w
```

Verás los 3 pods pasar por `Pending` → `ContainerCreating` → `Running` → `Ready`. Tarda 30–90 s en total: el primero en arrancar espera a que el quorum se forme.

Cuando los 3 estén `1/1 Ready`:

```bash
kubectl -n kafka get pods,svc,pvc
```

## Paso 5 — Pod `kafka-cli` (rol CLI en el inquilino `app-a`)

Este paso despliega el **pod de operación** del bloque: no es un broker, es un contenedor con la misma imagen Confluent solo para ejecutar comandos `kafka-*` vía `kubectl exec`. Todas las demos de topics, produce/consume y consumer groups de los Labs 5–10 parten de aquí.

### El manifiesto

[`manifest/apps/client/deployment.yaml`](manifest/apps/client/deployment.yaml) crea un ConfigMap con las propiedades del cliente y el Deployment **`kafka-cli`** en `app-a`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-client-config
  namespace: app-a
data:
  client.properties: |
    bootstrap.servers=kafka.kafka.svc.cluster.local:9092
    security.protocol=PLAINTEXT
    client.id=app-a-client
```

El **inquilino conoce únicamente el bootstrap**, no las direcciones individuales de los brokers. Eso es exactamente lo que un equipo de aplicación recibe del operador de plataforma en cualquier entorno real: un `bootstrap.servers` y, si aplica, credenciales.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-cli
  namespace: app-a
spec:
  template:
    spec:
      containers:
        - name: cli
          image: confluentinc/cp-kafka:7.7.1     # la misma imagen ya trae las CLIs
          command: ["sleep", "infinity"]         # solo nos interesa exec-ar a él
          volumeMounts:
            - { name: config, mountPath: /etc/kafka-client, readOnly: true }
      volumes:
        - name: config
          configMap: { name: kafka-client-config }
```

El pod **no arranca Kafka**: `command: ["sleep", "infinity"]`. Los binarios (`kafka-topics`, `kafka-console-producer`, …) ya están en el `PATH` de la imagen `cp-kafka`. El operador entra con `kubectl exec` y lanza las demos; no hace falta instalar nada en el Codespace salvo `kubectl`.

### Aplicarlo

```bash
kubectl apply -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/client/deployment.yaml
kubectl -n app-a rollout status deploy/kafka-cli
```

## Paso 6 — Crear un topic desde el inquilino

Abre una shell dentro del pod cliente:

```bash
kubectl -n app-a exec -it deploy/kafka-cli -- bash
```

Dentro del pod, comprueba que llegas al cluster:

```bash
kafka-broker-api-versions --bootstrap-server kafka.kafka.svc.cluster.local:9092 | head -5
```

Crea el topic `eventos` con 3 particiones y replication-factor 3:

```bash
kafka-topics --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --create --topic eventos \
  --partitions 3 --replication-factor 3
```

Descríbelo:

```bash
kafka-topics --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --describe --topic eventos
```

La salida muestra para cada partición su **líder**, sus **réplicas** y la lista de **ISR** (réplicas en sync). Como `replication-factor 3`, cada partición vive en los 3 brokers.

## Paso 7 — Producir mensajes

En la misma shell del pod cliente:

```bash
kafka-console-producer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic eventos
```

Escribe varias líneas (cada línea es un mensaje) y termina con `Ctrl+D`:

```
pedido-1 creado
pedido-2 confirmado
pedido-3 cancelado
```

## Paso 8 — Consumir desde el principio (persistencia)

```bash
kafka-console-consumer --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
  --topic eventos --from-beginning --timeout-ms 5000
```

Verás los tres mensajes en pantalla. El **flag `--from-beginning`** es lo que demuestra que Kafka **no es una cola**: los mensajes siguen en el log, y un consumidor que arranca de cero los puede releer.

Ejecuta el mismo `kafka-console-consumer` **otra vez**. Vuelven a aparecer los mismos mensajes. Eso es persistencia y *fan-out* — el corazón conceptual del Bloque 2.

Sal del pod con `exit`.

## Comprobaciones

- [ ] `kubectl -n kafka get pods` muestra `kafka-0`, `kafka-1`, `kafka-2` en `1/1 Ready`.
- [ ] `kubectl -n kafka get pvc` lista 3 PVCs `data-kafka-{0,1,2}` en `Bound`.
- [ ] `kafka-topics --describe --topic eventos` muestra 3 particiones con 3 réplicas e ISR completo.
- [ ] El consumer re-ejecutado con `--from-beginning` vuelve a leer los mismos mensajes.

## Retos opcionales (operacionales)

1. **Inspeccionar los topics internos del cluster**:
   ```bash
   kafka-topics --bootstrap-server kafka.kafka.svc.cluster.local:9092 --list
   ```
   Aparecerán `__consumer_offsets` y, si has tocado transacciones, `__transaction_state`. Son los topics que Kafka usa para sí mismo.

2. **Ver el log en disco de un broker**:
   ```bash
   kubectl -n kafka exec kafka-0 -- ls -la /var/lib/kafka/data
   kubectl -n kafka exec kafka-0 -- ls /var/lib/kafka/data | grep eventos
   ```
   Verás un directorio por partición presente en ese broker. Las particiones son ficheros reales.

3. **Anticipo del Lab 9 — Disaster recovery básico**: borra el pod `kafka-1` y observa que el cluster sigue operando:
   ```bash
   kubectl -n kafka delete pod kafka-1
   kafka-topics --bootstrap-server kafka.kafka.svc.cluster.local:9092 --describe --topic eventos
   ```
   En el `--describe` verás que algunas particiones tienen ISR reducido durante unos segundos. El StatefulSet recrea `kafka-1`, vuelve a unirse al quorum y la ISR se restaura. **Cero downtime para el cliente.**

## Troubleshooting

| Síntoma | Causa habitual | Acción |
|---------|----------------|--------|
| `kafka-0` se queda en `Running` pero `0/1 Ready` durante minutos | Quorum no se forma | `kubectl -n kafka logs kafka-0` y verifica que `KAFKA_CONTROLLER_QUORUM_VOTERS` y el headless service están bien. Suele faltar `publishNotReadyAddresses: true`. |
| Pods en `Pending` | Falta de CPU/RAM en el cluster kind | `kubectl describe pod` lo dice. Reduce `KAFKA_HEAP_OPTS` o el número de réplicas (acotando luego replication-factor). |
| `kafka-topics ... --create` da `LEADER_NOT_AVAILABLE` | Los brokers no son alcanzables todavía | Espera a que los 3 pods estén `Ready` y reintenta. |
| PVCs `Pending` | StorageClass por defecto no presente | `kubectl get sc` debe mostrar `standard (default)` (kind lo provee). |
| Cluster ID mismatch tras reinicio del StatefulSet | El PVC conserva un ID y el env trae otro | No cambies `CLUSTER_ID` salvo recreando los PVCs. |

## Vuelve a la teoría

Has cubierto el flujo conceptual completo. Regresa al capítulo desde el que llegaste y continúa la lectura:

[← Volver al **Capítulo 1 — Event streaming y modelo Kafka**](../fundamentos/01-event-streaming.md)

Cuando lo termines, el siguiente capítulo de teoría es:

[Capítulo 2 — Brokers, topics y particiones →](../fundamentos/02-brokers-topics-particiones.md)

## Limpieza

**Conserva este cluster.** Los Labs 6, 7, 8 y 9 lo reutilizan tal cual; ahorra mucho tiempo no redesplegarlo. Si necesitas liberar memoria:

```bash
kubectl delete -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/client/deployment.yaml
kubectl delete -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/apps/kafka-cluster/
kubectl -n kafka delete pvc -l app=kafka
kubectl delete -f bloque-2-kafka-confluent/lab-05-flujo-basico/manifest/00-namespaces.yaml
```

(El borrado del PVC libera el disco; el StatefulSet por sí solo no lo hace, deliberadamente, para evitar pérdida de datos accidental.)

---

[← Volver: Capítulo 1 — Event streaming](../fundamentos/01-event-streaming.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente capítulo: 2 — Brokers, topics y particiones →](../fundamentos/02-brokers-topics-particiones.md)
