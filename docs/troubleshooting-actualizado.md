# Guía de solución de problemas (Troubleshooting)

**Equipo 4 · Proyecto 6: Observabilidad y Seguridad en Kubernetes**

## 1. Propósito

Este documento registra las incidencias técnicas reales encontradas durante la construcción del laboratorio, su diagnóstico, la solución aplicada y el estado final.

El proyecto inició sobre **VirtualBox** y posteriormente migró a **Hyper-V** debido a problemas de estabilidad observados en el host Windows. Por ello, algunas incidencias corresponden a la etapa histórica de VirtualBox y se conservan como evidencia del proceso de ingeniería.

### Estado final de referencia

- Hipervisor: **Hyper-V**
- Red interna: `K8s-Internal`
- `cp01`: `192.168.56.10`
- `w01`: `192.168.56.11`
- `w02`: `192.168.56.12`
- Interfaz interna Linux: `eth0`
- Interfaz de Internet: `eth1`
- Flannel sobre `eth0`
- `firewalld`: deshabilitado en el laboratorio final
- KubePay: 1 réplica
- Kubernetes: `v1.34.10`

---

## 2. Resumen de incidencias

| ID | Incidencia | Etapa | Estado |
|---:|---|---|---|
| 01 | `No route to host` entre Pods por firewalld | VirtualBox | Histórica |
| 02 | `kubectl` intenta conectar a `localhost:8080` en worker | Ambas | Resuelta |
| 03 | Reejecución accidental de `kubeadm init` | Ambas | Resuelta |
| 04 | Desalineación de cgroups entre kubelet y containerd | Ambas | Resuelta |
| 05 | Flannel selecciona interfaz incorrecta en host multi-NIC | Ambas | Resuelta |
| 06 | Helm no instala por ausencia de `tar` | Ambas | Resuelta |
| 07 | `retentionSize` rechazado por CRD de Prometheus | Ambas | Resuelta |
| 08 | KubePay en `CrashLoopBackOff` por error de Python | Desarrollo | Resuelta |
| 09 | Grafana reinicia por liveness probe demasiado agresiva | Desarrollo | Resuelta |
| 10 | Grafana sin datos por drift de reloj | VirtualBox | Histórica |
| 11 | `kubectl port-forward` se interrumpe | Ambas | Mitigada |
| 12 | Dashboard sin datos por filtro de IP fijo | Ambas | Resuelta |
| 13 | Déficit de réplicas / `ImagePullBackOff` durante pruebas | Desarrollo | Resuelta |
| 14 | Soft lockups/watchdog en VirtualBox sobre Windows | VirtualBox | Resuelta migrando a Hyper-V |
| 15 | Error YAML al aplicar `PrometheusRule` | Alertas | Resuelta |
| 16 | Alerta CrashLoop oscilando entre `PENDING` e `INACTIVE` | Alertas | Resuelta |

---

# 3. Incidencias detalladas

## 01 — `No route to host` entre Pods por firewalld

### Síntoma

Durante la etapa inicial con VirtualBox, la comunicación Pod-a-Pod y hacia CoreDNS podía fallar con:

```text
No route to host
100% packet loss
```

### Causa

`firewalld` interfería con el forwarding entre las interfaces creadas por Flannel (`cni0` y `flannel.1`).

