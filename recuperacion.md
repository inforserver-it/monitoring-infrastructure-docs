# Recuperación — Plataforma de Monitorización

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Este documento describe cómo recuperar la plataforma de monitorización en caso
de fallo parcial o total.

Debe permitir reconstruir los componentes principales sin depender de recordar
manualmente cómo estaba montado el sistema.

La plataforma central está formada por:

```text
Fedora44
├── Grafana
├── Prometheus
├── Loki
└── YACE
```

Los servidores monitorizados son:

```text
lab01
python
security
mail01
mqh01
```

---

## 2. Principio de recuperación

Antes de modificar nada debe identificarse qué componente ha fallado.

No debe reconstruirse toda la plataforma si únicamente ha fallado un servicio.

Orden de diagnóstico:

```text
1. Fedora44
2. Docker
3. red Docker monitoring
4. Prometheus
5. Loki
6. YACE
7. Grafana
8. exporters/agentes
9. conectividad
10. AWS CloudWatch
```

---

## 3. Arquitectura que debe recuperarse

### Métricas Linux

```text
lab01 ──────┐
python ─────┤
security ───┼── Node Exporter ── Prometheus ── Grafana
mail01 ─────┘
```

### Métricas AWS

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

### Logs Linux

```text
Servidor
   ↓
Grafana Alloy
   ↓
Loki
   ↓
Grafana
```

### Logs mqh01

```text
mqh01
  ↓
AWS CloudWatch Logs
  ↓
Grafana
```

---

# UBICACIONES IMPORTANTES

---

## 4. Documentación

```text
/datos/docs/monitoring/
```

