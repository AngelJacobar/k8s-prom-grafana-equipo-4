# Equipo 4 · Proyecto 6 — Observabilidad y Seguridad en Kubernetes

## Descripción

Este repositorio contiene la implementación y evidencia técnica del **Proyecto 6: Observabilidad y Seguridad en Kubernetes**.

El laboratorio utiliza un clúster Kubernetes de tres nodos sobre Rocky Linux 9.8, ejecutado en Hyper-V. El objetivo es demostrar monitoreo, alertamiento, evaluación de configuración con CIS Kubernetes Benchmark y relación de evidencias con controles seleccionados de ISO/IEC 27001:2022.

El servicio de prueba **KubePay** funciona como workload representativo para generar métricas de disponibilidad, rendimiento y negocio.

---

## Arquitectura del laboratorio

El clúster está compuesto por tres máquinas virtuales:

| Nodo | Rol | IP interna | Carga principal |
|---|---|---|---|
| `cp01` | Control plane | `192.168.56.10` | API Server, etcd, scheduler, controller-manager |
| `w01` | Worker | `192.168.56.11` | Stack de monitoreo |
| `w02` | Worker | `192.168.56.12` | KubePay |

### Red

- Hipervisor: **Hyper-V**
- Switch interno: `K8s-Internal`
- Red interna: `192.168.56.0/24`
- CNI: **Flannel**
- Pod CIDR: `10.244.0.0/16`
- Los nodos disponen además de una segunda interfaz para salida a Internet mediante Hyper-V Default Switch.

---

## Componentes principales

- Kubernetes `v1.34.10`
- containerd
- Flannel
- Helm
- Local Path Provisioner
- kube-prometheus-stack
  - Prometheus
  - Grafana
  - Alertmanager
  - kube-state-metrics
  - node-exporter
- kube-bench
- KubePay

---

## KubePay

KubePay se ejecuta en el namespace `payments` y está programado en `w02` mediante:

```yaml
nodeSelector:
  workload: payments
```

Estado final del laboratorio:

- Deployment: `kubepay`
- Réplicas: `1`
- Service: `ClusterIP`
- Puerto: `8080`
- Endpoints:
  - `/healthz`
  - `/readyz`
  - `/metrics`
  - `POST /api/v1/payments`
- ServiceMonitor propio para Prometheus

El contenedor se ejecuta con controles de seguridad como:

