# Logs — Plataforma de Monitorización

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Este documento describe qué logs se recopilan, de dónde proceden y cómo llegan
hasta Grafana.

La plataforma utiliza actualmente dos arquitecturas:

```text
SERVIDORES CON LOKI

Servidor
   ↓
Grafana Alloy
   ↓
Loki
   ↓
Grafana


MQH01

mqh01
  ↓
AWS CloudWatch Logs
  ↓
Grafana
```

Grafana proporciona una interfaz común aunque los registros procedan de
backends diferentes.

---

## 2. Componentes

La arquitectura de logs utiliza:

```text
Grafana Alloy
Loki
AWS CloudWatch Logs
Grafana
journald
rsyslog
Docker JSON logs
```

Cada componente tiene una función diferente.

---

## 3. Responsabilidades

### Alloy

```text
recoge
procesa
etiqueta
envía
```

### Loki

```text
almacena
indexa mediante etiquetas
permite consultas LogQL
```

### CloudWatch Logs

```text
almacena los logs AWS de mqh01
```

### Grafana

```text
consulta
filtra
visualiza
```

---

# ETIQUETAS

---

## 4. Etiquetas Loki

Los registros enviados a Loki utilizan principalmente:

```text
server
service
log_type
```

Ejemplo:

```text
server="mail01"
service="ssh"
log_type="ssh"
```

---

## 5. server

Identifica el servidor.

Ejemplos:

```text
server="lab01"
server="python"
server="security"
server="mail01"
```

---

## 6. service

Identifica el servicio que origina el evento.

Ejemplo:

```text
service="ssh"
```

No es la etiqueta principal utilizada por el selector del dashboard.

---

## 7. log_type

Clasifica el registro para facilitar su consulta.

Ejemplos:

```text
ssh
auth
cron
fail2ban
kernel
errors
system
rspamd
clamav
```

---

## 8. Consulta dinámica

Grafana utiliza:

```logql
{server="$instance",log_type="$log_type"}
```

Variables:

```text
$instance
$log_type
```

---

# SELECTOR DE LOGS

---

## 9. Variable log_type

El selector principal de Grafana contiene actualmente:

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

No todos los servidores tienen todos los tipos de log.

Si una combinación no existe:

```text
Servidor = X
Log = Y
```

el panel puede quedar vacío.

Esto es comportamiento normal.

---

## 10. Logs fuera del selector

Alloy también recopila actualmente en `mail01`:

```text
postfix
dovecot
```

Estos tipos pueden consultarse mediante LogQL aunque no estén actualmente
incluidos en el selector visible del dashboard.

Ejemplos:

```logql
{server="mail01",log_type="postfix"}
```

```logql
{server="mail01",log_type="dovecot"}
```

No se modifica el selector únicamente por el hecho de que estas categorías
existan.

---

# MAIL01

---

## 11. Arquitectura

`mail01` es un servidor de producción.

Flujo:

```text
mail01
   ↓
journald / archivos / Docker
   ↓
Grafana Alloy
   ↓
Loki
   ↓
Grafana
```

---

## 12. Logs del sistema

| log_type | Fuente | Estado |
|---|---|---|
| `ssh` | journald / `ssh.service` | OPERATIVO |
| `auth` | journald / `SYSLOG_FACILITY=10` | OPERATIVO |
| `cron` | journald / `cron.service` | OPERATIVO |
| `fail2ban` | `/var/log/fail2ban.log` | OPERATIVO |
| `kernel` | journald / kernel | OPERATIVO |
| `errors` | journald / prioridades 0-3 | OPERATIVO |
| `system` | journald / systemd | OPERATIVO |

---

# SSH

---

## 13. Fuente

Los registros SSH se obtienen desde journald mediante:

```text
_SYSTEMD_UNIT=ssh.service
```

Etiquetas:

```text
server="mail01"
service="ssh"
log_type="ssh"
```

Consulta:

```logql
{server="mail01",log_type="ssh"}
```

---

## 14. Utilidad

Permite analizar:

```text
conexiones
intentos fallidos
usuarios inexistentes
errores de autenticación
cierres de conexión
actividad de sshd
```

`mail01` recibe numerosos intentos automatizados desde Internet.

La existencia de intentos fallidos no implica un acceso exitoso.

---

## 15. Seguridad SSH

Configuración efectiva actual:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Los detalles están documentados en:

```text
seguridad.md
```

---

# AUTH

---

## 16. Fuente

Los eventos generales de autenticación se obtienen mediante:

```text
SYSLOG_FACILITY=10
```

Incluyen actividad relacionada con:

```text
PAM
sudo
autenticaciones
apertura de sesiones
cierre de sesiones
determinados eventos SSH
```

Consulta:

```logql
{server="mail01",log_type="auth"}
```