Documentos principales:

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
alloy.md
alloy-logs.md
node-exporter.md
servidores.md
seguridad.md
recuperacion.md
```

---

## 5. Prometheus

```text
/datos/stacks/prometheus/
```

Archivos principales:

```text
/datos/stacks/prometheus/prometheus.yml
/datos/stacks/prometheus/compose.yaml
```

---

## 6. Loki

```text
/datos/stacks/loki/
```

La configuración y Docker Compose deben conservarse dentro del stack
correspondiente.

---

## 7. Grafana

```text
/datos/stacks/grafana/
```

El dashboard principal es:

```text
Monitorización Laboratorios
```

---

## 8. YACE

```text
/datos/stacks/yace/
```

Configuración:

```text
/datos/stacks/yace/config.yml
```

Las credenciales AWS no deben almacenarse en este documento.

---

## 9. Ansible

```text
/datos/ansible/masqhosting-ops/
```

Ansible puede utilizarse para recuperar o validar configuraciones de los
servidores cuando corresponda.

---

# RECUPERACIÓN DE FEDORA44

---

## 10. Si Fedora44 sigue funcionando

Si Fedora44 arranca pero la monitorización no funciona, no debe reinstalarse
nada inicialmente.

Primero debe determinarse qué componente ha fallado.

Comprobar los contenedores:

```bash
sudo docker ps
```

La plataforma debería contener los servicios principales:

```text
grafana
prometheus
loki
yace
```

---

## 11. Si Docker no está funcionando

Comprobar:

```bash
sudo systemctl status docker
```

Si Docker está detenido:

```bash
sudo systemctl start docker
```

No debe reinstalarse Docker únicamente porque un contenedor concreto esté
detenido.

---

## 12. Red Docker monitoring

Los componentes centrales utilizan:

```text
monitoring
```

Si fuera necesario reconstruir la red:

```bash
sudo docker network create monitoring
```

Solo debe ejecutarse si la red realmente no existe.

Los stacks deben conectarse posteriormente a esta red.

---

# RECUPERACIÓN DE PROMETHEUS

---

## 13. Función

Prometheus recibe métricas de:

```text
Node Exporter
YACE
```

Jobs:

```text
lab-servers
aws-mqh01
```

---

## 14. Configuración esperada

Servidores Node Exporter:

```text
lab01      192.168.1.101:9100
security   192.168.1.103:9100
python     192.168.1.104:9100
mail01     5.189.147.4:9100
```

Job:

```text
lab-servers
```

YACE:

```text
yace:5000
```

Job:

```text
aws-mqh01
```

Etiqueta:

```text
server="mqh01"
```

---

## 15. Levantar Prometheus

```bash
cd /datos/stacks/prometheus
sudo docker compose up -d
```

Si únicamente necesita reiniciarse:

```bash
sudo docker compose restart prometheus
```

---

## 16. Estado esperado

Los servidores Node Exporter deben aparecer como:

```text
UP
```

Y YACE debe proporcionar:

```promql
up{job="aws-mqh01"}
```

con resultado:

```text
1
```

---

# RECUPERACIÓN DE YACE

---

## 17. Función

YACE transforma:

```text
AWS CloudWatch
```

en:

```text
métricas Prometheus
```

Flujo:

```text
CloudWatch → YACE → Prometheus
```

---

## 18. Configuración

Archivo:

```text
/datos/stacks/yace/config.yml
```

YACE debe descubrir:

### AWS/EC2

```text
CPUUtilization
StatusCheckFailed
NetworkIn
NetworkOut
```

### mqh01-memory

```text
mem_used_percent
mem_total
```

### mqh01-disk

```text
disk_used_percent
disk_total
```

### mqh01-system

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

---

## 19. Dimensiones críticas

Para:

```text
mqh01-system
```

deben existir:

```text
InstanceId
metric_type
```

Este detalle es crítico.

Si las métricas existen en CloudWatch pero YACE no las descubre, comprobar
primero:

```text
dimensionNameRequirements
```

y confirmar que incluye:

```text
metric_type
```

---

## 20. Disco

Para las métricas de disco deben contemplarse:

```text
InstanceId
device
fstype
path
```

El filesystem utilizado por Grafana es:

```text
/
```

---

## 21. Levantar YACE

```bash
cd /datos/stacks/yace
sudo docker compose up -d
```

Si únicamente necesita reiniciarse:

```bash
sudo docker compose restart yace
```

---

## 22. Comunicación Prometheus → YACE

Prometheus debe utilizar:

```text
yace:5000
```

No debe configurarse permanentemente una IP interna Docker.

Las IP de los contenedores pueden cambiar.

---

# RECUPERACIÓN DE LOKI

---

## 23. Función

Loki almacena los registros enviados por los agentes integrados.

Flujo:

```text
Servidor → Alloy → Loki → Grafana
```

---

## 24. Levantar Loki

```bash
cd /datos/stacks/loki
sudo docker compose up -d
```

Si únicamente necesita reiniciarse:

```bash
sudo docker compose restart loki
```

---

## 25. Endpoint interno

Grafana y los componentes internos deben utilizar:

```text
http://loki:3100
```

No utilizar permanentemente la IP interna del contenedor.

---

## 26. Consulta base

La consulta utilizada por Grafana es:

```logql
{server="$instance", log_type="$log_type"}
```

Si Loki funciona pero Grafana no muestra logs, revisar:

```text
server
log_type
```

antes de modificar la infraestructura.

---

# RECUPERACIÓN DE GRAFANA

---

## 27. Función

Grafana es la interfaz central.

Datasource principales:

```text
Prometheus
Loki
cloudwatch-1
```

---

## 28. Levantar Grafana

```bash
cd /datos/stacks/grafana
sudo docker compose up -d
```

Si únicamente necesita reiniciarse:

```bash
sudo docker compose restart grafana
```

---

## 29. Dashboard principal

Nombre:

```text
Monitorización Laboratorios
```

Variables visibles:

```text
Servidor
Log
```

Variables internas:

```text
instance
log_type
```

---

## 30. Orden de servidores

```text
lab01
python
security
mail01
mqh01
```

---

## 31. Tipos de log

```text
ssh
apache
fail2ban
errors
kernel
auth
system
cron
clamav
hestia
rspamd
```

---

## 32. Recuperación de consultas

Las consultas PromQL definitivas están documentadas en:

```text
grafana.md
```

No deben reconstruirse de memoria.

Las consultas combinan:

```text
Node Exporter
OR
YACE
```

---

# RECUPERACIÓN DE NODE EXPORTER

---

## 33. Servidores

Node Exporter debe estar operativo en:

```text
lab01
python
security
mail01
```

`mqh01` no utiliza Node Exporter.

---

## 34. Puerto

```text
9100
```

Prometheus debe poder alcanzar:

```text
192.168.1.101:9100
192.168.1.103:9100
192.168.1.104:9100
5.189.147.4:9100
```

---

## 35. Si un servidor aparece DOWN

No debe asumirse inmediatamente que Node Exporter está roto.

Revisar conceptualmente:

```text
servidor encendido
       ↓