- `runAsNonRoot: true`
- UID/GID `10001`
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true`
- `capabilities.drop: ["ALL"]`
- `seccompProfile: RuntimeDefault`

---

## Observabilidad

El stack de observabilidad se ejecuta en el namespace `monitoring`.

Prometheus recopila métricas de:

- nodos;
- kubelet;
- objetos Kubernetes;
- KubePay;
- disponibilidad de targets;
- uso de CPU, memoria y almacenamiento;
- reinicios y estado de Pods.

Grafana presenta seis dashboards finales:

| # | Dashboard |
|---|---|
| 1 | Salud de nodos |
| 2 | KubePay — Salud y rendimiento |
| 3 | Estado del clúster |
| 4 | Capacidad de almacenamiento |
| 5 | Disponibilidad y alertas |
| 6 | Cumplimiento CIS |

Los JSON exportados se almacenan en:

```text
dashboards/
```

---

## Alertas

El proyecto implementa **10 reglas propias de Prometheus**, identificadas con el prefijo `P6`.

1. `P6NodeDown`
2. `P6HostHighCpuLoad`
3. `P6HostMemoryPressure`
4. `P6DiskSpaceFillingUp`
5. `P6KubeletDown`
6. `P6PodCrashLooping`
7. `P6PodOOMKilled`
8. `P6KubePayUnavailable`
9. `P6KubePayHighDeclineRate`
10. `P6KubePayReplicasMismatch`

Cada regla incluye:

- severidad;
- condición o umbral;
- tiempo de persistencia;
- descripción;
- referencia a runbook.

Se demostraron tres alertas disparándose realmente:

- `P6KubePayHighDeclineRate`
- `P6KubePayReplicasMismatch`
- `P6PodCrashLooping`

Las pruebas fueron verificadas en Prometheus y Alertmanager.

Archivos:

```text
manifests/alerts/proyecto6-alert-rules.yaml
docs/runbooks.md
```

---

## Evaluación CIS Kubernetes Benchmark

Se utilizó `kube-bench 0.16.0` con el benchmark `cis-1.12`.

### Resultado agregado

| Evaluación | PASS | FAIL | WARN |
|---|---:|---:|---:|
| Before | 85 | 14 | 52 |
| After | 92 | 7 | 52 |

La cantidad de hallazgos `FAIL` se redujo en **50%**.

Entre las remediaciones aplicadas se encuentran:

- permisos de `kubelet.service`;
- permisos de `/var/lib/kubelet/config.yaml`;
- deshabilitación de profiling en kube-apiserver;
- deshabilitación de profiling en kube-controller-manager;
- deshabilitación de profiling en kube-scheduler.

Los hallazgos residuales permanecen documentados como riesgos o configuraciones pendientes.

---

## ISO/IEC 27001:2022

El laboratorio relaciona sus evidencias con los siguientes controles:

- **A.8.16 — Monitoring activities**
- **A.8.9 — Configuration management**
- **A.8.6 — Capacity management**
- **Cláusula 9.1 — Monitoring, measurement, analysis and evaluation**

La matriz de relación y evidencias se encuentra en:

```text
docs/iso27001.md
```

---

## Estructura principal del repositorio

```text
.
├── README.md
├── dashboards/
│   ├── 01-salud-nodos.json
│   ├── 02-kubepay-salud.json
│   ├── 03-estado-cluster.json
│   ├── 04-capacidad-almacenamiento.json
│   ├── 05-disponibilidad-alertas.json
│   └── 06-cumplimiento-cis.json
├── docs/
│   ├── installation.md
│   ├── troubleshooting.md
│   ├── runbooks.md
│   └── iso27001.md
└── manifests/
    ├── alerts/
    │   └── proyecto6-alert-rules.yaml
    └── kubepay/
        ├── 00-namespace.yaml
        ├── 01-kubepay-configmap.yaml
        ├── 02-kubepay-deployment.yaml
        ├── 03-kubepay-service.yaml
        └── 04-kubepay-servicemonitor.yaml
```

---

## Validación rápida del clúster

Desde `cp01`:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployment,service -n payments
kubectl get servicemonitor -A
kubectl get prometheusrule -n monitoring
```

Estado esperado:

- `cp01`, `w01` y `w02` en `Ready`;
- KubePay `1/1 Running`;
- Prometheus, Grafana y Alertmanager activos;
- ServiceMonitor de KubePay disponible;
- reglas `P6` cargadas.

---

## Acceso a las interfaces

### Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 --address=0.0.0.0
```

### Prometheus

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 --address=0.0.0.0
```

### Alertmanager

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093 --address=0.0.0.0
```

Desde el equipo anfitrión:

```text
Grafana:      http://192.168.56.10:3000
Prometheus:   http://192.168.56.10:9090
Alertmanager: http://192.168.56.10:9093
```

---

## Documentación

- `docs/installation.md` — instalación y despliegue del laboratorio.
- `docs/troubleshooting.md` — incidencias encontradas y soluciones.
- `docs/runbooks.md` — procedimientos de atención de las 10 alertas.
- `docs/iso27001.md` — relación del laboratorio con controles ISO/IEC 27001:2022.

---

## Evidencias

Durante el proyecto se recopilaron evidencias de:

- estado de nodos y Pods;
- componentes de monitoreo;
- targets de Prometheus;
- dashboards de Grafana;
- consultas PromQL;
- reglas de alerta;
- alertas `FIRING`;
- recepción en Alertmanager;
- reportes kube-bench Before/After;
- remediaciones CIS.

Estas evidencias permiten demostrar el funcionamiento real del laboratorio y relacionar los resultados observados con la documentación entregada.
