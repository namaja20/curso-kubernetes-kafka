# Tema 4 — Replicación, ISR y tolerancia a fallos

[← Anterior: Tema 3 — Consumer groups y offsets](03-consumer-groups-offsets.md) · [Índice del bloque ↑](README.md) · [Siguiente: Tema 5 — KRaft →](05-kraft.md)

---

## Para qué este tema

Explicar el mecanismo por el que Kafka **no pierde mensajes cuando un broker se cae** y, sobre todo, qué quiere decir cada palabra que aparece después en `describe topic`: *leader*, *replicas*, *ISR*, *under-replicated*. Es el tema con más sustancia operativa del bloque: la mayoría de las incidencias serias en Kafka se diagnostican aquí.

## Idea clave en 30 segundos

> Cada partición se **replica** en varios brokers según el `replication.factor`. Uno de ellos es el **líder** y sirve todas las lecturas y escrituras; los demás son **seguidores** que copian al líder. El conjunto de réplicas que están **al día** se llama **ISR (In-Sync Replicas)**. Las escrituras se consideran durables cuando un mínimo de ISR (`min.insync.replicas`) las han recibido. Si el líder se cae, el controlador elige un **nuevo líder de entre los que están en ISR**. Si no quedan ISR suficientes, las escrituras se bloquean para preservar la durabilidad.

## Desarrollo

### 1. ¿Qué se replica?

**Particiones**, no topics. Si un topic tiene 3 particiones y factor de replicación 3, **cada partición** existe en 3 brokers (1 líder + 2 seguidores). El reparto se hace de forma que no haya dos réplicas de la misma partición en el mismo broker (no tendría sentido).

Ejemplo intuitivo en un cluster de 3 brokers, topic con 3 particiones y RF=3:

| Partición | Líder | Seguidores |
|-----------|-------|------------|
| 0 | broker-0 | broker-1, broker-2 |
| 1 | broker-1 | broker-0, broker-2 |
| 2 | broker-2 | broker-0, broker-1 |

Los líderes están repartidos para que **ningún broker concentre toda la carga**.

### 2. Cómo se escribe (el camino de un mensaje)

1. El productor descubre quién es el **líder** de la partición destino (vía metadatos del cluster).
2. Envía el mensaje **solo al líder**.
3. El líder lo añade a su log.
4. Los seguidores **piden** al líder los datos nuevos y los replican en sus propios logs.
5. Cuando suficientes réplicas confirman, el líder responde al productor con el ACK.

El "suficientes" depende del parámetro `acks` del productor:

| `acks` | Significado | Garantía |
|--------|-------------|----------|
| `0` | El productor no espera respuesta | Máxima velocidad, **puede perder datos** |
| `1` | Espera ACK solo del líder | Si el líder se cae antes de replicar, se pierde |
| `all` (o `-1`) | Espera ACK del líder + del mínimo ISR | **Garantía de durabilidad real** |

> **Talking point:** *"En producción seria, `acks=all` y `min.insync.replicas=2` con factor 3 es el patrón estándar. Cualquier cosa distinta debe justificarse explícitamente."*

### 3. ¿Qué es exactamente el ISR?

**ISR (In-Sync Replicas)** es el conjunto de réplicas (incluido el líder) que están **suficientemente al día** con el líder. "Suficientemente al día" significa que han hecho fetch reciente y no se han quedado atrás más de un umbral (`replica.lag.time.max.ms`, por defecto 30 s en Apache Kafka).

Tres detalles importantes:

- El **líder siempre está en ISR** de su propia partición.
- Un seguidor que se queda muy retrasado **se cae del ISR** automáticamente.
- Si más tarde alcanza al líder, **vuelve al ISR**.

### 4. `min.insync.replicas`: el seguro de durabilidad

Es una propiedad del **topic** (también puede ser global). Si el productor escribe con `acks=all`:

- Kafka acepta la escritura **solo si hay al menos `min.insync.replicas` en el ISR**.
- Si no hay suficientes, devuelve error `NOT_ENOUGH_REPLICAS` al productor.

Lo que esto significa:

- Con RF=3 y `min.insync.replicas=2`: tolera la caída de **1 broker** sin afectar a la escritura. Si caen 2, las escrituras se **bloquean** (mejor parar que aceptar mensajes que pueden perderse).
- Con RF=3 y `min.insync.replicas=1`: tolera la caída de 2, **pero sacrifica garantía**: la escritura con un único nodo vivo puede perderse si ese nodo muere antes de replicarla.

> **Talking point:** *"El sistema prefiere caerse antes que aceptar datos que no puede prometer. Esto suele sorprender: 'mi productor no puede escribir, pero el cluster no está caído del todo'. Es así por diseño."*

### 5. Elección de líder cuando un líder se cae

Cuando el broker que es líder de una partición desaparece:

1. El **controlador** del cluster (lo veremos en el tema de KRaft) detecta la baja.
2. Selecciona un **nuevo líder de entre las réplicas que estén en ISR**.
3. Notifica al resto de brokers y a los clientes mediante actualización de metadatos.
4. Los productores y consumidores reanudan operación contra el nuevo líder en cuestión de segundos.

