# Seguridad — Plataforma de Monitorización

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Este documento recoge los principios, decisiones y procedimientos de seguridad
relacionados con la plataforma de monitorización.

La infraestructura contiene servidores de laboratorio y servidores de
producción.

La seguridad debe aplicarse teniendo en cuenta esta separación.

---

## 2. Clasificación de servidores

### Laboratorio

```text
lab01
python
security
```

Estos servidores pueden utilizarse para:

- pruebas;
- aprendizaje;
- validación de configuraciones;
- experimentación;
- despliegues de laboratorio.

### Producción

```text
mail01
mqh01
```

Estos servidores alojan servicios reales.

No deben utilizarse para pruebas generales.

---

## 3. Principio fundamental

Antes de realizar cualquier cambio debe determinarse:

```text
¿Es laboratorio o producción?
```

Si es producción:

```text
¿Qué servicio se modifica?
¿Qué impacto puede tener?
¿Es necesario reiniciarlo?
¿Existe una alternativa mediante reload?
¿Cómo se revierte?
```

No deben realizarse reinicios o cambios generales cuando únicamente sea
necesario modificar un componente concreto.

---

# ACCESO SSH

---

## 4. Autenticación recomendada

Para servidores accesibles mediante SSH se prioriza:

```text
clave pública
```

frente a:

```text
contraseña
```

Cuando sea posible:

```text
PubkeyAuthentication yes
PasswordAuthentication no
```

---

## 5. Root por SSH

El acceso directo de root mediante SSH debe permanecer deshabilitado salvo
necesidad técnica explícita.

Configuración:

```text
PermitRootLogin no
```

La administración debe realizarse mediante un usuario autorizado y `sudo`.

---

# MAIL01

---

## 6. Importancia de mail01

`mail01` es un servidor de producción.

Proveedor:

```text
Contabo
```

IP:

```text
5.189.147.4
```

Además de sus servicios habituales, está integrado con:

```text
Node Exporter
Prometheus
Grafana
Loki
Fail2ban
```

Los cambios de seguridad deben realizarse con especial precaución para evitar
la pérdida de acceso remoto.

---

## 7. Situación detectada

Durante la revisión de los logs SSH de `mail01` se observó un volumen elevado
de intentos automáticos de autenticación.

Fail2ban mostraba cientos de miles de intentos fallidos acumulados y decenas de
miles de bloqueos históricos.

Este tipo de actividad es habitual en servidores SSH expuestos a Internet.

Los bots prueban automáticamente:

```text
usuarios comunes
contraseñas
credenciales filtradas
cuentas administrativas
```

El volumen de intentos no significa por sí mismo que exista una intrusión.

---

## 8. Fail2ban

Jail utilizado:

```text
sshd
```

Durante la revisión Fail2ban estaba:

```text
ACTIVO
```

y bloqueando direcciones IP.

Se observaron valores históricos superiores a:

```text
400.000 intentos fallidos
29.000 bloqueos
```

Estos valores son acumulados y pueden seguir aumentando.

Lo importante es comprobar que:

```text
Fail2ban está activo
el jail sshd está activo
las IP ofensivas son bloqueadas
```

---

## 9. Configuración SSH inicialmente efectiva

La configuración efectiva mostraba:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
```

El problema era:

```text
passwordauthentication yes
```

aunque el archivo principal de OpenSSH aparentemente ya desactivaba las
contraseñas.

---

## 10. Archivo principal

Se revisó:

```text
/etc/ssh/sshd_config
```

y contenía:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
```

Por tanto, el archivo principal no explicaba:

```text
passwordauthentication yes
```

en la configuración efectiva.

---

## 11. Descubrimiento del override

Se buscaron todas las definiciones mediante:

```bash
sudo grep -Rni 'PasswordAuthentication' \
  /etc/ssh/sshd_config \
  /etc/ssh/sshd_config.d/
```

Se encontró:

```text
/etc/ssh/sshd_config:57:PasswordAuthentication no
/etc/ssh/sshd_config.d/50-cloud-init.conf:1:PasswordAuthentication yes
```

El archivo:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

estaba sobrescribiendo el comportamiento esperado.

---

## 12. cloud-init

El drop-in de cloud-init contenía:

```text
PasswordAuthentication yes
```

Se modificó a:

```text
PasswordAuthentication no
```

Este detalle es importante para futuros diagnósticos.

No basta con revisar únicamente:

```text
/etc/ssh/sshd_config
```

También deben comprobarse:

```text
/etc/ssh/sshd_config.d/
```

---

