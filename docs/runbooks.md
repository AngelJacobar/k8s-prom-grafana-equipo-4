# Proyecto 6 — Runbooks de alertas

Este documento define los procedimientos de atención para las 10 reglas de alerta del Proyecto 6.

## Resumen de reglas

| # | Alerta | Severidad | Umbral / condición |
|---|---|---|---|
| 1 | `P6NodeDown` | critical | `node-exporter` sin responder durante 1 minuto |
| 2 | `P6HostHighCpuLoad` | warning | CPU > 80% durante 5 minutos |
| 3 | `P6HostMemoryPressure` | warning | Memoria > 90% durante 5 minutos |
| 4 | `P6DiskSpaceFillingUp` | warning | Disco raíz > 85% durante 5 minutos |
| 5 | `P6KubeletDown` | critical | Kubelet sin responder durante 1 minuto |
| 6 | `P6PodCrashLooping` | warning | CrashLoopBackOff reciente + al menos 2 reinicios en 10 min; condición sostenida 30 s |
| 7 | `P6PodOOMKilled` | critical | Reinicio reciente cuyo último motivo fue `OOMKilled` |
| 8 | `P6KubePayUnavailable` | critical | Target de KubePay sin responder durante 1 minuto |
| 9 | `P6KubePayHighDeclineRate` | warning | > 30% de pagos rechazados en 5 min, con mínimo 5 pagos |
| 10 | `P6KubePayReplicasMismatch` | warning | Réplicas disponibles < réplicas deseadas durante 1 minuto |

> Los umbrales de CPU y memoria se definieron a partir de la línea base observada en el laboratorio. CPU se mantiene normalmente por debajo de ~30% y `cp01` presenta el mayor uso de memoria, alrededor de 79–84%. Por ello se evita alertar por picos breves de CPU y se fija memoria en 90% para reducir falsos positivos.

---

## 1. P6NodeDown

**Severidad:** critical  
**Condición:** `up{job="node-exporter"} == 0` durante 1 minuto.

### Justificación
Un minuto permite distinguir una falla real de un scrape aislado. La pérdida de `node-exporter` puede indicar caída del nodo, problema de red o fallo del exporter.

### Diagnóstico
```bash
kubectl get nodes -o wide
kubectl get pods -n monitoring -o wide | grep node-exporter
```

Si el nodo responde por SSH:
```bash
systemctl status kubelet
ip addr
ip route
```

### Atención
1. Confirmar si el nodo está encendido y accesible.
2. Verificar conectividad entre Prometheus y el nodo.
3. Revisar el Pod de `node-exporter`.
4. Corregir la causa detectada antes de reiniciar componentes.

### Validación
```bash
kubectl get nodes
kubectl get pods -n monitoring | grep node-exporter
```

En Prometheus, comprobar que el target vuelva a `UP` y que la alerta se resuelva.

---

## 2. P6HostHighCpuLoad

**Severidad:** warning  
**Condición:** CPU > 80% durante 5 minutos.

### Justificación
La línea base del laboratorio se mantiene normalmente por debajo de ~30%. Se observaron picos breves cercanos a 80–90%, por lo que se exige una duración de 5 minutos para evitar alertas por picos transitorios.

### Diagnóstico
```bash
top
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
kubectl top pods -A --sort-by=cpu
```

### Atención
1. Identificar el proceso o Pod que consume CPU.
2. Revisar si el incremento corresponde a una carga legítima.
3. Si es un workload, revisar requests/limits.
4. Escalar o reiniciar únicamente si la causa lo justifica.

### Validación
Confirmar en Grafana/Prometheus que el uso de CPU vuelva por debajo del umbral y que la alerta pase a estado resuelto.

---

## 3. P6HostMemoryPressure

**Severidad:** warning  
**Condición:** uso de memoria > 90% durante 5 minutos.

### Justificación
`cp01` presenta normalmente el mayor consumo de memoria, aproximadamente 79–84%. Un umbral de 80–85% generaría falsos positivos; 90% permite detectar presión real antes de agotar memoria.

