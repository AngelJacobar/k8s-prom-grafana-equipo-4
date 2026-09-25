# Guía de instalación y validación — Proyecto 6

**Equipo 4 · Proyecto 6: Observabilidad y Seguridad en Kubernetes**

## 1. Propósito

Esta guía documenta la instalación y el estado final reproducible del laboratorio Kubernetes utilizado en el Proyecto 6.

El entorno final utiliza:

- Windows con **Hyper-V** como hipervisor.
- Tres máquinas virtuales con **Rocky Linux 9.8 Minimal**.
- Kubernetes `v1.34.10`.
- `containerd`.
- Flannel como CNI.
- Local Path Provisioner.
- `kube-prometheus-stack`.
- KubePay como workload de demostración.
- `kube-bench` para evaluación CIS.

No se almacenan contraseñas, tokens, hashes de certificados ni llaves privadas en este documento ni en el repositorio.

---

## 2. Decisión de arquitectura: VirtualBox → Hyper-V

### 2.1 Diseño original

La primera versión del laboratorio fue diseñada para ejecutarse sobre **VirtualBox 7.2.18**, utilizando:

- una interfaz NAT para salida a Internet;
- una red Host-Only para comunicación entre nodos;
- tres VMs Rocky Linux 9.8.

La arquitectura funcionó a nivel lógico y permitió avanzar con la instalación inicial de Kubernetes.

### 2.2 Problema observado

Durante las pruebas sostenidas en el equipo Windows se presentaron problemas de estabilidad en las VMs Linux, entre ellos:

- `soft lockups`;
- eventos de `watchdog`;
- pausas prolongadas de ejecución;
- degradación de respuesta bajo carga.

El comportamiento apareció en un host Windows donde las capacidades de virtualización de Microsoft y VBS estaban presentes. Aunque el clúster Kubernetes podía arrancar, la inestabilidad del hipervisor afectaba la confiabilidad de las pruebas y de las evidencias.

No se concluyó que Kubernetes fuera la causa del problema; el comportamiento se aisló a la capa de virtualización del host.

### 2.3 Decisión

Se decidió migrar el laboratorio a **Hyper-V**, manteniendo la misma arquitectura lógica de tres nodos.

La decisión se tomó por tres razones:

1. utilizar el hipervisor nativo del host Windows;
2. evitar la convivencia problemática observada entre VirtualBox y las funciones de virtualización/VBS del sistema;
3. conseguir una plataforma estable para ejecutar Prometheus, Grafana, Alertmanager, kube-bench y las pruebas de alertamiento.

La migración no cambió el objetivo del proyecto ni la arquitectura Kubernetes. Cambió únicamente la plataforma de virtualización y la forma de presentar las interfaces de red a las VMs.

---

## 3. Arquitectura final del laboratorio

### 3.1 Nodos

| Nodo | Rol | IP interna | Recursos actuales | Carga principal |
|---|---|---:|---:|---|
| `cp01` | Control plane | `192.168.56.10` | 2 vCPU / 2 GiB RAM | API Server, etcd, scheduler, controller-manager |
| `w01` | Worker | `192.168.56.11` | 2 vCPU / 3 GiB RAM | Observabilidad |
| `w02` | Worker | `192.168.56.12` | 2 vCPU / 2 GiB RAM | KubePay |

> Estos recursos corresponden al laboratorio final validado. Para una instalación con mayor capacidad física pueden incrementarse, especialmente en `w01`, donde se ejecuta Prometheus.

### 3.2 Red Hyper-V

Se utilizan dos adaptadores virtuales por VM:

- `eth0`: conectado al switch interno `K8s-Internal`;
- `eth1`: conectado a Hyper-V `Default Switch` para salida a Internet.

Red interna:

```text
192.168.56.0/24
```

Host Windows sobre `K8s-Internal`:

```text
192.168.56.1/24
```

Nodos:

```text
cp01  192.168.56.10
w01   192.168.56.11
w02   192.168.56.12
```

Red de Pods:

```text
10.244.0.0/16
```

---

## 4. Creación de la red en Hyper-V

En **Hyper-V Manager**:

1. Abrir **Virtual Switch Manager**.
2. Crear un nuevo switch de tipo **Internal**.
3. Asignar el nombre:

