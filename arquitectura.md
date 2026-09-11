# Arquitectura del Sistema de Monitorización

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

La plataforma de monitorización tiene como objetivo centralizar en **Fedora44**
la supervisión de los servidores de laboratorio y producción.

Grafana actúa como interfaz principal y permite consultar desde un único
dashboard:

- información del servidor;
- CPU;
- memoria RAM;
- almacenamiento;
- Load Average;
- uptime;
- estado;
- tráfico de red;
- logs;
- eventos de seguridad.

La arquitectura admite servidores monitorizados mediante tecnologías
diferentes sin necesidad de crear dashboards independientes.

---

## 2. Principio de diseño

La arquitectura sigue este principio:

```text
un servidor seleccionado
        ↓
un dashboard
        ↓
un valor por métrica
```

Grafana oculta las diferencias existentes entre los distintos sistemas de
recopilación.

Por ejemplo:

```text
lab01  → Node Exporter
mqh01  → AWS CloudWatch + YACE
```

pero ambos aparecen dentro del mismo dashboard.

---

## 3. Nodo central

El nodo central es:

```text
Fedora44
```

Estructura principal:

```text
/datos/
├── ansible/
├── docker/
├── docs/
├── imagenes/
├── monitoring/
├── repos/
├── scripts/
└── stacks/
```

Documentación:

```text
/datos/docs/monitoring/
```

Stacks principales:

```text
/datos/stacks/grafana/
/datos/stacks/prometheus/
/datos/stacks/loki/
/datos/stacks/yace/
```

Administración mediante Ansible:

```text
/datos/ansible/masqhosting-ops/
```

---

## 4. Componentes

### Grafana

Interfaz central de visualización.

Consume información procedente de:

```text
Prometheus
Loki
AWS CloudWatch Logs
```

---

### Prometheus

Sistema central de métricas.

Recibe métricas desde:

```text
Node Exporter
YACE
```

---

### Node Exporter

Expone métricas Linux mediante:

```text
TCP 9100
```

Actualmente se utiliza en:

```text
lab01
python
security
mail01
```

---

### Loki

Sistema centralizado de almacenamiento y consulta de logs.

Grafana consulta Loki mediante la red Docker interna.

Dirección interna:

```text
http://loki:3100
```

---

### Grafana Alloy

Agente utilizado para recopilar y etiquetar registros que posteriormente se
envían a Loki.

Los logs almacenados en Loki utilizan etiquetas que permiten identificar el
servidor y el tipo de registro.

---

### YACE

YACE significa:

```text
Yet Another CloudWatch Exporter
```

Se utiliza para convertir métricas de AWS CloudWatch en métricas compatibles
con Prometheus.

Flujo:

```text
CloudWatch
    ↓
YACE
    ↓
Prometheus
```

---

### AWS CloudWatch

AWS CloudWatch proporciona la monitorización de `mqh01`.

Se utiliza para:

```text
métricas EC2
métricas CloudWatch Agent
métricas personalizadas
logs
```

---

### CloudWatch Agent

Se ejecuta en `mqh01`.

Recopila métricas adicionales que EC2 no proporciona directamente, entre
ellas:

```text
RAM
disco
Load Average
uptime
```

También recopila determinados logs del servidor.

---

### Ansible

Se utiliza para administración, despliegues y comprobaciones de servidores.

Proyecto:

```text
/datos/ansible/masqhosting-ops/
```

---

## 5. Red Docker de monitorización

Los componentes centrales utilizan la red Docker:

```text
monitoring
```

Entre ellos se encuentran:

```text
Grafana
Prometheus
Loki
YACE
```

Siempre que sea posible se utilizan nombres DNS internos de Docker en lugar
de direcciones IP de los contenedores.

Ejemplos:

```text
prometheus:9090
loki:3100
yace:5000
```

Las IP internas Docker pueden cambiar y no deben utilizarse como configuración
permanente salvo necesidad específica de diagnóstico.

---

## 6. Arquitectura general

La arquitectura actual puede representarse así:

```text
                           ┌───────────────────────┐
                           │       Grafana         │
                           │       Fedora44        │
                           └───────────▲───────────┘
                                       │
                     ┌─────────────────┼─────────────────┐
                     │                 │                 │
                     │                 │                 │
                Prometheus            Loki        CloudWatch Logs
                     ▲                 ▲                 ▲
                     │                 │                 │
          ┌──────────┴─────────┐       │                 │
          │                    │       │                 │
   Node Exporter              YACE    Alloy              │
          ▲                    ▲                         │
          │                    │                         │
 ┌────────┼────────┬───────┐   │                         │
 │        │        │       │   │                         │
lab01   python  security mail01 │                         │
                               │                         │
                               └──── AWS CloudWatch ─────┤
                                          ▲              │
                                          │              │
                                        mqh01 ───────────┘
```

---

## 7. Arquitectura de métricas Linux

Los servidores Linux convencionales utilizan:

```text
Servidor
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

Servidores:

```text
lab01
python
security
mail01
```

Prometheus utiliza:

```text
job="lab-servers"
```

Cada servidor dispone de:

```text
server="<nombre_servidor>"
```

Ejemplos:

```text
server="lab01"
server="python"
server="security"
server="mail01"
```

---

## 8. Arquitectura de métricas de mqh01

`mqh01` es una instancia AWS EC2 de producción.

No utiliza Node Exporter.

La arquitectura definitiva es:

```text
mqh01
  │
  ├── métricas EC2
  │
  ├── CloudWatch Agent
  │
  └── métricas StatsD
  │
  ▼
AWS CloudWatch
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

Prometheus identifica estas métricas mediante:

```text
job="aws-mqh01"
server="mqh01"
```

---

## 9. Motivo para no utilizar Node Exporter en mqh01

Durante el diseño inicial se probó Node Exporter en `mqh01`.

El exporter funcionaba correctamente dentro de EC2, pero la arquitectura
requería proporcionar acceso desde Fedora44 hacia TCP 9100.

Fedora44 utiliza una IP pública dinámica.

Se decidió evitar:

```text
exponer TCP 9100
depender de una IP doméstica dinámica
mantener reglas externas innecesarias
```

La prueba fue revertida completamente.

La solución definitiva utiliza servicios nativos de AWS:

```text
CloudWatch
CloudWatch Agent
YACE
```

---

## 10. YACE

YACE se ejecuta en Fedora44 mediante Docker.

Configuración:

```text
/datos/stacks/yace/config.yml
```

Endpoint interno:

```text
yace:5000
```

Prometheus realiza el scrape mediante:

```text
job="aws-mqh01"
```

Estado esperado:

```text
up{job="aws-mqh01"} = 1
```

---

## 11. Métricas EC2 de mqh01

Las métricas nativas de AWS utilizadas incluyen:

```text
CPUUtilization
StatusCheckFailed
NetworkIn
NetworkOut
```

Estas métricas son descubiertas por YACE dentro del namespace AWS/EC2.

---

## 12. CloudWatch Agent en mqh01

CloudWatch Agent amplía las métricas disponibles.

Namespaces utilizados:

```text
mqh01-memory
mqh01-disk
mqh01-system
```

### Memoria

Incluye:

```text
mem_used_percent
mem_total
```

### Disco

Incluye:

```text
disk_used_percent
disk_total
```

Para el filesystem raíz se utiliza:

```text
path="/"
```

### Sistema

Incluye:

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

---

## 13. StatsD en mqh01

CloudWatch Agent dispone de un listener StatsD:

```text
127.0.0.1:8125/UDP
```

Las métricas personalizadas de Load Average y uptime se generan mediante:

```text
/usr/local/bin/mqh01-system-metrics
```

El script se ejecuta cada minuto mediante:

```text
/etc/cron.d/mqh01-system-metrics
```

Las métricas StatsD utilizan las dimensiones:

```text
InstanceId
metric_type
```

Este detalle es importante para la configuración de YACE.

Si `metric_type` no está incluido en los requisitos de dimensiones, YACE puede
no descubrir estas métricas aunque existan correctamente en CloudWatch.

---

## 14. Tráfico de red

Los servidores con Node Exporter utilizan:

```text
node_network_receive_bytes_total
node_network_transmit_bytes_total
```

Se excluye:

```text
device="lo"
```

y se suma el tráfico del resto de interfaces para mostrar un único valor.

`mqh01` utiliza:

```text
NetworkIn
NetworkOut
```

CloudWatch proporciona estas métricas como suma durante un periodo de:

```text
300 segundos
```

Para obtener bytes por segundo:

```text
NetworkIn / 300
NetworkOut / 300
```

---

## 15. Arquitectura de logs

Los logs utilizan una arquitectura independiente de las métricas.

Existen actualmente dos caminos principales.

### Loki

```text
Servidor
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

### AWS CloudWatch Logs

Para `mqh01`:

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

Por tanto, Grafana puede consultar logs procedentes de distintas fuentes sin
necesidad de separar el dashboard por servidor.

---

## 16. Etiquetas de Loki

La consulta principal utiliza:

```logql
{server="$instance", log_type="$log_type"}
```

Las dos etiquetas fundamentales son:

```text
server
log_type
```

Esto permite que los selectores de Grafana controlen la información mostrada.

---

## 17. CloudWatch Logs de mqh01

Los grupos principales son:

```text
/masqhosting/mqh01/errors
/masqhosting/mqh01/fail2ban
/masqhosting/mqh01/kernel
/masqhosting/mqh01/ssh
/masqhosting/mqh01/wordpress/apache
```

Categorías utilizadas:

```text
errors
fail2ban
kernel
ssh
apache
```

No se pretende recopilar todos los logs existentes.

Se mantiene el principio:

```text
recoger únicamente logs útiles
```

---

## 18. Consulta de CloudWatch Logs desde Grafana

Consulta utilizada:

```text
fields @timestamp, @message
| filter @log like /$instance/
| filter @log like /$log_type/
| sort @timestamp desc
| limit 100
```

El filtro por `$instance` es importante porque evita mostrar registros de
`mqh01` cuando el dashboard tiene seleccionado otro servidor.

El filtro `$log_type` permite utilizar el mismo panel para diferentes tipos
de registros.

---

## 19. Variables de Grafana

El dashboard utiliza dos selectores visibles.

### Servidor

Variable:

```text
instance
```

Orden:

```text
lab01
python
security
mail01
mqh01
```

### Log

Variable:

```text
log_type
```

Orden:

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

## 20. Dashboard único

No se mantienen dashboards diferentes para:

```text
Node Exporter
AWS
CloudWatch
```

Grafana utiliza consultas capaces de seleccionar la fuente correcta.

Conceptualmente:

```text
Node Exporter
     │
     ├──── OR ────► Panel Grafana
     │