### Diagnóstico
```bash
free -h
ps -eo pid,comm,%mem,%cpu --sort=-%mem | head
kubectl top pods -A --sort-by=memory
```

### Atención
1. Identificar procesos o Pods con mayor consumo.
2. Revisar crecimiento anómalo de memoria.
3. Validar requests/limits.
4. Reiniciar o escalar el componente solamente si la causa está confirmada.

### Validación
Confirmar que el uso de memoria se mantenga por debajo de 90% y que la alerta se resuelva.

---

## 4. P6DiskSpaceFillingUp

**Severidad:** warning  
**Condición:** uso de `/` > 85% durante 5 minutos.

### Justificación
El 85% proporciona margen para actuar antes de llegar a una condición crítica de almacenamiento.

### Diagnóstico
```bash
df -h /
du -xhd1 /var 2>/dev/null | sort -h
du -xhd1 / 2>/dev/null | sort -h
```

### Atención
1. Identificar el directorio que incrementó su consumo.
2. Revisar logs, imágenes y archivos temporales.
3. Eliminar únicamente archivos identificados como seguros de borrar.
4. Si el crecimiento es esperado, ampliar almacenamiento.

### Validación
```bash
df -h /
```

Confirmar que el uso quede debajo de 85% y que la alerta se resuelva.

---

## 5. P6KubeletDown

**Severidad:** critical  
**Condición:** `up{job="kubelet"} == 0` durante 1 minuto.

### Justificación
El kubelet es esencial para la operación de cada nodo. Un minuto filtra fallas breves de scrape y permite detectar rápidamente una pérdida sostenida.

### Diagnóstico
```bash
kubectl get nodes
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager
```

### Atención
1. Confirmar conectividad al nodo.
2. Revisar estado y logs de kubelet.
3. Validar containerd y conectividad con el control plane.
4. Reiniciar kubelet solo después de identificar la causa.

### Validación
```bash
systemctl is-active kubelet
kubectl get nodes
```

Confirmar target `kubelet` en estado `UP`.

---

## 6. P6PodCrashLooping

**Severidad:** warning  
**Condición:** el contenedor presentó `CrashLoopBackOff` en los últimos 5 minutos y acumuló al menos 2 reinicios en 10 minutos; la condición debe persistir 30 segundos.

### Justificación
La ventana temporal evita que la regla oscile entre `PENDING` e `INACTIVE` durante cada intento de reinicio. El mínimo de 2 reinicios diferencia un fallo aislado de un bucle real.

### Diagnóstico
```bash
kubectl get pods -A
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
```

### Atención
1. Revisar el error de la última ejecución.
2. Validar variables, ConfigMaps, Secrets, volúmenes y command/args.
3. Corregir la causa de terminación.
4. Volver a desplegar el workload.

### Validación
```bash
kubectl get pod <pod> -n <namespace>
```

El Pod debe quedar `Running` y el contador de reinicios debe estabilizarse.

**Prueba del laboratorio:** se utilizó el Pod temporal `p6-crashloop-demo` para demostrar la alerta sin afectar KubePay.

---

## 7. P6PodOOMKilled

**Severidad:** critical  
**Condición:** contenedor reiniciado recientemente cuyo último motivo de terminación fue `OOMKilled`.

### Justificación
`OOMKilled` indica que el contenedor excedió la memoria disponible o su límite, lo que puede degradar la disponibilidad del servicio.

### Diagnóstico
```bash
kubectl get pods -A
kubectl describe pod <pod> -n <namespace>
kubectl top pod <pod> -n <namespace>
```

### Atención
1. Confirmar el motivo `OOMKilled`.
2. Revisar consumo real y límites configurados.
3. Investigar fugas de memoria o picos de carga.
4. Ajustar límites solo si existe evidencia que lo justifique.

### Validación
Confirmar que no aparezcan nuevos `OOMKilled`, que el Pod permanezca `Running` y que la alerta se resuelva.

---

## 8. P6KubePayUnavailable

