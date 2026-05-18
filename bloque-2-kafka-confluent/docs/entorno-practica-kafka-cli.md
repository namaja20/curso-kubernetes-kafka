# Entorno de práctica — pod `kafka-cli`

En los laboratorios del Bloque 2 **no se instala Kafka en el portátil del alumno** ni se construyen imágenes propias a partir de paquetes Apache. Se usa la imagen oficial de Confluent **`confluentinc/cp-kafka`** en dos roles distintos dentro del cluster Kubernetes.

## Dos roles, misma imagen

| Rol | Recurso en K8s | Namespace | Qué hace |
|-----|----------------|-----------|----------|
| **Broker** | StatefulSet `kafka` (pods `kafka-0`, `kafka-1`, `kafka-2`) | `kafka` | Ejecuta el proceso Kafka (KRaft: broker + controller). Arranque: `/etc/confluent/docker/run`. |
| **CLI de operación** | Deployment `kafka-cli` | `app-a` (inquilino de ejemplo) | **No arranca ningún broker.** Comando: `sleep infinity`. Solo sirve para `kubectl exec` y lanzar binarios `kafka-*` ya incluidos en la imagen. |

La imagen `cp-kafka` incluye en `/usr/bin/` las herramientas de línea de comandos de Kafka (`kafka-topics`, `kafka-consumer-groups`, `kafka-console-producer`, etc.). **No hace falta un pod ni una imagen adicional** para las demos de topics, consumidores, offsets o lag.

El pod `kafka-cli` simula cómo un **equipo de aplicación** (o el operador haciendo pruebas desde un namespace de inquilino) se conecta al cluster usando únicamente `bootstrap.servers` expuesto por la plataforma.

## Cómo se despliega

- El **cluster de brokers** y el **pod `kafka-cli`** se crean en el [Lab 5](../lab-05-flujo-basico/README.md).
- Los Labs **6 a 12** reutilizan ese despliegue; no vuelven a levantar Kafka.

Manifiesto del pod CLI: [`lab-05-flujo-basico/manifest/apps/client/deployment.yaml`](../lab-05-flujo-basico/manifest/apps/client/deployment.yaml).

## Cómo ejecutar las demos

Desde fuera del cluster solo hace falta `kubectl`:

```bash
# Shell interactiva (recomendado para demos en clase)
kubectl -n app-a exec -it deploy/kafka-cli -- bash

# Un comando suelto
kubectl -n app-a exec deploy/kafka-cli -- kafka-topics --bootstrap-server kafka.kafka.svc.cluster.local:9092 --list
```

Bootstrap del curso:

```text
kafka.kafka.svc.cluster.local:9092
```

ConfigMap opcional montado en el pod: `/etc/kafka-client/client.properties`.

## Alternativa válida (no la habitual en el curso)

Los mismos binarios existen **dentro de cada pod broker**. Es posible ejecutar:

```bash
kubectl -n kafka exec -it kafka-0 -- bash
```

En clase se prefiere **`kafka-cli` en `app-a`** para separar claramente *plataforma* (`kafka`) de *cliente/operación de prueba* (`app-a`), alineado con el modelo multi-tenant por namespace.

## Qué no cubre este pod

| Componente | Imagen / acceso |
|------------|-----------------|
| Schema Registry | `confluentinc/cp-schema-registry` + REST |
| Kafka Connect | `confluentinc/cp-kafka-connect` + REST |
| ksqlDB | `confluentinc/cp-ksqldb-server` + `ksql` / REST |

Esas piezas se despliegan en laboratorios posteriores; no forman parte del rol `kafka-cli`.

## Referencia de comandos

[Lista de comandos `kafka-*` →](cheatsheet-comandos-kafka.md)

---

[← Índice del bloque](../fundamentos/README.md) · [Lab 5 — Despliegue del entorno](../lab-05-flujo-basico/README.md)
