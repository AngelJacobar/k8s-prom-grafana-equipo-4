# Guía de dashboards de Grafana

**Proyecto 6 — Observabilidad y Seguridad en Kubernetes**

## 1. Propósito

Este documento describe los seis dashboards finales utilizados en el laboratorio y cómo validar sus datos contra Prometheus y Kubernetes.

Los dashboards fueron ajustados con datos reales del entorno final y exportados nuevamente desde Grafana para su inclusión en el repositorio.

La fuente principal de datos es **Prometheus**. El dashboard de cumplimiento CIS utiliza información consolidada de las ejecuciones de `kube-bench`.

---

## 2. Dashboards finales

| # | Dashboard | Archivo |
|---:|---|---|
| 1 | Salud de nodos | `dashboards/01-salud-nodos.json` |
| 2 | KubePay — Salud y rendimiento | `dashboards/02-kubepay-salud.json` |
| 3 | Estado del clúster | `dashboards/03-estado-cluster.json` |
| 4 | Capacidad de almacenamiento | `dashboards/04-capacidad-almacenamiento.json` |
| 5 | Disponibilidad y alertas | `dashboards/05-disponibilidad-alertas.json` |
| 6 | Cumplimiento CIS | `dashboards/06-cumplimiento-cis.json` |

---

# 3. Dashboard 1 — Salud de nodos

## Objetivo

Visualizar el consumo y estado general de los tres nodos:

- `cp01`
- `w01`
- `w02`

## Fuente

`node-exporter`

## Métricas principales

### CPU por nodo

La consulta se basa en el tiempo de CPU ocioso y calcula el porcentaje utilizado.

Ejemplo:

```promql
100 * (
  1 -
  avg by(instance) (
    rate(node_cpu_seconds_total{
      job="node-exporter",
      mode="idle"
    }[5m])
  )
)
```

### Memoria por nodo

```promql
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```

### Uso del disco raíz

```promql
100 * (
  1 -
  node_filesystem_avail_bytes{
    mountpoint="/",
    fstype!~"tmpfs|overlay|squashfs"
  }
  /
  node_filesystem_size_bytes{
    mountpoint="/",
    fstype!~"tmpfs|overlay|squashfs"
  }
)
```

## Línea base observada

Durante operación normal se observaron aproximadamente:

| Nodo | CPU habitual | Memoria habitual |
|---|---:|---:|
| `cp01` | 24–26% | 79–84% |
| `w01` | 14–16% | 53–55% |
| `w02` | 6–7% | 31–33% |

Estos valores se utilizaron como referencia para justificar los umbrales de alertamiento.

## Validación

```bash
kubectl get nodes -o wide
```

En Prometheus confirmar que existen tres series distintas de `node-exporter`.

---

# 4. Dashboard 2 — KubePay: Salud y rendimiento

## Objetivo

Mostrar el comportamiento del workload de demostración `kubepay` y sus métricas propias.

## Fuentes

- métricas expuestas por `/metrics`;
- ServiceMonitor `kubepay`;
- Prometheus.

## Métricas disponibles

### Disponibilidad

```promql
kubepay_up
```

### Pagos aprobados y rechazados

```promql
kubepay_payments_total
```

Separados por:

```text
result="approved"
result="declined"
```

### Solicitudes HTTP

```promql
kubepay_http_requests_total
```

Labels relevantes:

```text
method
path
status
```

### Duración de solicitudes

```promql
kubepay_request_duration_seconds_sum
kubepay_request_duration_seconds_count
```

## Estado final de KubePay

```text
Namespace: payments
Deployment: kubepay
Réplicas: 1
Service: ClusterIP
Puerto: 8080
Nodo: w02
```

## Validación

```bash
kubectl get deployment,pods,svc -n payments
kubectl get servicemonitor -A | grep kubepay
```

También puede comprobarse el endpoint:

```bash
kubectl port-forward -n payments svc/kubepay 8080:8080
curl http://127.0.0.1:8080/metrics
```

---

# 5. Dashboard 3 — Estado del clúster

## Objetivo

Observar el estado de los objetos Kubernetes y detectar cambios operativos.

## Fuente

`kube-state-metrics`

## Paneles principales

### Pods por fase

```promql
sum by(namespace, phase) (
  kube_pod_status_phase{
    phase=~"Running|Pending|Failed|Unknown"
  } == 1
)
```

### Réplicas disponibles por Deployment

```promql
kube_deployment_status_replicas_available{
  namespace=~"monitoring|payments"
}
```

### Reinicios recientes

```promql
sum by(namespace, pod) (
  increase(
    kube_pod_container_status_restarts_total{
      namespace=~"monitoring|payments"
    }[1h]
  )
)
```

## Interpretación

Este dashboard permite identificar:

- Pods en estados distintos de `Running`;
- pérdida de réplicas disponibles;
- contenedores con reinicios recientes;
- cambios operativos en `monitoring` y `payments`.

## Validación