```text
K8s-Internal
```

4. En Windows, configurar el adaptador:

```text
vEthernet (K8s-Internal)
```

con:

```text
IPv4:    192.168.56.1
Máscara: 255.255.255.0
Gateway: vacío
DNS:     vacío
```

El switch interno se utiliza exclusivamente para comunicación estable entre el host y las VMs.

Cada VM debe tener además un segundo adaptador conectado a:

```text
Default Switch
```

para acceso a Internet.

---

## 5. Creación de las máquinas virtuales

Crear tres VMs compatibles con Rocky Linux 9.8:

```text
cp01
w01
w02
```

Asignar los recursos indicados en la sección de arquitectura.

Instalar:

```text
Rocky-9.8-x86_64-minimal.iso
```

Durante la instalación:

- no crear swap;
- utilizar el espacio principal para `/`;
- crear un usuario administrativo con `sudo`;
- no habilitar acceso SSH de root mediante contraseña;
- instalar únicamente el perfil Minimal.

---

## 6. Configuración de red en Rocky Linux

Primero identificar las interfaces:

```bash
ip -br addr
nmcli connection show
ip route
```

En el laboratorio final:

```text
eth0 = red interna K8s-Internal
eth1 = Hyper-V Default Switch / Internet
```

### 6.1 cp01

Configurar `eth0` con:

```text
192.168.56.10/24
```

Ejemplo con NetworkManager:

```bash
sudo nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.56.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes

sudo nmcli connection up eth0
```

### 6.2 w01

```bash
sudo nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.56.11/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes

sudo nmcli connection up eth0
```

### 6.3 w02

```bash
sudo nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.56.12/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes

sudo nmcli connection up eth0
```

Validar en cada VM:

```bash
ip -br addr
ip route
nmcli connection show
```

La ruta predeterminada debe permanecer sobre la interfaz conectada al `Default Switch`.

---

## 7. Resolución local de nombres

En los tres nodos agregar a `/etc/hosts`:

```text
192.168.56.10 cp01
192.168.56.11 w01
192.168.56.12 w02
```

Validar:

```bash
getent hosts cp01 w01 w02
ping -c 3 cp01
ping -c 3 w01
ping -c 3 w02
```

---

## 8. Preparación de Rocky Linux

En los tres nodos:

```bash
sudo dnf update -y
sudo dnf install -y curl wget tar git dnf-plugins-core
```

Verificar que swap esté deshabilitada:

```bash
swapon --show
sudo swapoff -a
```

Si existe una entrada swap en `/etc/fstab`, deshabilitarla de forma persistente.

---

## 9. Módulos y parámetros de kernel

En los tres nodos:

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/kubernetes.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Crear:

```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Aplicar:

```bash
sudo sysctl --system
```

Validar:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

---

## 10. SELinux y firewalld

Durante la construcción se utilizó SELinux en modo permisivo para compatibilidad con el laboratorio:

```bash
sudo setenforce 0
sudo sed -i.bak 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
getenforce
```

### firewalld

En la versión final del laboratorio `firewalld` se dejó **deshabilitado** durante las pruebas Kubernetes.

```bash
sudo systemctl disable --now firewalld
systemctl is-active firewalld
```

Resultado esperado:

```text
inactive
```

Esta decisión se tomó para eliminar interferencias con el forwarding y VXLAN de Flannel durante el laboratorio.

> **Importante:** esta configuración es una simplificación del entorno académico. No debe interpretarse como recomendación para un clúster de producción. En producción deben definirse reglas explícitas de red y firewall.

---

## 11. Instalación de containerd

Agregar el repositorio:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Instalar containerd:

```bash
sudo dnf install -y containerd.io
```

Generar configuración:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Asegurar:

```text
SystemdCgroup = true
```

Iniciar:

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
containerd --version
```

---

## 12. Instalación de Kubernetes 1.34

Crear `/etc/yum.repos.d/kubernetes.repo`:

```ini
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

Instalar la versión utilizada en el laboratorio:

```bash
sudo dnf install -y \
  kubelet-1.34.10-150500.1.1 \
  kubeadm-1.34.10-150500.1.1 \
  kubectl-1.34.10-150500.1.1 \
  --disableexcludes=kubernetes