### Diagnóstico

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
iptables -S FORWARD
```

### Solución utilizada en esa etapa

Se permitieron explícitamente la red de Pods y las interfaces de Flannel.

### Estado final

En la arquitectura final con Hyper-V se optó por mantener:

```bash
sudo systemctl disable --now firewalld
```

Esto simplifica el laboratorio académico y evita interferencias con VXLAN/forwarding.

> No debe interpretarse como recomendación para producción.

---

## 02 — `kubectl` intenta conectar a `localhost:8080`

### Síntoma

En un worker:

```text
The connection to the server localhost:8080 was refused
```

### Causa

El usuario no tenía un archivo kubeconfig válido en:

```text
$HOME/.kube/config
```

### Solución

La administración normal del laboratorio se realiza desde `cp01`.

Validar:

```bash
ls -la ~/.kube/config
kubectl config current-context
kubectl get nodes
```

---

## 03 — Reejecución accidental de `kubeadm init`

### Síntoma

```text
Port 6443 is in use
... /etc/kubernetes/manifests/... already exists
```

### Causa

Se ejecutó nuevamente `kubeadm init` sobre un control plane ya inicializado.

### Solución

Antes de ejecutar cualquier reset, validar:

```bash
kubectl get nodes
sudo ss -lntp | grep 6443
```

Si el clúster está operativo, no repetir `kubeadm init`.

---

## 04 — Desalineación de cgroups

### Síntoma

Kubelet inestable o nodo que no mantiene estado `Ready`.

### Causa

`containerd` y kubelet no utilizaban el mismo controlador de cgroups.

### Solución

En:

```text
/etc/containerd/config.toml
```

confirmar:

```toml
SystemdCgroup = true
```

Después:

```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

---

## 05 — Flannel selecciona la interfaz incorrecta

### Síntoma

Los nodos aparecen configurados, pero los Pods de distintos workers no se comunican.

### Causa

Las VMs tienen dos interfaces y Flannel puede seleccionar la interfaz asociada a la ruta por defecto.

### Etapa VirtualBox

La interfaz privada era:

```text
enp0s8
```

### Estado final Hyper-V

La interfaz interna es:

```text
eth0
```

Por ello Flannel debe usar:

```text
--iface=eth0
```

### Validación

```bash
kubectl logs -n kube-flannel -l app=flannel -c kube-flannel --tail=100
ip -br addr
ip route
```

---

## 06 — Helm falla porque Rocky Minimal no incluye `tar`

### Síntoma

```text
[ERROR] Could not find tar
Failed to install helm
```

### Solución

```bash
sudo dnf install -y tar git
```

Después instalar Helm nuevamente y validar:

```bash
helm version
```

---

## 07 — `retentionSize` rechazado por Prometheus Operator

### Síntoma

El CRD rechaza:

```text
retentionSize: 18Gi
```

### Causa

La validación del campo esperaba una unidad compatible con el formato aceptado por el CRD.

### Solución utilizada

```text
retentionSize: 18GiB
```

### Validación

```bash
kubectl get prometheus -n monitoring
```

---

## 08 — KubePay entra en `CrashLoopBackOff`

### Síntoma

Un Pod nuevo de KubePay fallaba al iniciar.

### Causa

Error sintáctico en el código Python almacenado en el ConfigMap.

### Diagnóstico

```bash
kubectl get pods -n payments
kubectl describe pod -n payments <pod>
kubectl logs -n payments <pod>
```

### Solución

Corregir el código del ConfigMap y volver a aplicar:

```bash
kubectl apply -f manifests/kubepay/01-kubepay-configmap.yaml
kubectl rollout restart deployment kubepay -n payments
kubectl rollout status deployment kubepay -n payments
```

---

## 09 — Grafana se reinicia por liveness probe

### Síntoma

Grafana podía reiniciarse bajo carga de consultas, con eventos de liveness fallida.

### Diagnóstico

```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana --tail=50
```

### Solución

Se incrementó la tolerancia de las probes para evitar reinicios falsos bajo carga.

### Validación

```bash
kubectl get pods -n monitoring
```

Grafana debe permanecer estable en `Running`.

---

## 10 — Grafana muestra `No data` por drift de reloj

### Etapa

**Incidencia histórica de VirtualBox.**

### Síntoma

Grafana mostraba `No data` en ventanas recientes pese a existir métricas.

### Causa

El reloj de las VMs quedaba desfasado después de pausas o suspensión del host.

### Diagnóstico

```bash
date
timedatectl
chronyc tracking
chronyc sources -v
```

### Solución

```bash
sudo systemctl restart chronyd
sudo chronyc makestep
```

