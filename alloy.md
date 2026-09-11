# Grafana Alloy — Recolección de Logs

**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Grafana Alloy actúa como agente de recolección de logs en los servidores
integrados con Loki.

Arquitectura:

```text
Servidor
   ↓
Grafana Alloy
   ↓
Loki
   ↓
Grafana
```

Alloy se encarga de:

- leer registros;
- procesarlos;
- asignar etiquetas;
- identificar servidor y servicio;
- enviarlos a Loki.

---

## 2. Alcance actual

La integración documentada en este archivo incluye especialmente `mail01`,
servidor de producción que utiliza Alloy para centralizar logs del sistema y
de Mailcow.

`mqh01` utiliza una arquitectura diferente:

```text
mqh01
  ↓
AWS CloudWatch Logs
  ↓
Grafana
```

Por tanto:

```text
mqh01 NO depende de Alloy para sus logs
```

---

## 3. Configuración de Alloy

Archivo principal:

```text
/etc/alloy/config.alloy
```

Servicio:

```text
alloy.service
```

Comandos operativos:

```bash
sudo systemctl status alloy
sudo systemctl restart alloy
sudo systemctl is-active alloy
sudo journalctl -u alloy -n 50 --no-pager
```

Resultado normal:

```text
active
```

---

## 4. Endpoint Loki

El endpoint utilizado actualmente por Alloy es:

```text
http://10.50.0.1:3100
```

Este endpoint corresponde al acceso utilizado por los agentes hacia Loki en
Fedora44.

No debe confundirse con el endpoint interno utilizado entre contenedores de
la red Docker:

```text
http://loki:3100
```

Ambos pueden existir porque corresponden a contextos de red diferentes.

---

## 5. Etiquetas

Los registros utilizan principalmente:

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

Grafana utiliza principalmente:

```text
server
log_type
```

Consulta:

```logql
{server="mail01",log_type="ssh"}
```

En el dashboard dinámico:

```logql
{server="$instance",log_type="$log_type"}
```

---

# MAIL01

---

## 6. Servidor

`mail01` es:

```text
PRODUCCIÓN
```

No debe tratarse como servidor de laboratorio.

Flujo de logs:

```text
mail01
   ↓
Alloy
   ↓
Loki
   ↓
Grafana
```

---

## 7. Logs del sistema

Actualmente se recopilan:

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

## 8. Logs de Mailcow

También se recopilan logs de servicios Mailcow ejecutados mediante Docker.

| log_type | Fuente | Estado |
|---|---|---|
| `postfix` | Docker JSON logs | OPERATIVO |
| `dovecot` | Docker JSON logs | OPERATIVO |
| `rspamd` | Docker JSON logs | OPERATIVO |
| `clamav` | Docker JSON logs | OPERATIVO |

---

# SSH

---

## 9. Origen

Los registros SSH se obtienen desde journald.

Unidad utilizada:

```text
ssh.service
```

Filtro:

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

## 10. Información disponible

Los registros SSH permiten analizar:

- conexiones;
- intentos fallidos;
- usuarios inexistentes;
- errores de autenticación;
- cierres de conexión;
- actividad de `sshd`;
- sesiones.

`mail01` recibe numerosos intentos automatizados procedentes de Internet.

Esto no implica por sí mismo que se haya producido una autenticación exitosa.

---

## 11. Seguridad SSH actual

La configuración efectiva de `mail01` es:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Por tanto:

```text
Root por SSH              DESACTIVADO
Contraseña SSH            DESACTIVADA
Clave pública             ACTIVADA
Keyboard Interactive      DESACTIVADO
```

Los detalles del endurecimiento están documentados en:

```text
seguridad.md
```

---

# AUTH

---

## 12. Origen

Los eventos generales de autenticación se obtienen mediante:

```text
SYSLOG_FACILITY=10
```

Pueden incluir:

- PAM;
- sudo;
- autenticaciones;
- apertura de sesiones;
- cierre de sesiones;
- determinados eventos SSH.

