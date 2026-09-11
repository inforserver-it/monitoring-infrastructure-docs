# Servidores — Plataforma de Monitorización

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Este documento mantiene la información operativa de los servidores incluidos
en la plataforma de monitorización.

Actualmente existen cinco servidores:

```text
lab01
python
security
mail01
mqh01
```

Se dividen en dos grupos:

```text
LABORATORIO
├── lab01
├── python
└── security

PRODUCCIÓN
├── mail01
└── mqh01
```

Esta separación debe respetarse en cualquier cambio, prueba, despliegue o
automatización.

---

## 2. Resumen

| Servidor | Entorno | Plataforma | Métricas | Logs |
|---|---|---|---|---|
| lab01 | Laboratorio | Linux | Node Exporter | Loki |
| python | Laboratorio | Linux | Node Exporter | Loki |
| security | Laboratorio | Linux | Node Exporter | Loki |
| mail01 | Producción | Contabo | Node Exporter | Loki |
| mqh01 | Producción | AWS EC2 | CloudWatch + YACE | CloudWatch Logs |

---

# LABORATORIO

---

## 3. lab01

Servidor principal de laboratorio.

### Identificación

```text
Nombre:       lab01
Entorno:      Laboratorio
IP:           192.168.1.101
```

### Métricas

Utiliza:

```text
Node Exporter
```

Endpoint:

```text
192.168.1.101:9100
```

Prometheus:

```text
job="lab-servers"
server="lab01"
```

Flujo:

```text
lab01
  ↓
Node Exporter
  ↓
Prometheus
  ↓
Grafana
```

### Logs

Los registros integrados con la plataforma utilizan:

```text
Grafana Alloy
      ↓
     Loki
      ↓
   Grafana
```

Las consultas utilizan:

```text
server="lab01"
```

---

## 4. python

Servidor de laboratorio destinado a trabajos relacionados con Python y
desarrollo.

### Identificación

```text
Nombre:       python
Entorno:      Laboratorio
IP:           192.168.1.104
```

### Métricas

Utiliza:

```text
Node Exporter
```

Endpoint:

```text
192.168.1.104:9100
```

Prometheus:

```text
job="lab-servers"
server="python"
```

Flujo:

```text
python
  ↓
Node Exporter
  ↓
Prometheus
  ↓
Grafana
```

### Logs

Los registros centralizados utilizan:

```text
Grafana Alloy
      ↓
     Loki
      ↓
   Grafana
```

Etiqueta:

```text
server="python"
```

---

## 5. security

Servidor de laboratorio orientado a seguridad y pruebas.

### Identificación

```text
Nombre:       security
Entorno:      Laboratorio
IP:           192.168.1.103
```

### Métricas

Utiliza:

```text
Node Exporter
```

Endpoint:

```text
192.168.1.103:9100
```

Prometheus:

```text
job="lab-servers"
server="security"
```

Flujo:

```text
security
   ↓
Node Exporter
   ↓
Prometheus
   ↓
Grafana
```

### Logs

Utiliza la arquitectura:

```text
Grafana Alloy
      ↓
     Loki
      ↓
   Grafana
```

Etiqueta:

```text
server="security"
```

---

# PRODUCCIÓN

---

## 6. mail01

`mail01` es un servidor de producción.

No debe tratarse como servidor de laboratorio ni incluirse automáticamente en
pruebas o despliegues destinados a los servidores de laboratorio.

### Identificación

```text
Nombre:       mail01
Entorno:      Producción
Proveedor:    Contabo
IP:           5.189.147.4
```

---

## 7. Métricas de mail01

Utiliza:

```text
Node Exporter
```

Endpoint:

```text
5.189.147.4:9100
```

Prometheus:

```text
job="lab-servers"
server="mail01"
```

Aunque el nombre histórico del job sea:

```text
lab-servers
```

`mail01` es un servidor de producción.

El nombre del job no determina el entorno operativo del servidor.

Flujo:

```text
mail01
  ↓
Node Exporter
  ↓
Prometheus
  ↓
Grafana
```

---

## 8. Logs de mail01

Los registros integrados con la plataforma utilizan:

```text
mail01
   ↓
Grafana Alloy
   ↓
Loki
   ↓
Grafana
```

Grafana utiliza:

```text
server="mail01"
```

junto con:

```text
log_type
```

para seleccionar los registros.

---

## 9. Seguridad SSH de mail01

`mail01` recibe numerosos intentos automáticos de autenticación SSH.

Fail2ban está operativo.

Durante la revisión de seguridad se comprobó inicialmente:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
```

El valor:

```text
passwordauthentication yes
```

no procedía de la configuración principal esperada.

---

## 10. Override de cloud-init

El archivo:

```text
/etc/ssh/sshd_config
```

ya contenía:

```text
PasswordAuthentication no
```

Sin embargo, existía:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

con:

```text
PasswordAuthentication yes
```

Este drop-in estaba modificando el comportamiento efectivo de OpenSSH.

Se corrigió para utilizar:

```text
PasswordAuthentication no
```

---

## 11. Configuración SSH efectiva de mail01

Después del cambio se verificó:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Estado:

```text
Root por SSH                 DESACTIVADO
Contraseña SSH               DESACTIVADA
Keyboard interactive         DESACTIVADO
Clave pública SSH            ACTIVADA
```

Después del cambio se realizó una nueva conexión desde Fedora44 mediante:

```bash
ssh mail
```

La conexión mediante clave pública funcionó correctamente.

---

## 12. Fail2ban de mail01

Jail:

```text
sshd
```

Fail2ban permanece activo y bloquea direcciones que realizan intentos
repetidos de autenticación.

Durante la revisión se observaron cientos de miles de intentos fallidos
acumulados y decenas de miles de bloqueos históricos.

Esto refleja principalmente actividad automatizada de Internet contra el
servicio SSH.

La presencia de intentos fallidos no implica por sí misma un acceso exitoso.

---

# MQH01

---

## 13. mqh01

`mqh01` es un servidor de producción alojado en AWS.

Aloja infraestructura relacionada con:

```text
masqhosting.com
```

Debe tratarse siempre como producción.

No debe utilizarse para pruebas generales.

---

## 14. Identificación de mqh01

```text
Nombre:          mqh01
Hostname:        masqhosting
Entorno:         Producción
Proveedor:       AWS
Servicio:        EC2
Región:          eu-south-2
Instance ID:     i-0d4beab843b2d2f95
Sistema:         Debian 13
Kernel:          6.12.107+deb13-cloud-amd64
CPU:             2
RAM:             ~1.9 GB
Disco raíz:      ~29.3 GB
```

---

## 15. Arquitectura de métricas de mqh01

`mqh01` no utiliza Node Exporter.

Arquitectura:

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

Prometheus:

```text
job="aws-mqh01"
server="mqh01"
```

---

## 16. Decisión sobre Node Exporter en mqh01

Durante el diseño de la monitorización se probó Node Exporter en `mqh01`.

El exporter funcionaba localmente.

Sin embargo, para que Fedora44 pudiera realizar el scrape era necesario
resolver el acceso externo a:

```text
TCP 9100
```

Fedora44 utiliza una IP pública dinámica.

Se decidió no mantener esta arquitectura para evitar:

```text
exposición innecesaria
reglas dependientes de IP dinámica
mayor superficie de ataque
configuración adicional
```

La prueba de Node Exporter fue revertida.

La arquitectura definitiva es:

```text
CloudWatch → YACE → Prometheus
```

---

## 17. CloudWatch de mqh01

AWS proporciona directamente métricas EC2 como:

```text
CPUUtilization
StatusCheckFailed
NetworkIn
NetworkOut
```

Estas métricas son descubiertas por YACE.

---

## 18. CloudWatch Agent

`mqh01` utiliza:

```text
Amazon CloudWatch Agent
```

Versión instalada durante la configuración:

```text
1.300069.0b1529
```

El agente recopila métricas adicionales del sistema.

---

## 19. Memoria de mqh01

Namespace:

```text
mqh01-memory
```

Métricas:

```text
mem_used_percent
mem_total
```

YACE las expone a Prometheus como:

```text
aws_cwagent_mem_used_percent_average
aws_cwagent_mem_total_average
```

---

## 20. Disco de mqh01

Namespace:

```text
mqh01-disk
```

Métricas:

```text
disk_used_percent
disk_total
```

Filesystem principal:

```text
/
```

Dimensiones utilizadas:

```text
InstanceId
device
fstype
path
```

---

## 21. Load Average y uptime de mqh01

Namespace:

```text
mqh01-system
```

Métricas:

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

Estas métricas son generadas mediante:

```text
/usr/local/bin/mqh01-system-metrics
```

y enviadas mediante StatsD.

---

## 22. StatsD

CloudWatch Agent escucha localmente en:

```text
127.0.0.1:8125/UDP
```

El script de métricas se ejecuta:

```text
cada minuto
```

mediante:

```text
/etc/cron.d/mqh01-system-metrics
```

---

## 23. Dimensiones StatsD

Las métricas personalizadas utilizan:

```text
InstanceId
metric_type
```

Valor:

```text
metric_type=gauge
```

YACE debe contemplar ambas dimensiones.

Este detalle es necesario para que las métricas:

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

sean descubiertas correctamente.

---

## 24. Red de mqh01

CloudWatch proporciona:

```text
NetworkIn
NetworkOut
```

Configuración:

```text
Statistic: Sum
Period: 300 segundos
```

Grafana convierte el resultado a bytes por segundo mediante:

```text
NetworkIn / 300
NetworkOut / 300
```

---

# LOGS DE MQH01

---

## 25. Arquitectura

Los logs de `mqh01` no se envían a Loki.

Arquitectura:

```text
mqh01
  ↓