Si **no queda ninguna réplica en ISR**, hay dos caminos posibles:

- **Por defecto (seguro):** la partición queda **sin líder** y no acepta escrituras hasta que vuelva una réplica en ISR.
- **Unclean leader election** (`unclean.leader.election.enable=true`): elegir un seguidor **fuera de ISR** como nuevo líder. Es **rápido pero peligroso**: el nuevo líder puede no tener los últimos mensajes y se **perderán datos**. Está **deshabilitado por defecto** y conviene mantenerlo así salvo casos muy específicos.

> **Pregunta al aula:** *"¿Preferís perder algunos mensajes y seguir escribiendo, o no perderlos y bloquear escrituras hasta recuperar? Esa decisión es `unclean.leader.election`."*

### 6. `under-replicated` y otros indicadores

Cuando hagas `kafka-topics --describe` verás cosas como:

```
Topic: pedidos  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
Topic: pedidos  Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3
Topic: pedidos  Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,2,1
```

La **partición 1** tiene `Isr: 2,3` pero `Replicas: 2,3,1` — la réplica 1 **no está en sync**. Eso es una partición **"under-replicated"**.

Causas habituales:

- Un broker está caído.
- Un broker está saturado y no consigue replicar al ritmo del líder.
- La red entre brokers está degradada.
- Configuración demasiado agresiva (latencias altas, particiones grandes, replicación lenta).

La métrica `UnderReplicatedPartitions > 0` durante un tiempo es **una alerta operativa importante**.

### 7. Cómo se relaciona esto con el LAB 9

En el laboratorio se va a:

1. Crear un topic con `replication.factor=3`.
2. Producir mensajes y comprobar el ISR.
3. **Matar deliberadamente un broker** (en nuestro caso, el pod).
4. Observar:
   - Cómo cambian los líderes.
   - Cómo el ISR pasa a 2.
   - Cómo el productor (con `acks=all`, `min.insync.replicas=2`) **sigue funcionando**.
   - Qué ocurre si se mata un segundo broker (escrituras bloqueadas).
5. Recuperar el broker y observar la **reincorporación al ISR**.

Después de este laboratorio, ningún participante puede confundir RF con `min.insync.replicas`.

### 8. Otros parámetros prácticos relacionados

- `replica.lag.time.max.ms` — umbral para considerar a un seguidor fuera de sync.
- `default.replication.factor` — factor por defecto de los topics creados.
- `auto.create.topics.enable` — si está activo (no recomendado en prod), los topics se crean con valores por defecto, **que muchas veces no son los correctos**.

## Diagrama: líder, seguidores e ISR

```mermaid
flowchart LR
    P["Productor (acks=all)"] --> L["Líder<br/>(broker-1)"]
    L --> F1["Seguidor<br/>(broker-2)"]
    L --> F2["Seguidor<br/>(broker-3)"]
    F1 -. "en sync" .-> ISR
    F2 -. "en sync" .-> ISR
    L -. "siempre en ISR" .-> ISR(("ISR"))
```

> Solo el líder dialoga con productores/consumidores. Los seguidores hacen *pull* del líder. La pertenencia al ISR depende del retardo. `min.insync.replicas` define cuántos miembros del ISR necesita la escritura.

## Errores típicos y preguntas frecuentes

- **"¿Por qué mis productores fallan si solo hay un broker caído?"** Probablemente tienes `acks=all` con `min.insync.replicas=3` y RF=3. Pierdes uno y ya no llegas al mínimo. Lo habitual es **RF=3, `min.insync.replicas=2`**.
- **"¿Las lecturas también pasan por el líder?"** Sí (con configuración estándar). Hay funciones avanzadas (*follower fetching*) para leer desde réplicas cercanas, pero no las trataremos.
- **"¿Cuánto tarda en elegirse nuevo líder?"** Segundos. Lo notarás en el throughput durante el fallo; raramente es un parón largo si el cluster está sano.
- **"¿Replicación = backup?"** **No.** Replicación protege contra **fallos de broker**, no contra **errores humanos** (borrar un topic) ni contra **corrupción de datos**. Backups siguen siendo necesarios en producción.
- **"¿Puedo cambiar la replicación de un topic existente?"** Sí, con `kafka-reassign-partitions`. No es trivial: hay que generar plan, ejecutarlo y verificarlo. Operación habitual en migraciones.

## Puente al siguiente tema

Hemos visto qué hace el cluster para no perder datos. Falta un componente vital: **quién decide quién es líder, quién entra y sale del ISR y quién está vivo**. Históricamente lo hacía **ZooKeeper**. En Kafka moderno lo hace **el propio cluster** mediante **KRaft**. Ese es el siguiente tema.

---

[← Anterior: Tema 3 — Consumer groups y offsets](03-consumer-groups-offsets.md) · [Índice del bloque ↑](README.md) · [Siguiente: Tema 5 — KRaft →](05-kraft.md)
