# Tema 1 — Kafka operado en Kubernetes (síntesis)

[← Índice del bloque](README.md) · [Siguiente: Tema 2 — Pipeline extremo a extremo →](02-pipeline-extremo-a-extremo.md)

---

## Para qué este tema

Cerrar el círculo: poner una **única imagen mental** donde el grupo vea cómo lo que aprendieron en el bloque 1 (Kubernetes) se aplica a lo que aprendieron en el bloque 2 (Kafka). No introducimos conceptos nuevos; **consolidamos**.

## Idea clave en 30 segundos

> Un cluster Confluent en Kubernetes es **un caso particular de aplicación stateful**: brokers como **StatefulSet** con identidad y volumen propio, **Services** internos para el descubrimiento, **ConfigMaps y Secrets** para configuración y credenciales, **PersistentVolumeClaims** para los datos. Sobre eso, el operador **CFK** ofrece la lógica específica de Kafka (rolling updates seguros, gestión del quorum, certificados, escalado). Diagnosticar un broker no se diferencia esencialmente de diagnosticar cualquier otra app en K8s: **`describe pod`, eventos, logs, probes** — más el `describe topic` que ya conocemos.

## Desarrollo

### 1. La foto completa, pieza a pieza

Sobre un namespace de trabajo (típicamente `confluent` o `kafka`), CFK crea:

| Pieza de K8s | Para qué la usa Kafka | Visto en... |
|--------------|----------------------|-------------|
| **StatefulSet** | Brokers y controladores: identidad estable, orden de arranque, volumen propio. | Bloque 1, tema 5 |
| **Headless Service** | Resolución DNS por pod (`kafka-0.kafka...`). Imprescindible para que cada broker tenga un nombre único alcanzable. | Bloque 1, tema 5 |
| **ClusterIP Service** | Bootstrap del cliente (un único nombre como punto de entrada). | Bloque 1, tema 5 |
| **PVC + StorageClass** | Almacenamiento durable para el log de cada broker. | Repaso aquí |
| **ConfigMap** | Plantillas de configuración de Kafka y resto de componentes. | Bloque 1, tema 5 / LAB 4 |
| **Secret** | Credenciales, certificados, claves. | Bloque 1, tema 5 / LAB 4 |
| **Deployment** | Componentes stateless: Schema Registry, Connect, ksqlDB, Control Center, operador CFK. | Bloque 1, tema 5 |
| **CRDs (CFK)** | `Kafka`, `SchemaRegistry`, `Connect`, `KafkaTopic`, etc. | Bloque 2, tema 9 |

> **Talking point:** *"Si quitamos los CRDs específicos, lo demás es K8s estándar. CFK añade la lógica de orquestación que Kafka necesita por encima."*

### 2. ¿Por qué StatefulSet y no Deployment para los brokers?

Repaso de tres motivos que ya vimos:

- **Identidad estable.** El broker 1 es siempre el broker 1, con el mismo nombre DNS. Si fuera Deployment, los pods serían intercambiables: `app-7c9-xyz` hoy, `app-3a4-abc` mañana. Inservible para un cluster Kafka.
- **Volumen propio.** Cada broker tiene **su log** en un PVC dedicado. Si el pod cae, el nuevo se reasocia al mismo volumen y recupera el estado.
- **Orden de arranque.** Los brokers arrancan en orden (`-0`, `-1`, `-2`); útil para inicialización segura del cluster.

### 3. Servicios y exposición

El **descubrimiento interno** entre brokers usa un **headless Service**: cada broker se llama `kafka-0.kafka-internal.namespace.svc.cluster.local`. Esto se ve directamente con `kubectl get svc` y `kubectl exec` a un pod para hacer `nslookup`.

Para los **clientes** (productores y consumidores) hay normalmente:

- Un **bootstrap interno**: un Service tipo ClusterIP al que apuntan los clientes que viven dentro del clúster.
- Un **bootstrap externo**: opcional, según la organización (LoadBalancer, Ingress, NodePort) si hay clientes fuera del cluster K8s.

CFK simplifica todo esto: declaras los **listeners** que quieres y el operador crea los Services correspondientes.

### 4. Almacenamiento: la decisión que importa

El log de Kafka es **stateful**. Cada partición vive en disco. Conclusión: **la calidad del StorageClass importa**.

Aspectos a recordar:

- **Persistencia** real: el PVC debe sobrevivir a reinicios del pod.
- **Rendimiento**: SSD/NVMe local o cloud-native rápido. Discos lentos son veneno para un cluster Kafka serio.
- **No se "comparte"**: cada broker tiene su PVC. No es un volumen compartido tipo NFS para todos los brokers.
- **Reaccionar a `PVC pending`**: si tu StorageClass no aprovisiona, los brokers no arrancan. Es de las primeras cosas a mirar en una incidencia.

### 5. Diagnosticar un cluster Kafka en K8s: dos capas

Cuando algo va mal, hay que pensar en **dos planos**:

- **Plano Kubernetes** — ¿están los pods *Running*? ¿Hay *CrashLoopBackOff*? ¿*ImagePullBackOff*? ¿*Pending* por falta de recursos o PVC? ¿Las *probes* fallan? Esto se mira con `kubectl describe pod`, `kubectl logs`, `kubectl get events`.
- **Plano Kafka** — ¿están los brokers vivos según el cluster? ¿El ISR está sano? ¿Hay particiones *under-replicated*? ¿El quorum KRaft está bien? Esto se mira con `kafka-topics --describe`, `kafka-metadata-quorum`, métricas.