## 13. Configuración efectiva final

Después del cambio se verificó:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Estado final:

```text
Root SSH                     DESACTIVADO
Contraseña SSH               DESACTIVADA
Keyboard Interactive         DESACTIVADO
Clave pública                ACTIVADA
```

---

## 14. Validación del acceso

Después de modificar SSH no se cerró la sesión existente inmediatamente.

Desde Fedora44 se abrió una nueva terminal y se realizó:

```bash
ssh mail
```

La conexión mediante clave pública funcionó correctamente.

Esto confirmó que el endurecimiento no había provocado pérdida de acceso.

---

## 15. Regla para futuros cambios SSH

Cuando se modifique SSH en un servidor remoto:

1. mantener abierta la sesión actual;
2. modificar únicamente lo necesario;
3. validar la sintaxis;
4. aplicar reload cuando sea suficiente;
5. abrir una segunda terminal;
6. comprobar una nueva conexión;
7. cerrar la sesión antigua únicamente cuando la nueva funcione.

Nunca debe cerrarse la única sesión disponible antes de comprobar el nuevo
acceso.

---

## 16. Configuración efectiva

Para diagnosticar OpenSSH es más fiable consultar la configuración efectiva que
limitarse a leer un único archivo.

Deben tenerse en cuenta:

```text
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/*.conf
```

Los drop-ins pueden modificar valores definidos en otros archivos.

---

## 17. Interpretación de logs SSH

Mensajes como:

```text
Failed password
Invalid user
preauth
```

indican intentos de autenticación o conexiones que no se completaron
correctamente.

No equivalen a una autenticación exitosa.

Los eventos exitosos pueden incluir:

```text
Accepted publickey
```

La investigación de un posible acceso debe distinguir siempre entre:

```text
intento
fallo
bloqueo
autenticación aceptada
sesión iniciada
```

---

# FAIL2BAN

---

## 18. Función

Fail2ban analiza registros de servicios y bloquea temporalmente direcciones IP
que realizan determinados patrones de intentos repetidos.

En `mail01` protege:

```text
sshd
```

---

## 19. Fail2ban no sustituye la configuración SSH

Fail2ban es una capa adicional.

La seguridad no debe depender exclusivamente de él.

La combinación utilizada es:

```text
Root SSH deshabilitado
        +
Contraseña deshabilitada
        +
Clave pública
        +
Fail2ban
```

---

## 20. Ataques automatizados

Un servidor SSH público seguirá recibiendo intentos aunque la autenticación por
contraseña esté desactivada.

Por tanto, después del endurecimiento pueden seguir apareciendo:

```text
Invalid user
preauth
connection closed
```

Esto no significa que la configuración haya dejado de funcionar.

---

# NODE EXPORTER

---

## 21. Seguridad de Node Exporter

Node Exporter normalmente escucha en:

```text
TCP 9100
```

Este endpoint proporciona métricas del sistema.

No debe exponerse indiscriminadamente a Internet.

Debe permitirse únicamente el acceso necesario desde el sistema de
monitorización.

---

## 22. mqh01 y Node Exporter

Durante el diseño se probó Node Exporter en `mqh01`.

La prueba requería proporcionar conectividad desde Fedora44 hasta:

```text
TCP 9100
```

Fedora44 dispone de una IP pública dinámica.

Se descartó mantener esta arquitectura para evitar:

```text
exposición pública
reglas dependientes de IP dinámica
mayor superficie de ataque
mantenimiento innecesario
```

La prueba se revirtió.

---

## 23. Arquitectura segura de mqh01

La solución definitiva utiliza:

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

De esta forma no es necesario que Fedora44 realice un scrape directo de un
exporter en `mqh01`.

---

# LOKI

---

## 24. Seguridad de Loki

Loki es un componente interno.

No debe exponerse directamente a Internet salvo que exista una necesidad
arquitectónica explícita y una protección adecuada.

Dentro de Docker se utiliza:

```text
http://loki:3100
```

mediante la red:

```text
monitoring
```

---

## 25. Logs sensibles

Los registros pueden contener información como:

```text
direcciones IP
usuarios
rutas
errores internos
hostnames
datos operativos
```

Por ello deben tratarse como información sensible de infraestructura.

No deben publicarse logs completos sin revisar previamente su contenido.

---

# PROMETHEUS

---

## 26. Seguridad de Prometheus

Prometheus centraliza información sobre:

```text
CPU
RAM
discos
interfaces
hostnames
servicios
estado de servidores
```

Su interfaz no debe exponerse innecesariamente a Internet.