AWS CloudWatch Logs
  ↓
Grafana
```

Esto evita duplicar innecesariamente los registros.

---

## 26. Logs disponibles

Actualmente:

```text
apache
ssh
fail2ban
errors
kernel
```

Log groups:

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

---

## 27. Apache

WordPress funciona mediante Docker.

Apache envía:

```text
stdout
stderr
```

a CloudWatch.

Log group:

```text
/masqhosting/mqh01/wordpress/apache
```

Stream:

```text
mqh01-wordpress
```

---

## 28. SSH

rsyslog genera:

```text
/var/log/ssh.log
```

Configuración:

```text
/etc/rsyslog.d/30-masqhosting-monitoring.conf
```

Regla:

```text
if ($programname == 'sshd' or $programname == 'sshd-session') then /var/log/ssh.log
```

CloudWatch Agent recopila posteriormente este archivo.

---

## 29. Errores

Archivo:

```text
/var/log/errors.log
```

Regla rsyslog:

```text
*.err /var/log/errors.log
```

CloudWatch:

```text
/masqhosting/mqh01/errors
```

---

## 30. Fail2ban

Archivo:

```text
/var/log/fail2ban.log
```

CloudWatch:

```text
/masqhosting/mqh01/fail2ban
```

---

## 31. Kernel

Archivo:

```text
/var/log/kern.log
```

CloudWatch:

```text
/masqhosting/mqh01/kernel
```

---

## 32. Retención de mqh01

```text
apache        30 días
kernel        30 días
errors        90 días
fail2ban      90 días
ssh           90 días
```

---

# GRAFANA

---

## 33. Selector de servidores

Grafana muestra los servidores en este orden:

```text
lab01
python
security
mail01
mqh01
```

Variable:

```text
instance
```

---

## 34. Dashboard

Dashboard:

```text
Monitorización Laboratorios
```

Aunque conserva este nombre histórico, actualmente incluye también servidores
de producción:

```text
mail01
mqh01
```

El dashboard permite visualizar:

```text
Información
CPU
RAM
Disco
Load Average
Uptime
Estado
Red
Logs
```

---

## 35. Diferencias ocultadas por Grafana

Para el usuario del dashboard:

```text
lab01
python
security
mail01
mqh01
```

se comportan como servidores seleccionables de la misma forma.

Internamente:

```text
lab01      ─┐
python      ├─ Node Exporter → Prometheus
security    │
mail01     ─┘