red disponible
       ↓
Node Exporter activo
       ↓
puerto accesible
       ↓
Prometheus puede hacer scrape
```

---

# RECUPERACIÓN DE LOGS

---

## 36. Loki

Si faltan logs de:

```text
lab01
python
security
mail01
```

revisar:

```text
agente Alloy
etiqueta server
etiqueta log_type
conectividad con Loki
```

---

## 37. mqh01

Los logs de `mqh01` no dependen de Loki.

Flujo:

```text
mqh01 → CloudWatch Logs → Grafana
```

Datasource:

```text
cloudwatch-1
```

Región:

```text
eu-south-2
```

---

## 38. Log groups de mqh01

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

---

## 39. Consulta CloudWatch Logs

```text
fields @timestamp, @message
| filter @log like /$instance/
| filter @log like /$log_type/
| sort @timestamp desc
| limit 100
```

---

# RECUPERACIÓN DE CLOUDWATCH AGENT

---

## 40. Servidor

CloudWatch Agent se ejecuta en:

```text
mqh01
```

Sistema:

```text
Debian 13
```

---

## 41. Funciones

Recopila:

```text
RAM
disco
métricas StatsD
logs
```

---

## 42. StatsD

Listener:

```text
127.0.0.1:8125/UDP
```

Configuración activa conocida:

```text
[[inputs.statsd]]
interval = "60s"
parse_data_dog_tags = true
service_address = "127.0.0.1:8125"

[inputs.statsd.tags]
  "aws:AggregationInterval" = "60s"
