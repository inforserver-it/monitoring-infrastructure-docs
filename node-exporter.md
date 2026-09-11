# Node Exporter — Métricas Linux

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Prometheus Node Exporter es el agente utilizado para exponer métricas del
sistema operativo Linux.

Node Exporter no almacena métricas ni proporciona dashboards.

Su función es:

```text
leer métricas Linux
       ↓
exponerlas mediante HTTP
       ↓
Prometheus realiza scrape
```

Arquitectura:

```text
Servidor Linux
      │
      ▼
Node Exporter :9100
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

---

## 2. Servidores actuales

Node Exporter se utiliza en:

```text
lab01
python
security
mail01
```

No se utiliza en:

```text
mqh01
```

---

## 3. Estado

| Servidor | Entorno | Node Exporter | Prometheus |
|---|---|---|---|
| lab01 | Laboratorio | OPERATIVO | UP |
| python | Laboratorio | OPERATIVO | UP |
| security | Laboratorio | OPERATIVO | UP |
| mail01 | Producción | OPERATIVO | UP |
| mqh01 | Producción | NO INSTALADO | CloudWatch + YACE |

---

# FUNCIONAMIENTO

---

## 4. Puerto

Puerto estándar:

```text
TCP 9100
```

Endpoint:

```text
http://IP_SERVIDOR:9100/metrics
```

Ejemplo:

```text
http://192.168.1.101:9100/metrics
```

---

## 5. Modelo Pull

Node Exporter no envía activamente las métricas a Prometheus.

Prometheus realiza:

```text
HTTP GET
```

contra:

```text
/metrics
```

Modelo:

```text
Prometheus
    │
    │ HTTP GET
    ▼
Node Exporter :9100
```

Este mecanismo se denomina:

```text
scrape
```

---

## 6. Métricas disponibles

Node Exporter proporciona información sobre:

```text
CPU
RAM
almacenamiento
filesystems
Load Average
uptime
interfaces de red
tráfico de entrada
tráfico de salida
hostname
kernel
arquitectura
sistema operativo
```

---

# SERVIDORES

---

## 7. lab01

```text
Servidor:       lab01
Entorno:        Laboratorio
IP:             192.168.1.101
Endpoint:       192.168.1.101:9100
```

Prometheus:

```text
job="lab-servers"
server="lab01"
```

Estado:

```text
Node Exporter   OPERATIVO
Prometheus      UP
```

---

## 8. python

```text
Servidor:       python
Entorno:        Laboratorio
IP:             192.168.1.104
Endpoint:       192.168.1.104:9100
```

Prometheus:

```text
job="lab-servers"
server="python"
```

Estado:

```text
Node Exporter   OPERATIVO
Prometheus      UP
```

---

## 9. security

```text
Servidor:       security
Entorno:        Laboratorio
IP:             192.168.1.103
Endpoint:       192.168.1.103:9100
```

Prometheus:

```text
job="lab-servers"
server="security"
```

Estado:

```text
Node Exporter   OPERATIVO
Prometheus      UP
```

---

# MAIL01

---

## 10. Identificación

`mail01` es un servidor de producción.

```text
Servidor:       mail01
Proveedor:      Contabo
Entorno:        Producción
IP:             5.189.147.4
Endpoint:       5.189.147.4:9100
```

Prometheus:

```text
job="lab-servers"
server="mail01"
```

Aunque el job conserve el nombre histórico:

```text
lab-servers
```

`mail01` NO es laboratorio.

---

## 11. Instalación

Node Exporter fue instalado mediante el paquete Debian:

```text
prometheus-node-exporter
```

La instalación se realizó mediante Ansible.

Comando utilizado:

```bash
ansible mail01 -b -K -m apt -a "name=prometheus-node-exporter state=present update_cache=yes"
```

---

## 12. Servicio

Servicio:

```text
prometheus-node-exporter.service
```

Estado esperado:

```text
active
enabled
```

Esto significa:

```text
servicio funcionando
        +