CloudWatch/YACE
```

Esto permite seleccionar `mqh01` igual que cualquier otro servidor.

---

## 21. Separación laboratorio y producción

### Laboratorio

```text
lab01
python
security
```

### Producción

```text
mail01
mqh01
```

Los cambios destinados a servidores de laboratorio no deben aplicarse
automáticamente a producción.

Especialmente:

```text
mail01
mqh01
```

no deben incluirse en despliegues o cambios destinados exclusivamente a
servidores de prueba salvo indicación explícita.

---

## 22. Seguridad de la arquitectura

Principios:

1. mínima exposición;
2. no abrir servicios internos innecesariamente;
3. separar laboratorio y producción;
4. proteger credenciales;
5. mantener secretos fuera de Git y Markdown;
6. utilizar IAM con permisos limitados;
7. evitar reglas públicas `0.0.0.0/0` innecesarias;
8. validar antes de reiniciar servicios;
9. mantener procedimientos de reversión;
10. documentar decisiones importantes.

---

## 23. Credenciales AWS

YACE necesita credenciales con permisos de lectura de CloudWatch.

Las credenciales deben almacenarse fuera de la documentación.

Nunca se deben incluir en:

```text
README
Markdown
Git
capturas públicas
salidas de comandos compartidas
```

Debe evitarse especialmente utilizar comandos que impriman variables de
entorno o secretos completos.

---

## 24. Seguridad SSH de mail01

`mail01` utiliza autenticación SSH mediante clave pública.

Configuración efectiva verificada:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Fail2ban protege el servicio SSH.

Durante el diagnóstico se descubrió que:

```text
/etc/ssh/sshd_config
```

tenía correctamente:

```text
PasswordAuthentication no
```

pero:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

estaba sobrescribiendo la configuración con:

```text
PasswordAuthentication yes
```

El drop-in de cloud-init fue corregido y posteriormente se verificó una nueva
conexión mediante clave pública.

---

## 25. Estado operativo

```text
Fedora44
  Grafana                    OPERATIVO
  Prometheus                 OPERATIVO
  Loki                       OPERATIVO
  YACE                       OPERATIVO
  Red Docker monitoring      OPERATIVA

lab01
  Node Exporter              OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO

python
  Node Exporter              OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO

security
  Node Exporter              OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO

mail01
  Node Exporter              OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO
  Fail2ban                   OPERATIVO
  SSH clave pública          OPERATIVO
  SSH contraseña             DESACTIVADO

mqh01
  AWS EC2                    OPERATIVO
  CloudWatch                 OPERATIVO
  CloudWatch Agent           OPERATIVO
  YACE                       OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO
  Métricas                   OPERATIVAS
  CloudWatch Logs            OPERATIVOS
```

---

## 26. Principios de recuperación

Para poder reconstruir la plataforma deben conservarse como mínimo:

```text
Docker Compose
configuración Grafana
dashboard Grafana
configuración Prometheus
configuración Loki
configuración YACE
configuración Alloy
documentación
scripts de métricas
configuración CloudWatch Agent
```

Los secretos deben respaldarse mediante un mecanismo independiente y seguro.

El procedimiento detallado se mantiene en:

```text
recuperacion.md
```

---

## 27. Pendientes

Las mejoras futuras incluyen:

- alertas de infraestructura;
- exportación y versionado definitivo del dashboard;
- backups automatizados;
- revisión de retención de métricas;
- revisión de retención de logs;
- pruebas periódicas de recuperación.

---

## 28. Regla de mantenimiento

Cuando cambie la arquitectura:

1. comprobar el funcionamiento real;
2. actualizar `estado-actual.md`;
3. actualizar este documento;
4. actualizar el documento específico del componente;
5. conservar únicamente la configuración definitiva;
6. documentar cualquier decisión de seguridad relevante.

`estado-actual.md` debe considerarse la referencia rápida del estado operativo.

Este documento describe la arquitectura técnica que permite alcanzar dicho
estado.