Consulta:

```logql
{server="mail01",log_type="auth"}
```

---

# CRON

---

## 13. Origen

Los registros de Cron proceden de:

```text
cron.service
```

Consulta:

```logql
{server="mail01",log_type="cron"}
```

---

## 14. Backup

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

## 15. Estado

Fail2ban está operativo en `mail01`.

Servicio:

```text
fail2ban.service
```

Jail utilizado:

```text
sshd
```

---

## 16. Origen real de los eventos

Los eventos operativos de Fail2ban no se obtienen principalmente desde:

```bash
journalctl -u fail2ban.service
```

Se encuentran en:

```text
/var/log/fail2ban.log
```

El archivo pertenece a:

```text
root:adm
```

---

## 17. Permisos de Alloy

Para permitir la lectura del archivo, el usuario `alloy` pertenece al grupo:

```text
adm
```

La incorporación se realizó mediante:

```bash
sudo usermod -aG adm alloy
```

Esto permite a Alloy leer:

```text
/var/log/fail2ban.log
```

---

## 18. Eventos Fail2ban

Entre los eventos disponibles se encuentran:

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

## 19. Origen

Los registros del kernel se obtienen desde journald.

Consulta:

```logql
{server="mail01",log_type="kernel"}
```

Durante la configuración se comprobaron eventos reales relacionados con el
sistema, memoria y actividad de WireGuard.

---

# ERRORS

---

## 20. Prioridades

La categoría `errors` recopila eventos con prioridades:

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

Un evento clasificado como error puede aparecer simultáneamente en otra
categoría.

Esto es comportamiento esperado.

---

# SYSTEM

---

## 21. Origen

Los registros generales de systemd utilizan:

```text
SYSLOG_IDENTIFIER=systemd
```

Permiten observar:

- arranque de servicios;
- parada de servicios;
- reinicios;
- servicios finalizados;
- actividad de systemd;
- actividad relacionada con Alloy;
- servicios de monitorización.

Consulta:

```logql
{server="mail01",log_type="system"}
```

---

# MAILCOW

---

## 22. Logs Docker

Los servicios Mailcow generan parte de sus registros mediante Docker.

Los archivos se encuentran bajo:

```text
/var/lib/docker/containers/
```

Formato:

```text
*-json.log
```

Actualmente se recopilan:

```text
postfix
dovecot
rspamd
clamav
```

---

## 23. Permisos Docker

Durante la integración se detectó:

```text
Permission denied
```

cuando Alloy intentaba leer determinados archivos JSON de Docker.

Para resolverlo se instaló soporte ACL:

```bash
sudo apt update
sudo apt install -y acl
```

Posteriormente se concedieron permisos específicos mediante:

```text
setfacl
```

al usuario:

```text
alloy
```

sobre las rutas necesarias.

---

## 24. Principio de permisos

No debe modificarse de forma indiscriminada la propiedad o permisos generales
de:

```text
/var/lib/docker
```

La solución utiliza permisos específicos mediante ACL para conceder únicamente
el acceso necesario.

---

# POSTFIX

---

## 25. Origen

Postfix se obtiene desde los logs JSON del contenedor Docker correspondiente.

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

## 26. Origen

Dovecot se obtiene desde los logs JSON del contenedor Docker correspondiente.

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

## 27. Origen

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

## 28. Origen

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

# CONSULTAS LOGQL

---

## 29. Todos los registros de mail01

```logql
{server="mail01"}
```

---

## 30. SSH

```logql
{server="mail01",log_type="ssh"}
```

---

## 31. Auth

```logql
{server="mail01",log_type="auth"}
```

---

## 32. Cron

```logql
{server="mail01",log_type="cron"}
```

---

## 33. Fail2ban

```logql
{server="mail01",log_type="fail2ban"}
```

---

## 34. Kernel

```logql
{server="mail01",log_type="kernel"}
```

---

## 35. Errors

```logql
{server="mail01",log_type="errors"}
```