arranque automático
```

---

## 13. Puerto

`mail01` escucha:

```text
*:9100
```

Prometheus puede acceder correctamente al exporter.

Estado:

```text
mail01 → UP
```

---

# MÉTRICAS

---

## 14. CPU

Métrica principal:

```text
node_cpu_seconds_total
```

Grafana calcula el uso de CPU a partir del tiempo en modo:

```text
idle
```

Consulta utilizada actualmente:

```promql
100 - (
  avg by (server) (
    rate(node_cpu_seconds_total{
      mode="idle",
      server="$instance",
      job="lab-servers"
    }[5m])
  ) * 100
)
```

En Grafana esta consulta forma parte de una expresión combinada con las
métricas AWS de `mqh01`.

---

## 15. Número de CPU

Puede determinarse utilizando las etiquetas `cpu` presentes en:

```text
node_cpu_seconds_total
```

La consulta utilizada en el panel de información es:

```promql
count(
  count by (cpu) (
    node_cpu_seconds_total{
      server="$instance",
      job="lab-servers"
    }
  )
)
```

---

# MEMORIA

---

## 16. Métricas

```text
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
```

---

## 17. Uso de RAM

Consulta:

```promql
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes{
      server="$instance",
      job="lab-servers"
    }
    /
    node_memory_MemTotal_bytes{
      server="$instance",
      job="lab-servers"
    }
  )
)
```

---

## 18. RAM total

Para obtener la RAM total:

```promql
node_memory_MemTotal_bytes{
  server="$instance",
  job="lab-servers"
}
```

Grafana convierte posteriormente el resultado a GB cuando corresponde.

---

# DISCO

---

## 19. Métricas

```text
node_filesystem_size_bytes
node_filesystem_avail_bytes
```

---

## 20. Filesystem raíz

El dashboard utiliza:

```text
mountpoint="/"
```

y excluye:

```text
tmpfs
overlay
```

Consulta de uso:

```promql
100 * (
  1 -
  (
    node_filesystem_avail_bytes{
      server="$instance",
      job="lab-servers",
      mountpoint="/",
      fstype!~"tmpfs|overlay"
    }
    /
    node_filesystem_size_bytes{
      server="$instance",
      job="lab-servers",
      mountpoint="/",
      fstype!~"tmpfs|overlay"
    }
  )
)
```

---

## 21. Tamaño total

```promql
node_filesystem_size_bytes{
  server="$instance",
  job="lab-servers",
  mountpoint="/",
  fstype!~"tmpfs|overlay"
}
```

---

# LOAD AVERAGE

---

## 22. Métricas

```text
node_load1
node_load5
node_load15
```

Representan:

```text
node_load1     → 1 minuto
node_load5     → 5 minutos
node_load15    → 15 minutos
```

---

## 23. Consultas

1 minuto:

```promql
node_load1{
  server="$instance",
  job="lab-servers"
}
```

5 minutos:

```promql
node_load5{
  server="$instance",
  job="lab-servers"
}
```

15 minutos:

```promql
node_load15{
  server="$instance",
  job="lab-servers"
}
```

---

# UPTIME

---

## 24. Métricas

```text
node_time_seconds
node_boot_time_seconds
```

Cálculo:

```promql
node_time_seconds{
  server="$instance",
  job="lab-servers"
}
-
node_boot_time_seconds{
  server="$instance",
  job="lab-servers"
}
```

El resultado está expresado en:

```text
segundos
```

---

# ESTADO

---

## 25. Métrica up

Prometheus proporciona:

```text
up
```

Para los servidores Node Exporter:

```promql
up{
  server="$instance",
  job="lab-servers"
}
```

Valores:

```text
1 → UP
0 → DOWN
```

---

# RED

---

## 26. Métricas

Entrada:

```text
node_network_receive_bytes_total
```

Salida:

```text
node_network_transmit_bytes_total
```

---

## 27. Loopback

El dashboard excluye:

```text
device="lo"
```

porque el tráfico loopback no representa tráfico de red externo del servidor.

---

## 28. Entrada

Consulta:

```promql
sum(
  rate(node_network_receive_bytes_total{
    server="$instance",
    job="lab-servers",
    device!="lo"
  }[1m])
)
```

---

## 29. Salida

Consulta:

```promql
sum(
  rate(node_network_transmit_bytes_total{
    server="$instance",
    job="lab-servers",
    device!="lo"
  }[1m])
)
```

El resultado representa:

```text
bytes por segundo
```

---

# INFORMACIÓN DEL SISTEMA

---

## 30. node_uname_info

Métrica:

```text
node_uname_info
```

Proporciona información como:

```text
instance
nodename
sysname
release
machine
```

Grafana utiliza esta métrica en el panel:

```text
Información
```

---

# MQH01

---

## 31. Situación actual

`mqh01` es una instancia AWS EC2 de producción.

Actualmente:

```text
Node Exporter              NO INSTALADO
Puerto 9100                NO UTILIZADO
Prometheus directo         NO
CloudWatch                 OPERATIVO
YACE                       OPERATIVO
Prometheus vía YACE        OPERATIVO
Grafana                    OPERATIVO
```

---

## 32. Arquitectura definitiva

Las métricas de `mqh01` utilizan:

```text
mqh01
  ↓