```

Habilitar kubelet:

```bash
sudo systemctl enable --now kubelet
```

Validar:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

---

## 13. Inicialización del control plane

En `cp01`:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Configurar `kubectl`:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

Guardar de forma segura el comando `kubeadm join`.

---

## 14. Unión de los workers

En `w01` y `w02` ejecutar el comando generado por `kubeadm init`.

Formato:

```bash
sudo kubeadm join 192.168.56.10:6443 \
  --token TOKEN \
  --discovery-token-ca-cert-hash sha256:HASH \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Si el token expira:

```bash
kubeadm token create --print-join-command
```

---

## 15. Instalación de Flannel

Descargar el manifiesto:

```bash
curl -LO https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Debido a que cada VM tiene dos interfaces, Flannel debe utilizar la interfaz interna `eth0`.

Configurar el argumento correspondiente:

```text
--iface=eth0
```

Aplicar:

```bash
kubectl apply -f kube-flannel.yml
```

Validar:

```bash
kubectl get nodes
kubectl get pods -n kube-flannel -o wide
```

Los tres nodos deben aparecer `Ready`.

---

## 16. Etiquetas de los workers

En `cp01`:

```bash
kubectl label node w01 workload=monitoring --overwrite
kubectl label node w02 workload=payments --overwrite
kubectl get nodes --show-labels
```

---

## 17. Helm y almacenamiento local

Instalar Helm y validar:

```bash
helm version
```

Instalar Local Path Provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
```

Validar:

```bash
kubectl get storageclass
kubectl get pods -n local-path-storage
```

La StorageClass utilizada es:

```text
local-path
```

---

## 18. kube-prometheus-stack

Crear el namespace:

```bash
kubectl create namespace monitoring
```

Instalar `kube-prometheus-stack` mediante Helm utilizando los valores definidos para el laboratorio.

Versión validada:

```text
Chart: 91.4.1
App:   v0.94.0
```

Estado final esperado:

- Prometheus `2/2 Running`;
- Grafana `3/3 Running`;
- Alertmanager `2/2 Running`;
- kube-state-metrics activo;
- node-exporter en los tres nodos.

Validar:

```bash
kubectl get pods -n monitoring -o wide
kubectl get pvc -n monitoring
kubectl get svc -n monitoring
```

Prometheus utiliza un PVC de 20 GiB y retención configurada de 15 días.

---

## 19. Despliegue de KubePay

Los manifiestos declarativos se almacenan en:

```text
manifests/kubepay/
```

Aplicar:

```bash
kubectl apply -f manifests/kubepay/00-namespace.yaml
kubectl apply -f manifests/kubepay/01-kubepay-configmap.yaml
kubectl apply -f manifests/kubepay/02-kubepay-deployment.yaml
kubectl apply -f manifests/kubepay/03-kubepay-service.yaml
kubectl apply -f manifests/kubepay/04-kubepay-servicemonitor.yaml
```

Estado final:

```text
Deployment: kubepay
Réplicas:   1
Namespace:  payments
Nodo:       w02
Service:    ClusterIP
Puerto:     8080
```

Validar:

```bash
kubectl get deployment,service,configmap -n payments
kubectl get servicemonitor -A | grep kubepay
kubectl get pods -n payments -o wide
```

---

## 20. Validación de métricas de KubePay

Acceso temporal:

```bash
kubectl port-forward -n payments svc/kubepay 8080:8080
```

Pruebas:

```bash
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/readyz
curl http://127.0.0.1:8080/metrics
```

Pago aprobado:

```bash
curl -X POST http://127.0.0.1:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{"amount":100,"currency":"MXN"}'
```

Pago rechazado por lógica de negocio:

```bash
curl -X POST http://127.0.0.1:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{"amount":20000,"currency":"MXN"}'
```

Un pago rechazado por lógica de negocio continúa respondiendo HTTP `200`; el resultado se identifica mediante:

```json
{"status":"declined"}
```

---

## 21. Acceso a Grafana, Prometheus y Alertmanager

### Grafana

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-grafana \
  3000:80 \
  --address=0.0.0.0
```

Desde Windows:

```text
http://192.168.56.10:3000
```

### Prometheus

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090 \
  --address=0.0.0.0
```