El acceso debe limitarse al entorno administrativo.

---

# GRAFANA

---

## 27. Seguridad de Grafana

Grafana concentra información procedente de toda la infraestructura.

Debe protegerse mediante:

```text
autenticación
contraseñas seguras
actualizaciones
control de acceso
mínima exposición necesaria
```

Los dashboards pueden revelar información operativa aunque no contengan
credenciales.

---

# AWS

---

## 28. IAM

Las integraciones AWS deben seguir el principio de:

```text
mínimo privilegio
```

YACE únicamente necesita permisos de lectura para descubrir y consultar las
métricas necesarias.

No deben concederse permisos administrativos si no son necesarios.

---

## 29. YACE

YACE utiliza credenciales AWS de lectura.

Estas credenciales permiten consultar CloudWatch.

Deben mantenerse fuera de:

```text
Git
Markdown
capturas públicas
documentación
scripts compartidos
```

---

## 30. Incidente de exposición de credenciales

Durante la configuración de YACE se utilizó un comando de Docker Compose que
mostró la configuración expandida.

La salida incluía una credencial AWS sensible.

La credencial fue posteriormente rotada.

Lección operativa:

```text
No ejecutar ni compartir comandos que expandan variables sensibles sin revisar
previamente su salida.
```

---

## 31. Docker Compose y secretos

Debe tenerse especial precaución con comandos capaces de mostrar:

```text
environment
.env
variables expandidas
credenciales
```

No debe copiarse su salida completa a:

```text
chats
documentación
tickets
GitHub
foros
```

---

## 32. Archivo .env

Los secretos de YACE se mantienen fuera de la documentación.

El archivo `.env` debe tener permisos restrictivos.

Permisos utilizados:

```text
600
```

Esto significa:

```text
propietario → lectura/escritura
grupo       → sin acceso
otros       → sin acceso
```

---

## 33. IAM ListAccountAliases

Durante la integración puede aparecer un warning relacionado con:

```text
iam:ListAccountAliases
```

Este aviso no impide actualmente el funcionamiento de YACE.

No debe ampliarse una política IAM únicamente para eliminar un warning si la
funcionalidad necesaria ya funciona.

---

# CLOUDWATCH

---

## 34. CloudWatch Agent

CloudWatch Agent se ejecuta en `mqh01`.

Recopila:

```text
memoria
disco
métricas personalizadas
logs
```

Los cambios en CloudWatch Agent deben realizarse con cuidado porque `mqh01` es
producción.

---

## 35. Configuración CloudWatch Agent

La configuración activa se transforma internamente a TOML.

Debe evitarse ejecutar repetidamente `fetch-config` utilizando como origen un
archivo generado dentro de:

```text
/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/
```

Durante pruebas anteriores esto provocó nombres recursivos como:

```text
file_file_...
```

La configuración debe mantenerse controlada y no regenerarse
innecesariamente.

---

# LOGS DE MQH01

---

## 36. Logs recopilados

Actualmente:

```text
apache
ssh
fail2ban
errors
kernel
```

No se recopilan todos los logs disponibles.

La recopilación debe responder a una necesidad operativa.

---

## 37. Retención

Retención actual:

```text
apache       30 días
kernel       30 días
errors       90 días
fail2ban     90 días
ssh          90 días
```

Los periodos pueden revisarse según:

```text
seguridad
coste
utilidad
requisitos legales
necesidades operativas
```

---

# DOCKER

---

## 38. Reinicios

No debe utilizarse:

```text
reiniciar todo Docker
```

como solución genérica.

Debe identificarse el componente afectado.

Ejemplos:

```text
cambio YACE       → reiniciar YACE
cambio Prometheus → reiniciar Prometheus
cambio Loki       → reiniciar Loki
```

Un cambio en Grafana no justifica reiniciar WordPress.

---

## 39. Producción WordPress

`mqh01` aloja WordPress de producción.

No deben reiniciarse:

```text
WordPress
Apache
base de datos
contenedores de aplicación
```

por cambios relacionados únicamente con:

```text
Grafana
Prometheus
YACE
consultas
dashboard
```

---

# MONITORIZACIÓN

---

## 40. La monitorización no debe aumentar innecesariamente la superficie de ataque

Antes de abrir un puerto para monitorización debe preguntarse:

```text
¿Puede obtenerse la información sin abrir este puerto?
```

Ejemplo real:

```text
Node Exporter público en mqh01
```

fue sustituido por:

```text
CloudWatch → YACE
```

---

## 41. Separación de funciones

La arquitectura mantiene:

### Métricas Linux

```text
Node Exporter → Prometheus
```

### Métricas AWS

```text
CloudWatch → YACE → Prometheus
```

### Logs Linux

```text
Alloy → Loki
```

### Logs AWS

```text
CloudWatch Logs → Grafana
```

No debe añadirse complejidad sin una necesidad concreta.

---

# PROCEDIMIENTO DE CAMBIO

---

## 42. Antes de modificar producción

Determinar:

```text
servidor
servicio
configuración
impacto
dependencias
método de aplicación
método de reversión
```

---

## 43. Durante el cambio

Aplicar únicamente:

```text
el cambio mínimo necesario
```

Evitar modificar simultáneamente varios componentes si no es imprescindible.

Esto facilita identificar la causa de cualquier problema.

---

## 44. Después del cambio

En cambios de infraestructura o seguridad se debe verificar el comportamiento
real antes de considerar terminado el trabajo.

En SSH:

```text
abrir nueva conexión
```

En monitorización:

```text
comprobar que la métrica aparece
```

En logs:

```text
comprobar que llegan eventos
```

En producción:

```text
comprobar que el servicio sigue disponible
```

---

# ESTADO DE SEGURIDAD

---

## 45. mail01

```text
Producción                    SÍ
Root SSH                      DESACTIVADO
PasswordAuthentication       DESACTIVADO
KbdInteractiveAuthentication DESACTIVADO
PubkeyAuthentication         ACTIVADO
Acceso mediante clave         OPERATIVO
Fail2ban                      OPERATIVO
Jail sshd                     OPERATIVO
Monitorización                OPERATIVA
```

---

## 46. mqh01

```text
Producción                    SÍ
AWS EC2                       OPERATIVO
CloudWatch                    OPERATIVO
CloudWatch Agent              OPERATIVO
YACE                          OPERATIVO
Node Exporter público         NO
Logs CloudWatch               OPERATIVOS
Credencial YACE               ROTADA
.env                          PROTEGIDO
```

---

## 47. Fedora44

```text
Función                       NODO CENTRAL
Grafana                       OPERATIVO
Prometheus                    OPERATIVO
Loki                          OPERATIVO
YACE                          OPERATIVO
Red Docker monitoring         OPERATIVA
```

Fedora44 constituye un punto importante de administración y observabilidad.

Debe protegerse porque centraliza acceso e información sobre varios
servidores.

---

# REGLAS PERMANENTES

---

## 48. Reglas

1. `mail01` y `mqh01` son producción.
2. No incluir producción automáticamente en pruebas de laboratorio.
3. Priorizar autenticación SSH mediante clave.
4. Mantener root SSH deshabilitado.
5. Mantener PasswordAuthentication deshabilitado donde se haya establecido.
6. Revisar `sshd_config.d` además de `sshd_config`.
7. Mantener Fail2ban activo en servidores donde se utilice.
8. No exponer Node Exporter públicamente sin necesidad.
9. No exponer Loki públicamente sin necesidad.
10. No exponer Prometheus públicamente sin necesidad.
11. Utilizar mínimo privilegio en AWS IAM.
12. No almacenar secretos en Markdown.
13. No almacenar secretos en Git.
14. No compartir salidas que contengan variables sensibles.
15. No reiniciar servicios de producción innecesariamente.
16. Utilizar reload cuando sea suficiente.
17. Aplicar cambios mínimos.
18. Comprobar el acceso antes de cerrar una sesión SSH existente.
19. Documentar los problemas difíciles de diagnosticar.
20. Rotar inmediatamente cualquier credencial que pueda haber quedado
    expuesta.

---

## 49. Lecciones importantes

### OpenSSH

```text
La configuración efectiva puede no coincidir con lo que aparece en
/etc/ssh/sshd_config.
```

Revisar siempre:

```text
/etc/ssh/sshd_config.d/
```

### AWS

```text
Un warning no justifica aumentar permisos automáticamente.
```

### Docker

```text
Una salida de diagnóstico puede contener secretos.
```

### Monitorización

```text
No todo exporter necesita estar expuesto directamente.
```

### Producción

```text
Si el cambio no afecta al servicio, no tocar el servicio.
```

---

## 50. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
prometheus.md
loki.md
servidores.md
recuperacion.md
```

`seguridad.md` contiene las decisiones y procedimientos de seguridad.

`servidores.md` identifica qué máquinas son laboratorio y cuáles son
producción.

`recuperacion.md` contiene los procedimientos de recuperación.

`estado-actual.md` representa la referencia rápida del estado operativo.
