# LAB 2 — Escalado y actualizaciones (con rollback)

[← Anterior: Lab 1 — Despliegue](../lab-01-despliegue/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 3 — Diagnóstico →](../lab-03-diagnostico/README.md)

---

## Objetivo

Operar el ciclo de vida de una aplicación ya desplegada:

- **Escalar** réplicas hacia arriba y hacia abajo.
- Hacer una **actualización progresiva** (`RollingUpdate`) sin interrupción de servicio.
- Provocar una actualización fallida y hacer **rollback** a la versión anterior.

## Prerrequisitos

- [Lab 1](../lab-01-despliegue/README.md) completado: el Deployment `web` y el Service `web` siguen vivos en el namespace `lab01`.

Comprueba el punto de partida:

```bash
kubectl -n lab01 get deploy,svc,pods
```

Debe haber **3 réplicas** de `web` en estado `Running` y `Ready`.

## Conceptos rápidos

- **Escalado horizontal** — cambiar el número de réplicas. Lo asume el ReplicaSet sin tocar el Deployment.
- **RollingUpdate** — estrategia por defecto: sustituye pods **de uno en uno** (acotado por `maxUnavailable` y `maxSurge`) para no perder disponibilidad.
- **Rollout** — cada cambio del `template` del Deployment genera una nueva revisión y un nuevo ReplicaSet.
- **Rollback** — volver a la revisión anterior; Kubernetes guarda histórico (`revisionHistoryLimit`).

## Paso 1 — Escalar el Deployment

Sube de 3 a 5 réplicas y observa la creación de los pods nuevos:

```bash
kubectl -n lab01 scale deploy/web --replicas=5
kubectl -n lab01 get pods -w
```

Cuando los 5 estén `Ready`, comprueba que el Service ya reparte tráfico entre todos:

```bash
kubectl -n lab01 get endpoints web
```

Vuelve a 3:

```bash
kubectl -n lab01 scale deploy/web --replicas=3
```

> En producción esto lo hace normalmente un **HorizontalPodAutoscaler (HPA)**, no una persona.

## Paso 2 — Actualización progresiva (rolling)

El Deployment del Lab 1 usa la imagen `nginx:1.27-alpine`. Vamos a actualizarlo a `nginx:1.27` (variante Debian) aplicando un manifiesto nuevo:

```bash
kubectl apply -f bloque-1-kubernetes/lab-02-escalado-actualizaciones/manifests/deployment-v2.yaml
```

En **otra terminal**, observa el rollout en directo:

```bash
kubectl -n lab01 rollout status deploy/web
kubectl -n lab01 get pods -w
```

Verás que:

- Aparece un **ReplicaSet nuevo** (`kubectl -n lab01 get rs`) con la imagen `nginx:1.27`.
- Los pods se sustituyen de uno en uno: nace uno nuevo (`maxSurge: 1`), se elimina uno viejo (`maxUnavailable: 1`).
- El Service nunca pierde endpoints disponibles → **sin downtime**.

Mira el histórico de revisiones:

```bash
kubectl -n lab01 rollout history deploy/web
```

## Paso 3 — Provocar un rollout fallido

Aplica un manifiesto con una imagen que **no existe**:

```bash
kubectl apply -f bloque-1-kubernetes/lab-02-escalado-actualizaciones/manifests/deployment-broken.yaml
```

Observa el rollout y los pods:

```bash
kubectl -n lab01 rollout status deploy/web --timeout=60s
kubectl -n lab01 get pods
```

Resultado esperado:

- Los pods nuevos quedan en `ImagePullBackOff` o `ErrImagePull`.
- `RollingUpdate` se detiene: mantiene los pods viejos vivos porque `maxUnavailable: 1` no permite tirarlos sin un sustituto sano.
- `rollout status` termina con error o expira el timeout.

Diagnostica con:

```bash
kubectl -n lab01 describe pod -l app=web | head -n 40
```

Los `Events` al final del describe explican el motivo: `Failed to pull image "nginx:does-not-exist"`.

## Paso 4 — Rollback a la versión anterior

```bash
kubectl -n lab01 rollout undo deploy/web
kubectl -n lab01 rollout status deploy/web
kubectl -n lab01 get pods
```

El cluster vuelve a la última revisión sana (la del Paso 2). El rollout completa y los pods quedan `Ready`.

Puedes confirmar con qué imagen está corriendo ahora:

```bash
kubectl -n lab01 get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## Comprobaciones

- [ ] Has escalado a 5 réplicas y vuelto a 3 sin downtime.
- [ ] El rollout a `nginx:1.27` se completó con un ReplicaSet nuevo y sin perder endpoints.
- [ ] El rollout a la imagen rota se quedó bloqueado con pods en `ImagePullBackOff`.
- [ ] `kubectl rollout undo` recuperó el servicio.

## Retos opcionales

1. **Forzar un nuevo rollout sin cambiar el manifest** (útil tras tocar un ConfigMap o Secret):
   ```bash
   kubectl -n lab01 rollout restart deploy/web
   ```
2. **Volver a una revisión concreta** del histórico:
   ```bash
   kubectl -n lab01 rollout history deploy/web
   kubectl -n lab01 rollout undo deploy/web --to-revision=1
   ```
3. **Cambiar la estrategia** a `Recreate` (tira todo y vuelve a crear) y repite el rolling: verás un breve corte de servicio.

## Troubleshooting

| Síntoma | Causa habitual | Pista |
|---------|----------------|------|
| `rollout status` se queda colgado | Pods nuevos no llegan a `Ready` | `describe pod` para ver el motivo (imagen, probe, recursos). |
| Rollback no hace nada | No hay revisiones previas (Deployment recién creado) | Revisa `rollout history`. |
| Pods se reinician en bucle al actualizar | Probes mal configuradas o imagen nueva con bug | Mira `kubectl logs <pod>` y los `Events`. |
| Tras `rollout restart`, queda 1 réplica vieja | Pod con `Terminating` largo | Espera o forza `--grace-period=0 --force` (último recurso). |

## Limpieza

Mantén el namespace `lab01` si vas a continuar. Si quieres ir limpio al [Lab 3](../lab-03-diagnostico/README.md), no es necesario borrar nada: el Lab 3 trabaja en su propio namespace.

---

[← Anterior: Lab 1 — Despliegue](../lab-01-despliegue/README.md) · [Índice del bloque ↑](../fundamentos/README.md) · [Siguiente: Lab 3 — Diagnóstico →](../lab-03-diagnostico/README.md)