AWS CloudWatch
  ↓
YACE
  ↓
Prometheus
  ↓
Grafana
```

No debe añadirse `mqh01` al job:

```text
lab-servers
```

como target Node Exporter.

Su job es:

```text
aws-mqh01
```

---

# HISTÓRICO DE LA PRUEBA MQH01

---

## 33. Intento mediante APT

Durante las pruebas iniciales se intentó instalar:

```text
prometheus-node-exporter
```

mediante APT.

La instalación fue detenida por:

```text
apt-listbugs
```

debido a un bug grave relacionado con:

```text
openipmi
```

Se decidió no forzar la instalación.

---

## 34. Prueba mediante binario

Para determinar si Node Exporter funcionaba correctamente en el sistema se
realizó una instalación temporal mediante el binario oficial.

Se utilizó:

```text
/usr/local/bin/node_exporter
```

Usuario:

```text
node_exporter
```

Servicio temporal:

```text
/etc/systemd/system/node_exporter.service
```

---

## 35. Resultado de la prueba

Node Exporter funcionó correctamente dentro de `mqh01`.

Se confirmó:

```text
servicio             active
servicio             enabled
puerto               9100 LISTEN
/metrics              operativo
```

También respondió mediante la IP privada EC2.

Por tanto, el problema no era Node Exporter.

---

## 36. Problema real

Prometheus en Fedora44 debía acceder desde Internet al puerto:

```text
TCP 9100
```

de la instancia EC2.

Fedora44 utiliza una IP pública dinámica.

Esto hacía poco adecuada una regla AWS basada permanentemente en:

```text
IP/32
```

---

## 37. Pruebas realizadas

Durante el diagnóstico se revisaron:

```text
AWS Security Groups
nftables
iptables
CrowdSec
conectividad externa
```

Se utilizaron temporalmente reglas relacionadas con:

```text
TCP 9100
```

---

## 38. Exposición temporal

Durante una prueba de diagnóstico llegó a existir temporalmente una regla:

```text
TCP 9100
0.0.0.0/0
```

Esta configuración fue exclusivamente temporal.

Fue eliminada.

No debe reproducirse como configuración permanente.

---

## 39. Decisión

Se decidió:

```text
NO utilizar Node Exporter en mqh01
```

porque CloudWatch proporciona una arquitectura más adecuada para esta
instancia AWS.

Ventajas:

```text
no exponer 9100
no depender de IP pública dinámica
reducir superficie de ataque
integración nativa AWS
simplificar conectividad
```

---

## 40. Reversión

Después de las pruebas se eliminó:

```text
/usr/local/bin/node_exporter
/etc/systemd/system/node_exporter.service
usuario node_exporter
archivos temporales
reglas firewall 9100
regla Security Group 9100
```

Estado final:

```text
Node Exporter      NO INSTALADO
Puerto 9100        NO UTILIZADO
```

---

# SEGURIDAD

---

## 41. Exposición

Node Exporter expone información operativa del sistema.

No debe exponerse indiscriminadamente a Internet.

Especialmente:

```text
TCP 9100
```

debe limitarse a los sistemas que realmente necesiten consultar el exporter.

---

## 42. Autenticación

Node Exporter no debe considerarse protegido simplemente porque:

```text
/metrics
```

no parezca contener información sensible a primera vista.

Puede revelar:

```text
hostname
kernel
CPU
memoria
discos
interfaces
filesystem
actividad
```

---

## 43. Regla permanente para mqh01

No volver a abrir:

```text
TCP 9100
```

en `mqh01` para la arquitectura actual.

Las métricas ya se obtienen mediante:

```text
CloudWatch → YACE
```

---

# ANSIBLE

---

## 44. Instalación Debian

Para un nuevo servidor Debian compatible puede utilizarse:

```bash
ansible SERVIDOR -b -K -m apt -a "name=prometheus-node-exporter state=present update_cache=yes"
```

Después:

```bash
ansible SERVIDOR -b -K -m systemd -a "name=prometheus-node-exporter state=started enabled=yes"
```

No ejecutar automáticamente sobre producción sin revisar previamente el
servidor.

---

## 45. Nuevo servidor

Antes de incorporar Node Exporter:

```text
1. identificar entorno
2. comprobar sistema operativo
3. comprobar método de instalación
4. revisar firewall
5. revisar conectividad
6. instalar exporter
7. habilitar servicio
8. permitir únicamente acceso necesario a 9100
9. añadir target Prometheus
10. asignar etiqueta server
11. confirmar UP
12. incorporar a Grafana
13. documentar
```

---

# DIAGNÓSTICO

---

## 46. Servicio

En Debian:

```bash
systemctl status prometheus-node-exporter --no-pager
```

Actividad:

```bash
systemctl is-active prometheus-node-exporter
```

Arranque:

```bash
systemctl is-enabled prometheus-node-exporter
```

---

## 47. Puerto

```bash
ss -lnt | grep ':9100'
```

---

## 48. Endpoint local

```bash
curl -s http://127.0.0.1:9100/metrics | head
```

Una respuesta normal contiene:

```text
# HELP
# TYPE
node_...
```

---

## 49. Endpoint remoto

Desde una máquina autorizada:

```bash
curl -s http://IP_SERVIDOR:9100/metrics | head
```

Si funciona localmente pero no remotamente, investigar:

```text
red
firewall
routing
Security Group
conectividad
```

antes de reinstalar Node Exporter.

---

# RELACIÓN CON OTROS COMPONENTES

---

## 50. Prometheus

```text
Node Exporter
      ↓