```text
http://192.168.56.10:9090
```

### Alertmanager

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-alertmanager \
  9093:9093 \
  --address=0.0.0.0
```

```text
http://192.168.56.10:9093
```

---

## 22. Dashboards

Importar los seis JSON almacenados en:

```text
dashboards/
```

Los dashboards finales son:

1. Salud de nodos.
2. KubePay — Salud y rendimiento.
3. Estado del clúster.
4. Capacidad de almacenamiento.
5. Disponibilidad y alertas.
6. Cumplimiento CIS.

Confirmar que los paneles muestran datos reales antes de exportar la versión definitiva.

---

## 23. Reglas de alerta

Aplicar:

```bash
kubectl apply -f manifests/alerts/proyecto6-alert-rules.yaml
```

Validar:

```bash
kubectl get prometheusrule proyecto6-alert-rules -n monitoring
```

El recurso contiene 10 reglas propias:

```text
P6NodeDown
P6HostHighCpuLoad
P6HostMemoryPressure
P6DiskSpaceFillingUp
P6KubeletDown
P6PodCrashLooping
P6PodOOMKilled
P6KubePayUnavailable
P6KubePayHighDeclineRate
P6KubePayReplicasMismatch
```

Los procedimientos de atención se encuentran en:

```text
docs/runbooks.md
```

---

## 24. Demostración de alertas

Se validaron tres alertas reales:

```text
P6KubePayHighDeclineRate
P6KubePayReplicasMismatch
P6PodCrashLooping
```

Para cada prueba se verificó:

1. condición anómala controlada;
2. regla en estado `FIRING` en Prometheus;
3. recepción en Alertmanager;
4. recuperación del entorno.

Al terminar las pruebas, KubePay quedó nuevamente con una réplica y los tres nodos en `Ready`.

---

## 25. Evaluación CIS con kube-bench

Versión:

```text
kube-bench 0.16.0
```

Benchmark utilizado:

```text
cis-1.12
```

Resultado agregado:

| Evaluación | PASS | FAIL | WARN |
|---|---:|---:|---:|
| Before | 85 | 14 | 52 |
| After | 92 | 7 | 52 |

Reducción de hallazgos `FAIL`:

```text
50%
```

Se aplicaron cinco controles distintos de hardening, incluyendo permisos de archivos de kubelet y deshabilitación de profiling en componentes del control plane.

Los reportes Before/After deben conservarse como evidencia.

---

## 26. Checkpoints de Hyper-V

Después de validar el clúster se crearon checkpoints para facilitar recuperación del laboratorio.

Checkpoints relevantes:

```text
Pre-CIS
Post-CIS-Hardening
```

`Post-CIS-Hardening` se creó con las VMs apagadas para mantener un punto consistente posterior a las remediaciones.

---

## 27. Validación final

Ejecutar desde `cp01`:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployment,service -n payments
kubectl get servicemonitor -A
kubectl get prometheusrule -n monitoring
kubectl get pvc -A
```

Estado esperado:

- `cp01` → `Ready`;
- `w01` → `Ready`;
- `w02` → `Ready`;
- KubePay → `1/1`;
- Prometheus → `Running`;
- Grafana → `Running`;
- Alertmanager → `Running`;
- ServiceMonitor de KubePay presente;
- `proyecto6-alert-rules` cargado.

---

## 28. Consideraciones y límites del laboratorio

Este entorno es académico y utiliza decisiones orientadas a reproducibilidad y demostración:

- `firewalld` está deshabilitado;
- SELinux opera en modo permisivo;
- KubePay utiliza una imagen pública de Python;
- el acceso a las interfaces de monitoreo se realiza mediante `port-forward`;
- el almacenamiento utiliza Local Path Provisioner;
- las VMs comparten un único host físico.

Estas decisiones deben considerarse límites del laboratorio y no una arquitectura de producción.

---

## 29. Documentación relacionada

```text
README.md
docs/troubleshooting.md
docs/runbooks.md
docs/iso27001.md
dashboards/
manifests/kubepay/
manifests/alerts/
```

La carpeta de evidencias conserva las capturas de Kubernetes, Grafana, Prometheus, Alertmanager y kube-bench utilizadas para validar el proyecto.