```

---

## 43. Script de métricas

```text
/usr/local/bin/mqh01-system-metrics
```

Genera:

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

---

## 44. Cron

```text
/etc/cron.d/mqh01-system-metrics
```

Configuración:

```cron
* * * * * root /usr/local/bin/mqh01-system-metrics
```

---

## 45. Precaución con fetch-config

No ejecutar repetidamente `fetch-config` utilizando como origen archivos
generados dentro de:

```text
/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/
```

Esto provocó anteriormente nombres recursivos:

```text
file_file_...
```

La configuración activa debe tratarse con cuidado.

---

# RECUPERACIÓN DE MAIL01

---

## 46. SSH

Configuración efectiva esperada:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

---

## 47. cloud-init

Si aparece nuevamente:

```text
passwordauthentication yes
```

aunque `/etc/ssh/sshd_config` diga:

```text
PasswordAuthentication no
```

revisar:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

Este archivo fue el origen del override detectado anteriormente.

---

## 48. Fail2ban

Jail esperado:

```text
sshd
```

Fail2ban debe permanecer activo.

No debe confundirse un gran número de intentos SSH con accesos exitosos.

---

# PÉRDIDA TOTAL DE FEDORA44

---

## 49. Escenario

Si Fedora44 deja de existir completamente, debe reconstruirse primero el nodo
central antes de modificar los servidores monitorizados.

Los servidores:

```text
lab01
python
security
mail01
mqh01
```

pueden continuar funcionando aunque Fedora44 esté caído.

La caída de Fedora44 afecta principalmente a:

```text
visualización
recopilación central
consultas
histórico local
```

No debe asumirse que los servidores de producción están caídos porque Grafana
no esté disponible.

---

## 50. Orden de reconstrucción

Orden recomendado:

```text
1. Sistema Fedora
2. Docker
3. estructura /datos
4. red Docker monitoring
5. Prometheus
6. Loki
7. YACE
8. Grafana
9. datasources
10. dashboard
11. Alloy / conectividad
12. validación de servidores
```

---

## 51. Estructura mínima

Debe recuperarse:

```text
/datos/docs/
/datos/stacks/
/datos/ansible/
```

y cualquier otro directorio necesario para volúmenes persistentes de los
servicios.

---

## 52. Docker

Instalar Docker y Docker Compose según la versión soportada por Fedora.

Después crear o recuperar:

```text
monitoring
```

y levantar cada stack individualmente.

---

## 53. Orden de servicios

Orden práctico:

```text
Prometheus
Loki
YACE
Grafana
```

Grafana se levanta después de los backends para que sus datasources puedan
conectarse correctamente.

---

## 54. AWS

La recuperación de Fedora44 no requiere reinstalar CloudWatch Agent en
`mqh01` si el agente continúa funcionando.

CloudWatch seguirá recibiendo:

```text
métricas
logs
```

aunque Fedora44 esté apagado.

Cuando YACE vuelva a funcionar podrá volver a consultar las métricas
existentes en AWS.

---

# DATOS QUE DEBEN RESPALDARSE

---

## 55. Configuraciones

Como mínimo:

```text
Prometheus
Loki
Grafana
YACE
Alloy
Node Exporter
CloudWatch Agent
scripts personalizados
cron
documentación
```

---

## 56. Grafana

Debe conservarse:

```text
configuración
datasources
dashboard
volumen persistente o backup equivalente
```

Especialmente debe exportarse periódicamente el JSON del dashboard:

```text
Monitorización Laboratorios
```

porque contiene:

```text
paneles
variables
consultas
transformaciones
mappings
diseño
```

---

## 57. Prometheus

Debe conservarse:

```text
prometheus.yml
compose.yaml
```

El histórico de métricas puede respaldarse si se considera necesario, pero la
configuración es prioritaria para poder reconstruir el servicio.

---

## 58. YACE

Debe conservarse:

```text
config.yml
compose.yaml
```

Los secretos deben respaldarse por separado mediante un mecanismo seguro.

Nunca deben introducirse en este Markdown.

---

## 59. Loki

Debe conservarse:

```text
configuración
compose
volumen de datos si se desea preservar histórico
```

La pérdida del histórico no impide reconstruir la recopilación futura.

---

## 60. Documentación

Debe conservarse íntegramente:

```text
/datos/docs/monitoring/
```

Esta documentación contiene las decisiones necesarias para reconstruir la
arquitectura sin repetir todo el proceso de investigación.

---

# SECRETOS

---

## 61. Credenciales

No deben almacenarse en este documento:

```text
contraseñas
AWS Access Keys
AWS Secret Access Keys
tokens
claves SSH privadas
credenciales Grafana
```

---

## 62. YACE

Las credenciales AWS utilizadas por YACE deben recuperarse desde su ubicación
segura.

No deben copiarse a:

```text
Markdown
Git
README
capturas
chats
```

---

## 63. Credencial comprometida

Si existe cualquier sospecha de que una credencial haya sido expuesta:

```text
1. rotarla
2. actualizar el servicio
3. invalidar la anterior
4. revisar actividad
```

No debe mantenerse una credencial expuesta únicamente porque siga
funcionando.

---

# DIAGNÓSTICO RÁPIDO

---

## 64. Grafana funciona pero no hay métricas

Revisar:

```text
Grafana
   ↓
datasource Prometheus
   ↓
Prometheus
   ↓
target
```

Si afecta únicamente a `mqh01`:

```text
Grafana
   ↓