mqh01 → CloudWatch → YACE → Prometheus
```

Grafana combina las métricas mediante PromQL.

---

# PRODUCCIÓN

---

## 36. Regla de seguridad

Los servidores:

```text
mail01
mqh01
```

son producción.

No deben incluirse en:

```text
pruebas generales
despliegues de laboratorio
cambios experimentales
automatizaciones de prueba
reinicios masivos
```

salvo indicación explícita.

---

## 37. Cambios en producción

Antes de modificar producción debe determinarse:

```text
qué servidor se modifica
qué servicio se modifica
qué impacto puede producir
si requiere restart o únicamente reload
cómo revertir el cambio
```

Cuando un cambio únicamente afecta a Fedora44, Grafana, Prometheus o YACE,
no deben reiniciarse servicios de producción innecesariamente.

---

# ESTADO ACTUAL

---

## 38. lab01

```text
Entorno                    LABORATORIO
Node Exporter              OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO
Logs                       INTEGRADOS
```

---

## 39. python

```text
Entorno                    LABORATORIO
Node Exporter              OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO
Logs                       INTEGRADOS
```

---

## 40. security

```text
Entorno                    LABORATORIO
Node Exporter              OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO
Logs                       INTEGRADOS
```

---

## 41. mail01

```text
Entorno                    PRODUCCIÓN
Node Exporter              OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO
Logs                       INTEGRADOS
Fail2ban                   OPERATIVO
SSH root                   DESACTIVADO
SSH contraseña             DESACTIVADA
SSH clave pública          OPERATIVA
```

---

## 42. mqh01

```text
Entorno                    PRODUCCIÓN
AWS EC2                    OPERATIVO
CloudWatch                 OPERATIVO
CloudWatch Agent           OPERATIVO
YACE                       OPERATIVO
Prometheus                 UP
Grafana                    OPERATIVO

CPU                        OPERATIVA
RAM                        OPERATIVA
Disco                      OPERATIVO
Load 1m                    OPERATIVO
Load 5m                    OPERATIVO
Load 15m                   OPERATIVO
Uptime                     OPERATIVO
Estado                     OPERATIVO
NetworkIn                  OPERATIVO
NetworkOut                 OPERATIVO

CloudWatch Logs            OPERATIVO
Apache                     OPERATIVO
SSH                        OPERATIVO
Fail2ban                   OPERATIVO
Errors                     OPERATIVO
Kernel                     OPERATIVO
```

---

# INCORPORACIÓN DE NUEVOS SERVIDORES

---

## 43. Servidor de laboratorio

Antes de añadirlo se debe definir:

```text
hostname
IP
sistema operativo
función
Node Exporter
logs
etiqueta server
```

Si utiliza Node Exporter:

```text
Servidor
   ↓
Node Exporter
   ↓
Prometheus
   ↓
Grafana
```

---

## 44. Servidor de producción

Un servidor de producción debe añadirse explícitamente.

No debe heredarse automáticamente una configuración pensada para laboratorio.

Debe decidirse individualmente:

```text
métricas
logs
seguridad
acceso
retención
alertas
```

---

## 45. Servidor AWS

Para futuras instancias AWS puede reutilizarse el patrón de `mqh01`:

```text
EC2
 ↓
CloudWatch
 ↓
YACE
 ↓
Prometheus
 ↓
Grafana
```

y para logs:

```text
EC2
 ↓
CloudWatch Logs
 ↓
Grafana
```

Esto evita exponer exporters directamente a Internet cuando no sea necesario.

---

## 46. Regla de nombres

Cada servidor debe disponer de una etiqueta estable:

```text
server="<nombre>"
```

Ejemplos:

```text
server="lab01"
server="python"
server="security"
server="mail01"
server="mqh01"
```

El nombre debe mantenerse consistente entre:

```text
Prometheus
Loki
Grafana
CloudWatch
documentación
automatizaciones
```

---

## 47. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
node-exporter.md
seguridad.md
recuperacion.md
```

`servidores.md` mantiene la información operativa de cada máquina.

`estado-actual.md` representa la referencia rápida de toda la plataforma.

`seguridad.md` contiene las decisiones y procedimientos relacionados con
seguridad.

`recuperacion.md` contiene los procedimientos necesarios para reconstruir la
plataforma.
