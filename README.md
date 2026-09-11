# Monitorización de Infraestructura

Documentación técnica de la plataforma centralizada de monitorización y observabilidad de la infraestructura.

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## Objetivo

Centralizar desde Fedora44 la supervisión de servidores Linux de laboratorio y producción, incluyendo:

- métricas del sistema;
- estado de los servidores;
- CPU, RAM y almacenamiento;
- Load Average;
- uptime;
- tráfico de red;
- logs;
- eventos de seguridad;
- métricas y logs procedentes de AWS.

Grafana actúa como interfaz principal de consulta.

---

## Arquitectura actual

La plataforma utiliza:

| Componente | Función | Estado |
|---|---|---|
| Grafana | Visualización | OPERATIVO |
| Prometheus | Métricas | OPERATIVO |
| Loki | Logs | OPERATIVO |
| Node Exporter | Métricas Linux | OPERATIVO |
| YACE | CloudWatch → Prometheus | OPERATIVO |
| AWS CloudWatch | Métricas y logs de mqh01 | OPERATIVO |
| Ansible | Administración | OPERATIVO |

---

## Flujo de métricas Linux

Los servidores Linux monitorizados mediante Node Exporter utilizan:

```text
lab01 ──────┐
python ─────┤
security ───┼── Node Exporter ── Prometheus ── Grafana
mail01 ─────┘
```

Prometheus utiliza:

```text
job="lab-servers"
```

y la etiqueta:

```text
server="<servidor>"
```

---

## Flujo de métricas AWS

`mqh01` utiliza una arquitectura diferente:

```text
mqh01
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

No se utiliza Node Exporter en `mqh01`.

---

## Servidores monitorizados

| Servidor | Entorno | Plataforma | Métricas | Grafana |
|---|---|---|---|---|
| `lab01` | Laboratorio | Linux | Node Exporter | OK |
| `python` | Laboratorio | Linux | Node Exporter | OK |
| `security` | Laboratorio | Linux / Seguridad | Node Exporter | OK |
| `mail01` | Producción | Contabo | Node Exporter | OK |
| `mqh01` | Producción | AWS EC2 | CloudWatch + YACE | OK |

---

## Dashboard principal

Dashboard:

```text
Monitorización Laboratorios
```

El diseño utiliza un único dashboard dinámico.

Selector principal:

```text
Servidor
```

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

Segundo selector:

```text
Log
```

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

## Información disponible

Para cada servidor se intenta presentar una única vista con:

```text
Información del servidor
CPU
RAM
Disco
Load Average
Uptime
Estado
Red
Logs
```

Principio de diseño:

```text
un servidor seleccionado
        ↓
un valor por métrica
```

Se evitan paneles duplicados por servidor.

---

## Logs

La plataforma utiliza dos fuentes principales de logs.

### Loki

Loki centraliza los registros de los servidores integrados mediante agentes de logs.

Consulta base utilizada en Grafana:

```logql
{server="$instance", log_type="$log_type"}
```

### AWS CloudWatch Logs

`mqh01` utiliza AWS CloudWatch Logs.

Grafana accede mediante el datasource:

```text
cloudwatch-1
```

Entre los logs disponibles para `mqh01` se encuentran:

```text
apache
ssh
fail2ban
errors
kernel
```

La consulta de CloudWatch Logs utilizada en Grafana filtra tanto por servidor como por tipo de log para impedir que los registros de `mqh01` aparezcan al seleccionar otro servidor.

---

## Seguridad

Los servidores se clasifican en:

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

Los servidores de producción no deben utilizarse para pruebas generales.

### SSH de mail01

En `mail01` se ha endurecido SSH.

Configuración efectiva verificada:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Por tanto:

- acceso directo de root: desactivado;
- autenticación SSH mediante contraseña: desactivada;
- autenticación keyboard-interactive: desactivada;
- autenticación mediante clave pública: activada.

Se comprobó posteriormente una nueva conexión SSH mediante clave pública.

Fail2ban permanece operativo y protege el servicio SSH frente a intentos automatizados.

---

## Estructura de documentación

```text
/datos/docs/monitoring/
│
├── README.md
├── estado-actual.md
├── arquitectura.md
├── grafana.md
├── prometheus.md
├── loki.md
├── alloy.md
├── alloy-logs.md
├── node-exporter.md
├── servidores.md
├── seguridad.md
└── recuperacion.md
```

### Documentos principales

- `estado-actual.md` — fuente rápida del estado operativo real.
- `arquitectura.md` — arquitectura y flujos de la plataforma.
- `grafana.md` — dashboard, variables, paneles y consultas.
- `prometheus.md` — jobs, targets y PromQL.
- `loki.md` — almacenamiento y consulta de logs.
- `alloy.md` — agente Grafana Alloy.
- `alloy-logs.md` — configuración relacionada con logs.
- `node-exporter.md` — métricas Linux mediante Node Exporter.
- `servidores.md` — información individual de cada servidor.
- `seguridad.md` — decisiones y procedimientos de seguridad.
- `recuperacion.md` — reconstrucción y recuperación de la plataforma.

---

## Ubicaciones principales

Documentación:

```text
/datos/docs/monitoring/
```

Prometheus:

```text
/datos/stacks/prometheus/
```

Loki:

```text
/datos/stacks/loki/
```

Grafana:

```text
/datos/stacks/grafana/
```

YACE:

```text
/datos/stacks/yace/
```

Ansible:

```text
/datos/ansible/masqhosting-ops/
```

---

## Estado resumido

```text
Fedora44
  Grafana                    OPERATIVO
  Prometheus                 OPERATIVO
  Loki                       OPERATIVO
  YACE                       OPERATIVO

lab01
  Monitorización             OPERATIVA

python
  Monitorización             OPERATIVA

security
  Monitorización             OPERATIVA

mail01
  Node Exporter              OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO
  Fail2ban                   OPERATIVO
  SSH por contraseña         DESACTIVADO
  SSH por clave              OPERATIVO

mqh01
  AWS CloudWatch             OPERATIVO
  CloudWatch Agent           OPERATIVO
  YACE                       OPERATIVO
  Prometheus                 UP
  Grafana                    OPERATIVO
  Métricas sistema           OPERATIVAS
  Logs CloudWatch            OPERATIVOS
```

---

## Regla de documentación

Cuando se realice un cambio importante:

1. comprobar que funciona;
2. documentar la configuración definitiva;
3. actualizar `estado-actual.md`;
4. actualizar el documento específico del componente;
5. actualizar este README únicamente si cambia la arquitectura general.

No deben almacenarse en esta documentación:

- contraseñas;
- claves privadas;
- AWS Secret Access Keys;
- tokens;
- credenciales;
- secretos de producción.

---

## Próximos trabajos

Pendientes principales:

- configurar alertas;
- exportar y versionar el dashboard de Grafana;
- completar el procedimiento de recuperación;
- revisar backups de configuración;
- revisar políticas de retención de métricas y logs.