---

# CRON

---

## 17. Fuente

Los registros de Cron proceden de:

```text
cron.service
```

Consulta:

```logql
{server="mail01",log_type="cron"}
```

---

## 18. Backup

Entre las tareas observadas se encuentra:

```text
/usr/local/bin/ifs-backup backup mail
```

Los logs propios del sistema de backup se almacenan en:

```text
/opt/docker/backups/logs/cron/mail-cron.log
```

---

# FAIL2BAN

---

## 19. Fuente

Fail2ban está operativo en `mail01`.

Los eventos se encuentran en:

```text
/var/log/fail2ban.log
```

El archivo pertenece a:

```text
root:adm
```

---

## 20. Acceso de Alloy

El usuario:

```text
alloy
```

pertenece al grupo:

```text
adm
```

Esto permite leer:

```text
/var/log/fail2ban.log
```

La incorporación se realizó mediante:

```bash
sudo usermod -aG adm alloy
```

---

## 21. Eventos

Entre los eventos aparecen:

```text
Found
Ban
Unban
```

Consulta:

```logql
{server="mail01",log_type="fail2ban"}
```

---

# KERNEL

---

## 22. Fuente

Los registros del kernel se obtienen desde journald.

Consulta:

```logql
{server="mail01",log_type="kernel"}
```

Durante las pruebas se observaron eventos reales relacionados con:

```text
memoria
drop_caches
WireGuard
kernel Linux
```

---

# ERRORS

---

## 23. Fuente

La categoría `errors` recopila eventos del journal con prioridades:

```text
0 = emerg
1 = alert
2 = crit
3 = err
```

Consulta:

```logql
{server="mail01",log_type="errors"}
```

---

# SYSTEM

---

## 24. Fuente

Los eventos generales de systemd se clasifican mediante:

```text
SYSLOG_IDENTIFIER=systemd
```

Permiten visualizar:

```text
arranque de servicios
parada de servicios
reinicios
servicios finalizados
actividad systemd
actividad Alloy
servicios de monitorización
```

Consulta:

```logql
{server="mail01",log_type="system"}
```

---

# MAILCOW

---

## 25. Arquitectura

Parte de los logs de Mailcow se obtiene directamente de los archivos JSON
generados por Docker.

Ruta general:

```text
/var/lib/docker/containers/
```

Archivos:

```text
*-json.log
```

---

## 26. Servicios

Actualmente:

| log_type | Fuente | Estado |
|---|---|---|
| `postfix` | Docker JSON logs | OPERATIVO |
| `dovecot` | Docker JSON logs | OPERATIVO |
| `rspamd` | Docker JSON logs | OPERATIVO |
| `clamav` | Docker JSON logs | OPERATIVO |

---

# PERMISOS DOCKER

---

## 27. Problema detectado

Durante la configuración, Alloy no podía acceder a determinados logs Docker.

Error:

```text
Permission denied
```

El servidor no disponía inicialmente de:

```text
setfacl
```

---

## 28. ACL

Se instaló:

```bash
sudo apt update
sudo apt install -y acl
```

Posteriormente se concedió acceso específico a Alloy.

Ejemplos:

```bash
sudo setfacl -m u:alloy:rx /var/lib/docker
sudo setfacl -m u:alloy:rx /var/lib/docker/containers
```

Para directorios concretos:

```bash
sudo setfacl -Rm u:alloy:rX /var/lib/docker/containers/ID_CONTENEDOR
```

---

## 29. Regla de permisos

No utilizar:

```text
chmod 777
```

para resolver problemas de acceso a logs.

Debe utilizarse el mínimo permiso necesario mediante:

```text
grupos
ACL
```

---

# POSTFIX

---

## 30. Fuente

Postfix se obtiene desde los logs JSON de Docker.

Consulta:

```logql
{server="mail01",log_type="postfix"}
```

Estado:

```text
OPERATIVO
```

---

# DOVECOT

---

## 31. Fuente

Dovecot se obtiene desde los logs JSON de Docker.

Consulta:

```logql
{server="mail01",log_type="dovecot"}
```

Estado:

```text
OPERATIVO
```

---

# RSPAMD

---

## 32. Fuente

Rspamd se obtiene desde el archivo JSON del contenedor Docker correspondiente.

Consulta:

```logql
{server="mail01",log_type="rspamd"}
```

Estado:

```text
OPERATIVO
```

---

# CLAMAV

---

## 33. Fuente

ClamAV/Clamd se obtiene desde el archivo JSON del contenedor Docker
correspondiente.

Consulta:

```logql
{server="mail01",log_type="clamav"}
```

Estado:

```text
OPERATIVO
```

---

# DEPENDENCIA DE IDs DOCKER

---

## 34. Situación actual