---

## 36. System

```logql
{server="mail01",log_type="system"}
```

---

## 37. Postfix

```logql
{server="mail01",log_type="postfix"}
```

---

## 38. Dovecot

```logql
{server="mail01",log_type="dovecot"}
```

---

## 39. Rspamd

```logql
{server="mail01",log_type="rspamd"}
```

---

## 40. ClamAV

```logql
{server="mail01",log_type="clamav"}
```

---

## 41. Varios tipos

Ejemplo:

```logql
{server="mail01",log_type=~"kernel|errors|system"}
```

---

## 42. Dashboard dinámico

La consulta principal del dashboard es:

```logql
{server="$instance",log_type="$log_type"}
```

Variables:

```text
$instance
$log_type
```

---

# DUPLICIDAD

---

## 43. Eventos en varias categorías

Un mismo evento puede pertenecer a varias categorías.

Ejemplo:

```text
ssh
auth
errors
```

Un fallo relacionado con SSH puede cumplir simultáneamente los criterios de
las tres categorías.

Esto es consecuencia de la clasificación lógica utilizada y no implica
necesariamente duplicación incorrecta por parte de Alloy.

---

# CONTENEDORES DOCKER

---

## 44. Dependencia actual de IDs

Existe una limitación conocida en parte de la integración de Mailcow.

Algunas rutas de logs Docker dependen actualmente de:

```text
IDs concretos de contenedores
```

Especialmente se identificó esta situación para:

```text
rspamd
clamav
```

Los IDs de Docker no son permanentes.

Si Mailcow recrea un contenedor:

```text
ID antiguo
    ↓
desaparece

ID nuevo
    ↓
nuevo archivo JSON
```

Alloy podría continuar intentando leer la ruta antigua.

---

## 45. Riesgo

Una recreación de los contenedores puede provocar que:

```text
servicio funcionando
        +
contenedor nuevo
        +
Alloy activo
        =
log deja de aparecer en Loki
```

Por tanto, que Alloy esté:

```text
active
```

no garantiza por sí mismo que todos los logs Docker estén llegando.

---

## 46. Mejora pendiente

Implementar:

```text
descubrimiento dinámico de contenedores Docker
```

para evitar depender de IDs estáticos.

La solución deberá identificar los contenedores por atributos estables como:

```text
nombre
labels
servicio
```

en lugar de IDs efímeros.

---

## 47. Regla de producción

Hasta que el descubrimiento dinámico esté completamente probado:

```text
NO retirar la configuración actualmente funcional.
```

La nueva solución debe validarse antes de sustituir la existente.

`mail01` es producción.

---

# OPERACIÓN

---

## 48. Cambio de configuración

Antes de modificar Alloy en producción debe conocerse exactamente:

```text
qué log se quiere recoger
dónde se genera realmente
qué permisos necesita
qué etiqueta tendrá
qué consulta utilizará Grafana
```

No debe añadirse una fuente basándose únicamente en dónde se supone que un
servicio debería escribir.

Primero debe identificarse el origen real del registro.

---

## 49. Aplicación de cambios

Después de modificar:

```text
/etc/alloy/config.alloy
```

puede reiniciarse únicamente Alloy:

```bash
sudo systemctl restart alloy
```

No es necesario reiniciar:

```text
Mailcow
Docker
Postfix
Dovecot
Rspamd
ClamAV
Fail2ban
SSH
```

por un cambio exclusivo en Alloy.

---

## 50. Diagnóstico de Alloy

Estado:

```bash
sudo systemctl status alloy
```

Estado simplificado:

```bash
sudo systemctl is-active alloy
```

Journal:

```bash
sudo journalctl -u alloy -n 50 --no-pager
```

Resultado esperado:

```text
active
```

---

## 51. Diagnóstico de un log ausente

Si un log deja de aparecer:

```text
1. comprobar que el servicio genera eventos
2. identificar la fuente real
3. comprobar permisos de Alloy
4. comprobar la configuración de Alloy
5. comprobar etiquetas
6. comprobar Loki
7. comprobar la consulta Grafana
```

