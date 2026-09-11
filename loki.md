# Loki — Centralización de Logs

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Loki es el sistema central de almacenamiento y consulta de logs de la
plataforma de monitorización.

Su función es recibir registros de los servidores integrados con el sistema
centralizado de logs y permitir su consulta desde Grafana.

La arquitectura actual utiliza dos caminos para los registros:

```text
Servidores compatibles → Alloy → Loki → Grafana

mqh01 → AWS CloudWatch Logs → Grafana
```

Por tanto, Loki no es la única fuente de logs del dashboard.

Grafana unifica ambas fuentes desde la interfaz de monitorización.

---

## 2. Arquitectura general de logs

```text
                       ┌─────────────────────┐
                       │       Grafana       │
                       │      Fedora44       │
                       └─────────▲───────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                   Loki              AWS CloudWatch Logs
                    ▲                         ▲
                    │                         │
                  Alloy                     mqh01
                    ▲
                    │
             servidores Linux
```

Las métricas y los logs siguen arquitecturas independientes.

Las métricas utilizan principalmente:

```text
Node Exporter → Prometheus
CloudWatch → YACE → Prometheus
```

Los logs utilizan:

```text
Alloy → Loki
CloudWatch Logs
```

---

## 3. Ubicación

Stack de Loki:

```text
/datos/stacks/loki/
```

Loki se ejecuta mediante Docker en Fedora44.

---

## 4. Contenedor

Imagen utilizada:

```text
grafana/loki:3.7.6
```

Puerto interno:

```text
3100
```

Endpoint utilizado dentro de la red Docker:

```text
http://loki:3100
```

---

## 5. Red Docker

Loki participa en la red:

```text
monitoring
```

Los componentes deben utilizar el nombre DNS interno:

```text
loki
```

en lugar de depender de una dirección IP interna Docker.

Las IP de los contenedores pueden cambiar al recrearlos.

---

## 6. Flujo Loki

El flujo general es:

```text
Servidor
   │
   ▼
Logs locales
   │
   ▼
Grafana Alloy
   │
   ▼
Loki
   │
   ▼
Grafana
```

Alloy se encarga de:

- leer los logs;
- asignar etiquetas;
- identificar el servidor;
- identificar el tipo de registro;
- enviar los eventos a Loki.

Loki se encarga de almacenarlos y permitir consultas mediante LogQL.

---

## 7. Etiquetas principales

La arquitectura utiliza principalmente:

```text
server
log_type
```

Ejemplo conceptual:

```text
server="mail01"
log_type="ssh"
```

Estas etiquetas permiten utilizar el mismo panel de Grafana para distintos
servidores y tipos de registro.

---

## 8. Consulta principal

La consulta base utilizada en Grafana es:

```logql
{server="$instance", log_type="$log_type"}
```

Variables:

```text
$instance  → servidor seleccionado
$log_type  → tipo de log seleccionado
```

Ejemplo:

```text
Servidor = mail01
Log      = ssh
```

produce conceptualmente:

```logql
{server="mail01", log_type="ssh"}
```

---

## 9. Variable Servidor

Variable Grafana:

```text
instance
```

Servidores:

```text
lab01
python
security
mail01
mqh01
```

La misma variable se utiliza tanto para métricas como para logs.

---

## 10. Variable Log

Variable:

```text
log_type
```

Valores disponibles:

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

No todos los servidores tienen necesariamente todos estos tipos de log.

Si un servidor no dispone del tipo seleccionado, el panel puede no mostrar
registros.

Esto es comportamiento normal.

---

## 11. Grafana Alloy

Grafana Alloy actúa como agente de recopilación de logs para los servidores
integrados con Loki.

La documentación específica se encuentra en:

```text
alloy.md
alloy-logs.md
```

La responsabilidad de Alloy es mantener una estructura coherente de
etiquetas antes de enviar los registros a Loki.

---

## 12. Filosofía de recopilación

No se pretende enviar todos los archivos existentes de `/var/log` a Loki.

Se recopilan únicamente registros con utilidad operativa o de seguridad.

Principio:

```text
recoger logs útiles
        ↓
etiquetarlos correctamente
        ↓
centralizarlos
        ↓
consultarlos desde Grafana
```

Esto evita almacenar ruido innecesario.

---

# MQUH01 / AWS CLOUDWATCH LOGS

---

## 13. mqh01

`mqh01` utiliza una arquitectura diferente para los logs.

No depende de Loki para mostrar sus registros en Grafana.

Flujo:

```text
mqh01
  │
  ▼
CloudWatch Agent / aplicaciones
  │
  ▼
AWS CloudWatch Logs
  │
  ▼
Grafana
```

Esto aprovecha la integración nativa de la instancia AWS con CloudWatch.

---

## 14. Datasource CloudWatch

Grafana dispone del datasource:

```text
cloudwatch-1
```

Región:

```text
eu-south-2
```

Este datasource se utiliza directamente para consultar los registros de
`mqh01`.

---

## 15. Log groups de mqh01

Los grupos utilizados actualmente incluyen:

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

---

## 16. Categorías de logs de mqh01

Las categorías previstas son:

| log_type | Contenido |
|---|---|
| `apache` | Apache / WordPress |
| `ssh` | conexiones OpenSSH |
| `fail2ban` | bloqueos y eventos Fail2ban |
| `errors` | errores del sistema |
| `kernel` | mensajes del kernel |

Actualmente no se pretende utilizar para `mqh01`:

```text
system
wordpress
docker
clamav
rspamd
hestia
cron
```

Esto puede ampliarse en el futuro si existe una necesidad operativa real.

---

## 17. Apache / WordPress

El WordPress de `mqh01` se ejecuta mediante Docker.

Apache envía sus registros mediante:

```text
stdout
stderr
```

Estos registros llegan al grupo:

```text
/masqhosting/mqh01/wordpress/apache
```

Stream:

```text
mqh01-wordpress
```

No es necesario modificar ni reiniciar WordPress para las consultas normales
desde Grafana.

---

## 18. SSH

Los eventos SSH de `mqh01` se recopilan en:

```text
/var/log/ssh.log
```

rsyslog dispone de una regla específica para los programas:

```text
sshd
sshd-session
```

Configuración:

```text
/etc/rsyslog.d/30-masqhosting-monitoring.conf
```

Regla:

```text
if ($programname == 'sshd' or $programname == 'sshd-session') then /var/log/ssh.log
```

CloudWatch Agent recoge posteriormente este archivo.

---

## 19. Errores

La misma configuración de rsyslog genera:

```text
/var/log/errors.log
```

mediante:

```text
*.err /var/log/errors.log
```

Estos registros se envían a:

```text
/masqhosting/mqh01/errors
```

---

## 20. Fail2ban

Archivo:

```text
/var/log/fail2ban.log
```

Grupo CloudWatch:

```text
/masqhosting/mqh01/fail2ban
```

Permite consultar desde Grafana eventos relacionados con bloqueos y actividad
de Fail2ban.

---

## 21. Kernel

Archivo:

```text
/var/log/kern.log
```

Grupo:

```text
/masqhosting/mqh01/kernel
```

---

## 22. Retención de logs de mqh01

Configuración actual:

```text
apache       30 días
kernel       30 días
errors       90 días
fail2ban     90 días
ssh          90 días
```

La retención puede revisarse posteriormente en función de:

- coste;
- utilidad;
- requisitos operativos;
- necesidades de seguridad.

---

# GRAFANA

---

## 23. Panel de logs

El dashboard principal utiliza un único panel de logs.

Variables:

```text
$instance
$log_type
```

El objetivo es evitar crear:

```text
panel SSH
panel Apache
panel Fail2ban
panel Kernel
panel Errors
```

para cada servidor.

Se mantiene un único panel dinámico.

---

## 24. Consulta Loki

Datasource:

```text
Loki
```

Consulta:

```logql
{server="$instance", log_type="$log_type"}
```

Cuando el servidor seleccionado utiliza Loki, esta consulta devuelve los
registros correspondientes.

---

## 25. Consulta CloudWatch

Datasource:

```text
cloudwatch-1
```

Log groups:

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

Consulta Logs Insights:

```text
fields @timestamp, @message
| filter @log like /$instance/
| filter @log like /$log_type/
| sort @timestamp desc
| limit 100
```

---

## 26. Filtro por servidor

Esta línea es importante:

```text
| filter @log like /$instance/
```

Los grupos CloudWatch de `mqh01` contienen:

```text
mqh01
```

en su nombre.

Por tanto:

```text
Servidor seleccionado = mqh01
```

permite mostrar los registros.

Si se selecciona:

```text
lab01
python
security
mail01
```

la consulta CloudWatch no devuelve resultados.

Esto evita mezclar los registros AWS con otros servidores.

---

## 27. Filtro por tipo de log

La línea:

```text
| filter @log like /$log_type/
```

selecciona el grupo correspondiente al tipo elegido.

Ejemplo:

```text
$log_type = ssh
```

coincide con:

```text
/masqhosting/mqh01/ssh
```

Ejemplo:

```text
$log_type = apache
```

coincide con:

```text
/masqhosting/mqh01/wordpress/apache
```

---

## 28. Resultado de la arquitectura híbrida

El funcionamiento conceptual del panel es:

```text
                    Servidor seleccionado
                            │
                    Tipo de log seleccionado
                            │
                  ┌─────────┴─────────┐
                  │                   │
                Loki             CloudWatch
                  │                   │
                  └─────────┬─────────┘
                            │
                            ▼
                       Panel Grafana
```