```bash
kubectl get pods -A
kubectl get deployments -A
```

Los valores visualizados deben ser coherentes con el estado reportado por Kubernetes.

---

# 6. Dashboard 4 — Capacidad de almacenamiento

## Objetivo

Mostrar el uso de almacenamiento de Prometheus y el espacio disponible en los nodos.

## Fuentes

- `kubelet_volume_stats_*`
- `node_filesystem_*`
- Local Path Provisioner

## Elementos principales

### Uso del PVC de Prometheus

Se comparan:

```promql
kubelet_volume_stats_used_bytes
```

y:

```promql
kubelet_volume_stats_capacity_bytes
```

para el PVC de Prometheus en el namespace `monitoring`.

### Porcentaje usado

Conceptualmente:

```promql
100 *
kubelet_volume_stats_used_bytes
/
kubelet_volume_stats_capacity_bytes
```

### Disco raíz por nodo

```promql
node_filesystem_avail_bytes{
  mountpoint="/",
  fstype!~"tmpfs|overlay|squashfs"
}
```

## Retención

Prometheus se configuró con:

```text
15 días
```

La retención configurada no garantiza por sí sola que siempre puedan conservarse los 15 días si el volumen se llena antes.

## Validación

```bash
kubectl get pvc -n monitoring
kubectl get pv
kubectl get storageclass
```

Confirmar que el PVC de Prometheus está `Bound`.

---

# 7. Dashboard 5 — Disponibilidad y alertas

## Objetivo

Mostrar el estado de los targets de Prometheus y únicamente las alertas propias del Proyecto 6.

## Fuente

Prometheus.

## Targets disponibles

Ejemplo:

```promql
sum by(job) (up)
```

## Targets caídos

```promql
sum by(job) (up == 0)
```

## Alertas del Proyecto 6 en FIRING

La versión final utiliza:

```promql
sum by(alertname, severity) (
  ALERTS{
    alertstate="firing",
    project="proyecto6"
  }
)
```

El filtro:

```text
project="proyecto6"
```

evita mezclar las reglas propias con las alertas incluidas por `kube-prometheus-stack`.

## Validación

Para demostrar una alerta completa deben observarse tres elementos:

1. condición anómala controlada;
2. alerta `FIRING` en Prometheus;
3. alerta recibida por Alertmanager.

Después se aplica el procedimiento definido en:

```text
docs/runbooks.md
```

## Alertas demostradas

Se probaron realmente:

```text
P6KubePayHighDeclineRate
P6KubePayReplicasMismatch
P6PodCrashLooping
```

---

# 8. Dashboard 6 — Cumplimiento CIS

## Objetivo

Presentar la comparación de resultados obtenidos con `kube-bench` antes y después de las remediaciones.

Este dashboard representa una **evaluación puntual**, no monitoreo continuo.

## Herramienta

```text
kube-bench 0.16.0
Benchmark: cis-1.12
```

## Resultado agregado

| Evaluación | PASS | FAIL | WARN |
|---|---:|---:|---:|
| Before | 85 | 14 | 52 |
| After | 92 | 7 | 52 |

Reducción de hallazgos `FAIL`:

```text
50%
```

## Remediaciones principales

Se documentaron cinco controles distintos:

- permisos de `kubelet.service`;
- permisos de `/var/lib/kubelet/config.yaml`;
- `--profiling=false` en kube-apiserver;
- `--profiling=false` en kube-controller-manager;
- `--profiling=false` en kube-scheduler.

## Evidencia

El dashboard debe interpretarse junto con:

- reportes `kube-bench` Before;
- reportes `kube-bench` After;
- capturas de terminal;
- registro de remediaciones;
- hallazgos residuales.

El dashboard no sustituye dichos reportes.

---

# 9. Importación de dashboards

En Grafana:

```text
Dashboards
  → New
  → Import
```

Seleccionar el archivo JSON correspondiente.

Para los dashboards que lo soliciten, seleccionar la fuente:

```text
Prometheus
```

Después de importar:

1. validar que los paneles muestran datos;
2. comprobar el rango de tiempo;
3. revisar que labels, namespaces y targets coincidan;
4. guardar;
5. exportar nuevamente el JSON definitivo.

---

# 10. Evidencia recomendada

Para cada dashboard conservar:

- captura del dashboard con datos reales;
- archivo JSON final;
- captura o registro de una consulta PromQL representativa cuando aplique.

Además:

```text
Prometheus Targets
Prometheus Alerts
Alertmanager
kube-bench Before/After
```

permiten relacionar los dashboards con evidencia externa verificable.

---

# 11. Estado final

Los seis dashboards fueron utilizados para demostrar:

- salud de infraestructura;
- estado de Kubernetes;
- métricas propias de KubePay;
- capacidad de almacenamiento;
- disponibilidad y alertamiento;
- cumplimiento CIS.

Los archivos JSON exportados desde Grafana constituyen las versiones finales almacenadas en el repositorio.