> **Talking point:** *"Un broker puede estar 'Running' en Kubernetes y 'fuera del ISR' en Kafka. Las dos cosas son verdad, y por eso miramos las dos capas."*

### 6. Operaciones cotidianas en este montaje

Algunas tareas habituales y dónde se hacen:

| Tarea | Capa | Cómo |
|-------|------|------|
| Levantar / bajar el cluster | K8s + CFK | Modificar CR `Kafka` (réplicas, recursos). |
| Crear un topic | Kafka (o CFK) | `kafka-topics --create ...` o CR `KafkaTopic`. |
| Cambiar retención de un topic | Kafka | `kafka-configs --alter ...`. |
| Ver consumer groups y lag | Kafka | `kafka-consumer-groups --describe ...`. |
| Reiniciar un broker | K8s | `kubectl delete pod kafka-N` (StatefulSet lo recrea). |
| Reemplazar un broker | K8s + CFK | Reescalar StatefulSet vía CR; CFK reasigna. |
| Rotar certificados | CFK | El operador gestiona la rotación al cambiar el secreto. |
| Ver logs de un broker | K8s | `kubectl logs -f kafka-0`. |

### 7. Lo que **no** cambia respecto a Kafka "tradicional"

Esto es importante para que el grupo no piense que K8s lo cambia todo:

- El protocolo de Kafka es el mismo.
- Los productores y consumidores **no saben ni les importa** que el cluster esté en K8s.
- El **bootstrap.servers** apunta a un nombre DNS: ya sea un Service interno en K8s o un balancer externo, para el cliente es transparente.
- Todo lo aprendido en bloque 2 sobre topics, particiones, ISR y consumer groups **se aplica idéntico**.

### 8. Connect, Schema Registry y ksqlDB en este escenario

Como aplicaciones **stateless** (en su mayoría), se despliegan con **Deployments** vía CRs específicos de CFK:

- **Schema Registry** — Deployment de varias réplicas; los datos viven en un topic Kafka, no en disco propio.
- **Connect** — Deployment de workers; los offsets de conectores viven en topics Kafka.
- **ksqlDB** — Deployment con almacenamiento local opcional (para estado de streams); si lo usa, va con PVCs.

Todos consumen el cluster Kafka como bootstrap, y Schema Registry como contrato. Es el mismo patrón que conocíamos, solo que cada cuadro del diagrama es un Deployment en K8s con su Service y su CR.

## Diagrama: el cluster completo en K8s

```mermaid
flowchart TB
    subgraph ns["Namespace confluent"]
        subgraph k["Kafka"]
            kc1["Pod controller-0"]
            kc2["Pod controller-1"]
            kc3["Pod controller-2"]
            kb1["Pod broker-0<br/>(PVC)"]
            kb2["Pod broker-1<br/>(PVC)"]
            kb3["Pod broker-2<br/>(PVC)"]
            ksvc["Service Kafka (bootstrap)"]
        end
        sr["Pod Schema Registry x2"]
        cc["Pod Connect worker x2"]
        kdb["Pod ksqlDB"]
        cfk["Pod operador CFK"]
        cm["ConfigMaps"]
        sec["Secrets / Certs"]
    end

    client["Productor/Consumidor<br/>(otro namespace o externo)"]

    client --> ksvc
    ksvc --> kb1
    ksvc --> kb2
    ksvc --> kb3
    sr --> ksvc
    cc --> ksvc
    kdb --> ksvc
    cfk -.gestiona.-> k
    cfk -.gestiona.-> sr
    cfk -.gestiona.-> cc
    cfk -.gestiona.-> kdb
```

## Errores típicos y preguntas frecuentes

- **"¿Un broker se puede mover de nodo K8s sin perder datos?"** Sí, porque el PVC es independiente del pod. K8s reprograma el pod en otro nodo y le re-adjunta el volumen. **Si la StorageClass no soporta esto** (ej. discos locales sin replicación), la respuesta es **no**: el broker se queda anclado a un nodo concreto.
- **"¿Puedo escalar brokers como un Deployment?"** Solo "para arriba" y con cuidado. Añadir brokers no balancea solo: hay que reasignar particiones (`kafka-reassign-partitions`). CFK ayuda con esto en versiones recientes.
- **"¿Pueden los productores estar fuera del clúster K8s?"** Sí. Hay que exponer un listener externo (LoadBalancer, Ingress con TCP passthrough). En CFK se configura en el CR `Kafka`.
- **"¿Los clientes en K8s reciben balanceo del Service?"** **Conceptualmente sí**, pero los clientes Kafka usan **conexión persistente y descubrimiento de líderes**; el Service balancea solo el primer contacto (bootstrap). Después, los clientes hablan directamente con los pods que sirven cada partición. Por eso necesitamos resolución DNS por pod (headless Service).
- **"¿Si reinicio el cluster K8s entero?"** Los volúmenes sobreviven; al recuperarse los pods, los brokers retoman el estado. El cluster Kafka vuelve a estar disponible. Hay que ser metódico: KRaft prefiere que arranquen primero suficientes controladores para tener quórum.

## Puente al siguiente tema

Tenemos la foto completa de la operación. Falta el ejercicio final: **construir y observar un pipeline real** Producer → Topic → Consumer dentro de este montaje. Eso lo plantea el siguiente tema, que prepara el LAB 14.

---

[← Índice del bloque](README.md) · [Siguiente: Tema 2 — Pipeline extremo a extremo →](02-pipeline-extremo-a-extremo.md)