Parte de la configuración de logs de Mailcow depende actualmente de:

```text
IDs concretos de contenedores Docker
```

Esta situación se identificó especialmente para:

```text
Rspamd
ClamAV
```

---

## 35. Riesgo

Los IDs de Docker son efímeros.

Si Mailcow recrea un contenedor:

```text
contenedor antiguo
        ↓
desaparece
        ↓
nuevo contenedor
        ↓
nuevo ID
        ↓
nueva ruta JSON
```

Alloy podría continuar intentando leer una ruta asociada al ID antiguo.

---

## 36. Consecuencia

Puede producirse esta situación:

```text
Mailcow                    OPERATIVO
Contenedor                 OPERATIVO
Alloy                      ACTIVE
Loki                       OPERATIVO
Grafana                    OPERATIVO

pero...

log concreto               NO LLEGA
```

Por tanto, comprobar únicamente que Alloy está activo no garantiza que todos
los logs Docker estén llegando.

---

## 37. Mejora pendiente

Implementar:

```text
descubrimiento dinámico de contenedores Docker
```

La futura solución debe identificar los servicios mediante elementos estables
como:

```text
nombre
labels Docker
servicio
```

en lugar de:

```text
container ID
```

---

## 38. Producción

Hasta que la nueva solución esté probada:

```text
NO sustituir la configuración funcional actual.
```

`mail01` es producción.

---

# DUPLICIDAD DE EVENTOS

---

## 39. Clasificación múltiple

Un mismo evento puede pertenecer a varias categorías.

Ejemplo:

```text
ssh
auth
errors
```

Un error SSH puede cumplir simultáneamente los criterios de estas categorías.

Esto es comportamiento esperado.

No significa necesariamente que Alloy esté enviando incorrectamente el mismo
evento varias veces.

---

# MQH01

---

## 40. Arquitectura

`mqh01` es producción y utiliza AWS.

Sus logs no se envían a Loki.

Flujo:

```text
mqh01
  ↓
CloudWatch Agent / aplicaciones
  ↓
AWS CloudWatch Logs
  ↓
Grafana
```

---

## 41. Logs disponibles

Actualmente:

```text
apache
ssh
fail2ban
errors
kernel
```

---

## 42. Log groups

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

---

# APACHE MQH01

---

## 43. WordPress

WordPress funciona mediante Docker.

Apache envía sus registros mediante:

```text
stdout
stderr
```

CloudWatch group:

```text
/masqhosting/mqh01/wordpress/apache
```

Stream:

```text
mqh01-wordpress
```

---

# SSH MQH01

---

## 44. Fuente

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

# ERRORS MQH01

---

## 45. Fuente

Archivo:

```text
/var/log/errors.log
```

Regla:

```text
*.err /var/log/errors.log
```

Grupo:

```text
/masqhosting/mqh01/errors
```

---

# FAIL2BAN MQH01

---

## 46. Fuente

Archivo:

```text
/var/log/fail2ban.log
```

Grupo:

```text
/masqhosting/mqh01/fail2ban
```

---

# KERNEL MQH01

---

## 47. Fuente

Archivo:

```text
/var/log/kern.log
```

Grupo:

```text
/masqhosting/mqh01/kernel
```

---

# RETENCIÓN MQH01

---

## 48. Periodos

```text
apache       30 días
kernel       30 días

errors       90 días
fail2ban     90 días
ssh          90 días
```

---

# CLOUDWATCH LOGS EN GRAFANA

---

## 49. Datasource

```text
cloudwatch-1
```

Región:

```text
eu-south-2
```

---

## 50. Consulta

Grafana utiliza:

```text
fields @timestamp, @message
| filter @log like /$instance/
| filter @log like /$log_type/
| sort @timestamp desc
| limit 100
```

---

## 51. Funcionamiento

Para:

```text
$instance = mqh01
```

el filtro:

```text
@log like /mqh01/
```

coincide con los grupos AWS correspondientes.

Para:

```text
lab01
python
security
mail01
```

CloudWatch no devuelve registros de `mqh01`.

---

# PANEL ÚNICO DE GRAFANA

---

## 52. Diseño

El dashboard mantiene:

```text
UN panel de logs
```

en lugar de crear paneles independientes para:

```text
SSH
Apache
Fail2ban
Kernel
Errors
...
```

---

## 53. Selección

El usuario selecciona:

```text
Servidor
Log
```

Grafana consulta los backends configurados.

Conceptualmente:

```text
              Servidor + Log
                    │
          ┌─────────┴─────────┐
          │                   │
        Loki             CloudWatch
          │                   │
          └─────────┬─────────┘
                    │
                    ▼
                 Grafana
```

---

# CONSULTAS LOGQL ÚTILES

---

## 54. Todos los logs de mail01