Prometheus
   ↓
YACE
   ↓
CloudWatch
```

---

## 65. Grafana funciona pero no hay logs

Para servidores Loki:

```text
Grafana
   ↓
Loki
   ↓
Alloy
   ↓
archivo de log
```

Para `mqh01`:

```text
Grafana
   ↓
CloudWatch datasource
   ↓
CloudWatch Logs
```

---

## 66. Solo falla mqh01

Revisar por separado:

### Métricas

```text
CloudWatch
YACE
Prometheus
Grafana
```

### Logs

```text
CloudWatch Logs
Grafana
```

No reiniciar WordPress por un fallo de métricas.

---

## 67. Solo falla un servidor Node Exporter

Revisar:

```text
servidor
Node Exporter
red
Prometheus target
```

No reiniciar Prometheus completo automáticamente si los demás servidores
funcionan.

---

## 68. Solo falla un tipo de log

Revisar:

```text
archivo origen
agente
etiqueta log_type
consulta Grafana
```

Si los demás logs funcionan, no reconstruir Loki.

---

# PRINCIPIOS DE EMERGENCIA

---

## 69. Regla 1

```text
No tocar lo que funciona.
```

---

## 70. Regla 2

```text
Identificar primero el componente que falla.
```

---

## 71. Regla 3

```text
Un fallo de monitorización no significa necesariamente un fallo del servidor.
```

---

## 72. Regla 4

```text
mail01 y mqh01 son producción.
```

No reiniciarlos por problemas que estén únicamente en Fedora44.

---

## 73. Regla 5

```text
No exponer servicios para solucionar rápidamente un problema.
```

No abrir:

```text
9100
3100
9090
```

a Internet indiscriminadamente.

---

## 74. Regla 6

```text
No imprimir secretos durante el diagnóstico.
```

Especialmente:

```text
.env
variables Docker
credenciales AWS
```

---

## 75. Regla 7

Cuando se solucione un problema nuevo que haya requerido investigación:

```text
documentarlo
```

El objetivo es que el mismo problema no tenga que investigarse dos veces.

---

# ESTADO QUE DEBE CONSEGUIRSE

---

## 76. Fedora44

```text
Grafana                    OPERATIVO
Prometheus                 OPERATIVO
Loki                       OPERATIVO
YACE                       OPERATIVO
Red monitoring             OPERATIVA
```

---

## 77. Node Exporter

```text
lab01                      UP
python                     UP
security                   UP
mail01                     UP
```

---

## 78. AWS

```text
mqh01
  CloudWatch               OPERATIVO
  CloudWatch Agent         OPERATIVO
  YACE                     OPERATIVO
  Prometheus               UP
  Grafana                  OPERATIVO
  CloudWatch Logs          OPERATIVO
```

---

## 79. Dashboard

```text
Servidor                   OPERATIVO
Log                        OPERATIVO

CPU                        OPERATIVA
RAM                        OPERATIVA
Disco                      OPERATIVO
Load 1m                    OPERATIVO
Load 5m                    OPERATIVO
Load 15m                   OPERATIVO
Uptime                     OPERATIVO
Estado                     OPERATIVO
Red entrada                OPERATIVA
Red salida                 OPERATIVA
Logs                       OPERATIVOS
```

---

## 80. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
alloy.md
alloy-logs.md
node-exporter.md
servidores.md
seguridad.md
```

Si se produce una pérdida total:

```text
estado-actual.md
```

indica qué debe existir.

```text
arquitectura.md
```

explica cómo se conecta.

```text
grafana.md
```

contiene las consultas del dashboard.

```text
prometheus.md
```

contiene los jobs y métricas.

```text
loki.md
```

contiene la arquitectura de logs.

```text
servidores.md
```

contiene la información de cada servidor.

```text
seguridad.md
```

contiene las decisiones de seguridad.

Y este documento:

```text
recuperacion.md
```

indica el orden para volver a levantarlo.