Prometheus
```

Prometheus realiza el scrape.

---

## 51. Grafana

```text
Node Exporter
      ↓
Prometheus
      ↓
Grafana
```

Grafana no consulta directamente Node Exporter.

---

## 52. Loki

Node Exporter:

```text
NO recopila logs
```

Para logs:

```text
Alloy → Loki → Grafana
```

En `mqh01`:

```text
CloudWatch Logs → Grafana
```

---

## 53. YACE

YACE no sustituye Node Exporter de forma general.

Se utiliza específicamente para exponer a Prometheus las métricas AWS
necesarias.

Actualmente:

```text
lab01
python
security
mail01
        ↓
Node Exporter

mqh01
        ↓
CloudWatch
        ↓
YACE
```

---

# ESTADO ACTUAL

---

## 54. lab01

```text
Node Exporter              OPERATIVO
Prometheus                 UP
```

---

## 55. python

```text
Node Exporter              OPERATIVO
Prometheus                 UP
```

---

## 56. security

```text
Node Exporter              OPERATIVO
Prometheus                 UP
```

---

## 57. mail01

```text
Node Exporter              OPERATIVO
Prometheus                 UP
Entorno                    PRODUCCIÓN
```

---

## 58. mqh01

```text
Node Exporter              NO INSTALADO
Puerto 9100                NO UTILIZADO
CloudWatch                 OPERATIVO
CloudWatch Agent           OPERATIVO
YACE                       OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO
Entorno                    PRODUCCIÓN
```

---

## 59. Regla final

La arquitectura actual es:

```text
lab01 ──────┐
python ─────┤
security ───┼── Node Exporter ── Prometheus ── Grafana
mail01 ─────┘


mqh01
  │
  ▼
CloudWatch
  │
  ▼
YACE
  │
  ▼
Prometheus
  │
  ▼
Grafana
```

No existe actualmente ninguna necesidad de modificar esta separación.

---

## 60. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
alloy.md
logs.md
servidores.md
seguridad.md
recuperacion.md
```

`node-exporter.md` documenta las métricas Linux y los servidores que utilizan
Node Exporter.

`prometheus.md` documenta el scraping y las consultas combinadas.

`servidores.md` documenta cada servidor.

`seguridad.md` documenta las decisiones relacionadas con la exposición de
servicios.

`recuperacion.md` contiene el procedimiento de recuperación.