```logql
{server="mail01"}
```

---

## 55. SSH

```logql
{server="mail01",log_type="ssh"}
```

---

## 56. Auth

```logql
{server="mail01",log_type="auth"}
```

---

## 57. Cron

```logql
{server="mail01",log_type="cron"}
```

---

## 58. Fail2ban

```logql
{server="mail01",log_type="fail2ban"}
```

---

## 59. Kernel

```logql
{server="mail01",log_type="kernel"}
```

---

## 60. Errors

```logql
{server="mail01",log_type="errors"}
```

---

## 61. System

```logql
{server="mail01",log_type="system"}
```

---

## 62. Postfix

```logql
{server="mail01",log_type="postfix"}
```

---

## 63. Dovecot

```logql
{server="mail01",log_type="dovecot"}
```

---

## 64. Rspamd

```logql
{server="mail01",log_type="rspamd"}
```

---

## 65. ClamAV

```logql
{server="mail01",log_type="clamav"}
```

---

## 66. Varios tipos

```logql
{server="mail01",log_type=~"kernel|errors|system"}
```

---

# DIAGNÓSTICO

---

## 67. Log Loki ausente

Seguir:

```text
¿el servicio genera el log?
          ↓
¿Alloy puede leerlo?
          ↓
¿la fuente configurada es correcta?
          ↓
¿las etiquetas son correctas?
          ↓
¿Alloy puede alcanzar Loki?
          ↓
¿Loki lo recibe?
          ↓
¿Grafana lo consulta correctamente?
```

---

## 68. Log Docker ausente

Además revisar:

```text
¿se recreó el contenedor?
¿cambió el ID?
¿la ruta JSON sigue existiendo?
¿la ACL sigue permitiendo acceso?
```

---

## 69. Log mqh01 ausente

Seguir:

```text
archivo / aplicación
        ↓
CloudWatch Agent
        ↓
CloudWatch Logs
        ↓
Log Group
        ↓
Grafana CloudWatch datasource
        ↓
consulta
```

No revisar Loki para un problema exclusivo de logs de `mqh01`.

---

# SEGURIDAD

---

## 70. Información sensible

Los logs pueden contener:

```text
direcciones IP
usuarios
direcciones de correo
hostnames
rutas internas
errores
información de servicios
```

No deben compartirse registros completos sin revisar previamente su contenido.

---

## 71. Credenciales

Nunca deben enviarse a los logs deliberadamente:

```text
contraseñas
tokens
AWS Secret Access Keys
claves privadas
credenciales
```

---

## 72. Permisos

Los agentes deben tener:

```text
mínimo acceso necesario
```

No modificar globalmente los permisos de Docker para facilitar la lectura de
logs.

---

# ESTADO ACTUAL

---

## 73. mail01 — sistema

```text
SSH                         OPERATIVO
Auth                        OPERATIVO
Cron                        OPERATIVO
Fail2ban                    OPERATIVO
Kernel                      OPERATIVO
Errors                      OPERATIVO
System                      OPERATIVO
```

---

## 74. mail01 — Mailcow

```text
Postfix                     OPERATIVO
Dovecot                     OPERATIVO
Rspamd                      OPERATIVO
ClamAV                      OPERATIVO
```

---

## 75. mqh01

```text
CloudWatch Logs             OPERATIVO
Apache                      OPERATIVO
SSH                         OPERATIVO
Fail2ban                    OPERATIVO
Errors                      OPERATIVO
Kernel                      OPERATIVO
```

---

## 76. Infraestructura

```text
Alloy                       OPERATIVO
Loki                        OPERATIVO
CloudWatch Logs             OPERATIVO
Grafana                     OPERATIVO
```

---

## 77. Pendiente técnico

```text
Mailcow:
sustituir dependencia de IDs Docker estáticos
por descubrimiento dinámico de contenedores.
```

No modificar la configuración funcional de producción hasta disponer de una
alternativa validada.

---

## 78. Regla para nuevos logs

Antes de incorporar un nuevo registro determinar:

```text
1. servidor
2. servicio
3. fuente real
4. backend: Loki o CloudWatch
5. log_type
6. permisos necesarios
7. retención
8. utilidad
9. posible información sensible
```

No se recopila un log únicamente porque exista.

---

## 79. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
alloy.md
logs.md
node-exporter.md
servidores.md
seguridad.md
recuperacion.md
```

Responsabilidades:

```text
alloy.md
    ↓
agente y funcionamiento de Alloy

logs.md
    ↓
fuentes, categorías y recorrido de los logs

loki.md
    ↓
backend Loki

grafana.md
    ↓
visualización y consultas

seguridad.md
    ↓
seguridad y permisos

recuperacion.md
    ↓
recuperación ante fallos
```