Para logs Docker revisar además:

```text
¿se recreó el contenedor?
¿cambió su ID?
¿la ruta configurada sigue existiendo?
```

---

# RELACIÓN CON GRAFANA

---

## 52. Variable Servidor

Grafana utiliza:

```text
instance
```

Ejemplo:

```text
mail01
```

---

## 53. Variable Log

Grafana utiliza:

```text
log_type
```

El dashboard actual muestra como opciones:

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

No todos los valores tienen que existir en todos los servidores.

---

## 54. Postfix y Dovecot

Alloy recopila actualmente:

```text
postfix
dovecot
```

aunque estos valores no forman parte actualmente del selector visible
`log_type` documentado para el dashboard principal.

Por tanto, pueden consultarse directamente mediante LogQL:

```logql
{server="mail01",log_type="postfix"}
```

```logql
{server="mail01",log_type="dovecot"}
```

Si en el futuro se desea acceder a ellos desde el selector principal, deberán
añadirse explícitamente a la variable `log_type` de Grafana.

---

# SEGURIDAD

---

## 55. Principio de mínimo acceso

Alloy debe disponer únicamente de los permisos necesarios para leer las fuentes
configuradas.

No debe utilizarse:

```text
chmod 777
```

como solución a problemas de permisos.

Para archivos pertenecientes a grupos adecuados puede utilizarse pertenencia
a grupos.

Ejemplo:

```text
alloy → grupo adm → /var/log/fail2ban.log
```

Para casos específicos pueden utilizarse:

```text
ACL
```

---

## 56. Logs como información sensible

Los logs pueden contener:

```text
IP
usuarios
hostnames
direcciones de correo
rutas internas
errores
información operativa
```

No deben compartirse registros completos públicamente sin revisar su
contenido.

---

## 57. Producción

En `mail01` debe aplicarse siempre:

```text
cambio mínimo
       ↓
solo Alloy si afecta a Alloy
       ↓
validar
       ↓
mantener servicios de correo intactos
```

---

# ESTADO ACTUAL

---

## 58. Alloy

```text
mail01                      OPERATIVO
Loki                        OPERATIVO
Grafana                     OPERATIVO
```

---

## 59. Logs de sistema mail01

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

## 60. Mailcow

```text
Postfix                     OPERATIVO
Dovecot                     OPERATIVO
Rspamd                      OPERATIVO
ClamAV                      OPERATIVO
```

---

## 61. Pendiente

```text
Descubrimiento dinámico de contenedores Docker
```

Objetivo:

```text
eliminar dependencia de IDs estáticos
```

Prioridad:

```text
mejora técnica
```

No debe sustituirse la configuración funcional de producción hasta que la
alternativa haya sido probada.

---

## 62. Reglas permanentes

1. `mail01` es producción.
2. No reiniciar Mailcow por cambios exclusivos de Alloy.
3. Identificar siempre el origen real del log.
4. Mantener etiquetas `server` y `log_type` coherentes.
5. No utilizar permisos globales para resolver problemas de lectura.
6. Mantener el acceso a Fail2ban mediante el grupo `adm`.
7. Mantener las ACL necesarias para los logs Docker.
8. Recordar que los IDs de contenedores pueden cambiar.
9. No retirar una configuración funcional hasta validar su sustitución.
10. Mantener `mqh01` fuera de esta arquitectura de Alloy mientras utilice
    CloudWatch Logs.

---

## 63. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
alloy-logs.md
node-exporter.md
servidores.md
seguridad.md
recuperacion.md
```

`alloy.md` documenta el funcionamiento operativo del agente.

`alloy-logs.md` contiene la configuración específica de las fuentes de logs.

`loki.md` documenta el backend central de registros.

`grafana.md` documenta las consultas del dashboard.

`seguridad.md` contiene las decisiones de seguridad relacionadas con
producción.