### Estado final

La migración a Hyper-V eliminó el patrón de inestabilidad que motivó esta incidencia, aunque la sincronización NTP sigue siendo una comprobación recomendada.

---

## 11 — `kubectl port-forward` se interrumpe

### Síntoma

Grafana, Prometheus o Alertmanager dejan de responder desde Windows.

### Causa

`port-forward` depende del ciclo de vida del Pod o de la sesión que mantiene el túnel.

### Diagnóstico

```bash
ps aux | grep port-forward
ss -lntp | grep -E '3000|9090|9093'
```

### Solución

Reiniciar el comando correspondiente.

Grafana:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-grafana \
  3000:80 \
  --address=0.0.0.0
```

Prometheus:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090 \
  --address=0.0.0.0
```

Alertmanager:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-alertmanager \
  9093:9093 \
  --address=0.0.0.0
```

---

## 12 — Dashboard sin datos por filtro de IP fijo

### Síntoma

Un panel de Grafana muestra:

```text
No data
```

aunque el target correspondiente está `UP`.

### Causa

La consulta PromQL incluía una IP fija perteneciente a otra versión del laboratorio.

### Ejemplo problemático

```promql
node_filesystem_size_bytes{instance="192.168.56.11:9100"}
```

si el entorno usa otra dirección.

### Solución

Evitar dependencias innecesarias de IP fija y utilizar labels o filtros compatibles con el entorno real.

### Validación

Primero consultar las series existentes en Prometheus y después ajustar el dashboard.

---

## 13 — Déficit de réplicas / `ImagePullBackOff` durante pruebas

### Contexto

Durante una etapa anterior se utilizaron dos réplicas de KubePay y se realizaron pruebas de fallo de rollout.

### Síntoma

Una réplica quedaba activa y otra en:

```text
ImagePullBackOff
ErrImagePull
```

### Causa

Se configuró deliberadamente una imagen inexistente durante una prueba.

### Solución histórica

```bash
kubectl rollout undo deployment kubepay -n payments
```

### Estado final

El diseño final del laboratorio se estandarizó en:

```text
KubePay: 1 réplica
```

Por lo tanto, este caso se conserva únicamente como evidencia histórica de troubleshooting.

---

## 14 — Soft lockups y watchdog en VirtualBox

### Síntoma

Durante ejecución sostenida del laboratorio en VirtualBox se observaron:

```text
soft lockup
watchdog timeout
```

junto con pausas prolongadas y degradación de respuesta de las VMs.

### Contexto

El host utiliza Windows y disponía de funciones de virtualización/VBS activas.

### Análisis

El problema no se atribuyó al funcionamiento lógico de Kubernetes. Los síntomas aparecían en la capa de virtualización y comprometían la estabilidad necesaria para ejecutar Prometheus, Grafana y las pruebas de alertamiento.

### Decisión

Se abandonó VirtualBox como plataforma final y se migró el laboratorio a **Hyper-V**.

Se mantuvieron:

- Rocky Linux 9.8;
- Kubernetes;
- topología de tres nodos;
- Flannel;
- namespaces;
- stack de monitoreo;
- KubePay.

Cambió principalmente:

```text
Hipervisor: VirtualBox -> Hyper-V
Red privada: Host-Only -> K8s-Internal
Interfaces: enp0s8/enp0s3 -> eth0/eth1
```

### Resultado

Después de la migración se consiguió una plataforma suficientemente estable para terminar:

- los seis dashboards;
- las diez alertas;
- las tres demostraciones reales;
- kube-bench Before/After;
- hardening CIS;
- recolección de evidencias.

Por esta razón Hyper-V constituye la arquitectura final documentada del proyecto.

---

## 15 — Error YAML al aplicar `PrometheusRule`

### Síntoma

Al validar las reglas:

```bash
kubectl apply --dry-run=server -f ~/proyecto6-alert-rules.yaml
```

se obtuvo un error similar a:

```text
error converting YAML to JSON:
yaml: line 243: could not find expected ':'
```

### Causa

Existía un carácter `/` aislado dentro del YAML.

### Diagnóstico

Se inspeccionó el rango alrededor de la línea reportada:

```bash
nl -ba ~/proyecto6-alert-rules.yaml | sed -n '235,252p'
```

### Solución

Eliminar la línea inválida y volver a validar:

```bash
sed -i '243d' ~/proyecto6-alert-rules.yaml
kubectl apply --dry-run=server -f ~/proyecto6-alert-rules.yaml
```

Una vez superado el dry-run:

```bash
kubectl apply -f ~/proyecto6-alert-rules.yaml
```

---

## 16 — `P6PodCrashLooping` oscila entre `PENDING` e `INACTIVE`

### Síntoma

La alerta destinada a detectar `CrashLoopBackOff` entraba en `PENDING`, después regresaba temporalmente a `INACTIVE` y volvía a comenzar.

### Causa

La métrica:

```promql
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}
```

no permanece continuamente en `1`.

Durante cada intento automático de reinicio, Kubernetes cambia transitoriamente el estado del contenedor, lo que reiniciaba el temporizador `for`.

### Primera versión

La condición directa sobre `waiting_reason` resultó demasiado sensible a estos cambios transitorios.

### Solución final

Se cambió la expresión por:

```promql
(
  max_over_time(
    kube_pod_container_status_waiting_reason{
      reason="CrashLoopBackOff"
    }[5m]
  ) == 1
)
and on(namespace, pod, container)
(
  increase(
    kube_pod_container_status_restarts_total[10m]
  ) >= 2
)
```

con:

```yaml
for: 30s
```

### Justificación

La nueva lógica combina:

- evidencia reciente de `CrashLoopBackOff`;
- al menos dos reinicios en 10 minutos;
- persistencia de 30 segundos.

Esto evita que un cambio transitorio de estado reinicie constantemente la evaluación.

### Validación

La regla final fue confirmada en Prometheus y alcanzó:

```text
FIRING
```

durante la prueba controlada con:

```text
p6-crashloop-demo
```

y también fue recibida por Alertmanager.

---

# 4. Comandos generales de diagnóstico

## Estado del clúster

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

## Eventos recientes

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

## Estado de KubePay

```bash
kubectl get deployment,pods,svc -n payments -o wide
```

## Estado de monitoreo

```bash
kubectl get pods -n monitoring -o wide
```

## Targets y recursos de Prometheus Operator

```bash
kubectl get servicemonitor,prometheusrule -A
```

## Runtime

```bash
sudo systemctl status containerd --no-pager
sudo systemctl status kubelet --no-pager
```

## Red

```bash
ip -br addr
ip route
```

---

# 5. Criterio de recuperación

Después de resolver una incidencia, comprobar:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployment kubepay -n payments
kubectl get prometheusrule proyecto6-alert-rules -n monitoring
```

Estado esperado:

```text
cp01   Ready
w01    Ready
w02    Ready
```

KubePay:

```text
READY   UP-TO-DATE   AVAILABLE
1/1     1            1
```

También deben estar disponibles:

- Prometheus;
- Grafana;
- Alertmanager;
- ServiceMonitor de KubePay;
- reglas `P6`.

---

# 6. Lección principal

La principal conclusión operativa del troubleshooting fue que no todos los problemas observados pertenecían a Kubernetes.

La investigación permitió separar fallas entre distintas capas:

```text
Host / hipervisor
      ↓
Sistema operativo
      ↓
Runtime
      ↓
Kubernetes
      ↓
CNI
      ↓
Aplicación
      ↓
Observabilidad
```

La migración de VirtualBox a Hyper-V es el ejemplo más importante: cambiar la capa de virtualización estabilizó el entorno sin modificar el diseño lógico del clúster.

Mantener esta separación por capas facilitó identificar causas raíz, aplicar cambios controlados y conservar evidencia reproducible del proceso.