**Severidad:** critical  
**Condición:** target `job="kubepay"` sin responder durante 1 minuto.

### Justificación
KubePay es el workload principal del laboratorio. Una pérdida sostenida de su endpoint de métricas puede acompañar indisponibilidad del servicio o fallas de conectividad.

### Diagnóstico
```bash
kubectl get deployment kubepay -n payments
kubectl get pods -n payments -o wide
kubectl get svc kubepay -n payments
kubectl get servicemonitors -n monitoring
```

### Atención
1. Verificar Pod, Service y ServiceMonitor.
2. Revisar logs de KubePay.
3. Confirmar conectividad al puerto 8080.
4. Recuperar el Deployment si está degradado.

### Validación
```bash
kubectl get deployment kubepay -n payments
kubectl get pods -n payments
```

Además, comprobar en Prometheus que `up{job="kubepay"}` regrese a `1`.

---

## 9. P6KubePayHighDeclineRate

**Severidad:** warning  
**Condición:** más del 30% de los pagos de los últimos 5 minutos tienen `result="declined"`, con al menos 5 transacciones observadas, durante 1 minuto.

### Justificación
El mínimo de 5 pagos evita que una única transacción rechazada genere una tasa engañosa del 100%. El umbral de 30% permite detectar un incremento significativo de rechazos en el contexto del laboratorio.

### Diagnóstico
En Prometheus:
```promql
sum by (result) (increase(kubepay_payments_total[5m]))
```

Calcular la tasa:
```promql
100 *
sum(increase(kubepay_payments_total{result="declined"}[5m]))
/
clamp_min(sum(increase(kubepay_payments_total[5m])), 1)
```

### Atención
1. Confirmar que exista volumen suficiente de transacciones.
2. Comparar `approved` vs `declined`.
3. Revisar si los rechazos responden a datos de entrada esperados o a un cambio anómalo.
4. Revisar logs y comportamiento de KubePay.

### Validación
Generar o esperar tráfico válido y comprobar que la tasa caiga por debajo de 30%. La alerta debe resolverse al salir la condición de la ventana de 5 minutos.

**Prueba del laboratorio:** se generaron 5 transacciones, 3 rechazadas y 2 aprobadas, obteniendo una tasa de rechazo del 60% y la alerta llegó a `FIRING` y a Alertmanager.

---

## 10. P6KubePayReplicasMismatch

**Severidad:** warning  
**Condición:** réplicas disponibles de KubePay inferiores a las réplicas deseadas durante 1 minuto.

### Justificación
La regla compara el estado real contra el deseado en lugar de fijar un número estático de réplicas, por lo que sigue siendo válida si el Deployment cambia de escala.

### Diagnóstico
```bash
kubectl get deployment kubepay -n payments
kubectl get pods -n payments -o wide
kubectl describe deployment kubepay -n payments
```

### Atención
1. Identificar por qué una réplica no puede programarse o iniciar.
2. Revisar nodeSelector, recursos disponibles y eventos.
3. Recuperar capacidad o corregir la configuración.
4. Confirmar que el número `AVAILABLE` alcance el número deseado.

### Validación
```bash
kubectl get deployment kubepay -n payments
```

`READY` y `AVAILABLE` deben coincidir con las réplicas deseadas.

**Prueba del laboratorio:** se aplicó `cordon` a `w02` y se escaló temporalmente KubePay de 1 a 2 réplicas. La segunda quedó `Pending`, la alerta llegó a `FIRING` y a Alertmanager. Después se restauró KubePay a 1 réplica y se ejecutó `uncordon w02`.

---

## Evidencia de las tres alertas demostradas

Las tres alertas seleccionadas para la demostración fueron:

1. `P6KubePayHighDeclineRate`
2. `P6KubePayReplicasMismatch`
3. `P6PodCrashLooping`

Para cada una se debe conservar evidencia de:

- condición provocada;
- estado `FIRING` en Prometheus;
- recepción de la alerta en Alertmanager;
- recuperación o limpieza del escenario de prueba.