No es necesario que todos los servidores utilicen el mismo backend de logs.

---

# SEGURIDAD

---

## 29. Logs SSH y Fail2ban

Los logs de:

```text
ssh
fail2ban
```

son especialmente importantes para detectar:

- intentos de autenticación;
- usuarios inexistentes;
- bots;
- ataques automatizados;
- bloqueos de direcciones IP;
- actividad anómala.

---

## 30. mail01

`mail01` recibe un volumen elevado de intentos automatizados contra SSH.

Fail2ban está operativo.

La configuración efectiva de OpenSSH ha sido endurecida:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Por tanto, aunque los bots continúen intentando conexiones, la autenticación
mediante contraseña está desactivada.

Los logs siguen siendo útiles para observar y analizar esta actividad.

---

## 31. Diferencia entre intento y acceso

La presencia de entradas como:

```text
Failed password
Invalid user
preauth
```

no significa que se haya producido un acceso exitoso.

Un acceso real debe analizarse buscando eventos de autenticación aceptada,
por ejemplo:

```text
Accepted publickey
```

Los logs deben interpretarse en contexto y no únicamente por el volumen de
intentos.

---

# OPERACIÓN

---

## 32. Reinicio de Loki

Si únicamente se modifica Loki:

```bash
cd /datos/stacks/loki
sudo docker compose restart loki
```

No deben reiniciarse otros servicios sin necesidad.

---

## 33. Reinicio de Alloy

Si se modifica únicamente la configuración de Alloy, debe reiniciarse el
agente correspondiente al servidor afectado.

No debe utilizarse un reinicio general de toda la plataforma por un cambio
local de logs.

---

## 34. mqh01

Los cambios en las consultas de Grafana no requieren reiniciar:

```text
WordPress
Apache
CloudWatch Agent
Docker
```

Si únicamente cambia una consulta de Grafana, no debe tocarse el servidor de
producción.

---

## 35. Secretos

No deben almacenarse en los logs o documentación:

```text
contraseñas
tokens
claves privadas
AWS Secret Access Keys
credenciales completas
```

También debe evitarse copiar públicamente salidas de comandos que puedan
mostrar variables de entorno sensibles.

---

# DECISIONES TÉCNICAS

---

## 36. Loki no tiene que recibir todos los logs

No existe la obligación de enviar todos los servidores a Loki.

La arquitectura permite utilizar el backend más adecuado.

Actualmente:

```text
servidores Linux compatibles → Loki
mqh01                       → CloudWatch Logs
```

Grafana proporciona la capa común de visualización.

---

## 37. mqh01 permanece en CloudWatch Logs

No existe necesidad actual de duplicar los registros de `mqh01` enviándolos
también a Loki.

Eso provocaría:

```text
duplicación
más almacenamiento
más configuración
más puntos de fallo
```

Mientras CloudWatch Logs funcione correctamente, Grafana puede consultarlo
directamente.

---

## 38. Métricas y logs no deben confundirse

Para `mqh01`:

### Métricas

```text
mqh01
  ↓
CloudWatch
  ↓
YACE
  ↓
Prometheus
  ↓
Grafana
```

### Logs

```text
mqh01
  ↓
CloudWatch Logs
  ↓
Grafana
```

YACE únicamente participa en las métricas.

No participa en los logs.

---

## 39. Estado actual

```text
Loki
  Servicio                     OPERATIVO
  Docker                       OPERATIVO
  Red monitoring               OPERATIVA
  Grafana                      INTEGRADO

Consultas
  server                       OPERATIVA
  log_type                     OPERATIVA

mqh01
  CloudWatch Logs              OPERATIVO
  apache                       OPERATIVO
  ssh                          OPERATIVO
  fail2ban                     OPERATIVO
  errors                       OPERATIVO
  kernel                       OPERATIVO

Grafana
  Loki datasource              OPERATIVO
  CloudWatch datasource        OPERATIVO
  Panel dinámico de logs       OPERATIVO
```

---

## 40. Regla para nuevos logs

Antes de añadir un nuevo tipo de log debe determinarse:

1. qué información aporta;
2. qué servidor lo genera;
3. dónde se almacena originalmente;
4. si debe enviarse a Loki o CloudWatch;
5. qué valor tendrá `log_type`;
6. cuánto tiempo debe conservarse;
7. si puede contener información sensible.

No se deben añadir logs simplemente porque existan.

---

## 41. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
alloy.md
alloy-logs.md
servidores.md
seguridad.md
recuperacion.md
```

`loki.md` describe la arquitectura central de logs.

`alloy.md` y `alloy-logs.md` contienen los detalles del agente de
recopilación.

`grafana.md` contiene las consultas utilizadas en el dashboard.

`estado-actual.md` representa la referencia rápida del estado operativo.
