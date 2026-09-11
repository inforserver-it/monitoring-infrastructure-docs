# Grafana — Monitorización Centralizada

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Grafana es la interfaz principal de visualización de la plataforma de
monitorización.

El dashboard principal permite seleccionar un servidor y visualizar desde una
única pantalla:

- información del sistema;
- CPU;
- RAM;
- disco;
- Load Average;
- uptime;
- estado;
- tráfico de red;
- logs.

La arquitectura permite combinar servidores monitorizados mediante Node
Exporter con `mqh01`, cuyas métricas proceden de AWS CloudWatch a través de
YACE.

---

## 2. Dashboard principal

Nombre:

```text
Monitorización Laboratorios
```

Principio de diseño:

```text
un servidor seleccionado
        ↓
un dashboard
        ↓
un valor por métrica
```

No se utilizan paneles duplicados para cada servidor.

---

## 3. Fuentes de datos

Grafana utiliza principalmente:

```text
Prometheus
Loki
AWS CloudWatch
```

### Prometheus

Prometheus proporciona las métricas de:

```text
lab01
python
security
mail01
mqh01
```

Para los cuatro primeros servidores las métricas proceden de Node Exporter.

Para `mqh01` proceden de:

```text
AWS CloudWatch
      ↓
     YACE
      ↓
 Prometheus
```

### Loki

Loki proporciona los logs de los servidores integrados mediante el sistema
centralizado de logs.

### AWS CloudWatch

Los logs de `mqh01` se consultan directamente desde AWS CloudWatch.

Datasource:

```text
cloudwatch-1
```

Región:

```text
eu-south-2
```

---

## 4. Variables visibles

El dashboard mantiene únicamente dos selectores visibles:

```text
Servidor
Log
```

---

## 5. Variable Servidor

Nombre interno:

```text
instance
```

Etiqueta:

```text
Servidor
```

Tipo:

```text
Custom
```

Contenido:

```json
[
  {"value":"lab01","text":"lab01","server":"lab01","aws":"none"},
  {"value":"python","text":"python","server":"python","aws":"none"},
  {"value":"security","text":"security","server":"security","aws":"none"},
  {"value":"mail01","text":"mail01","server":"mail01","aws":"none"},
  {"value":"mqh01","text":"mqh01","server":"mqh01","aws":"i-0d4beab843b2d2f95"}
]
```

Orden mostrado:

```text
lab01
python
security
mail01
mqh01
```

---

## 6. Variable Log

Nombre interno:

```text
log_type
```

Etiqueta:

```text
Log
```

Tipo:

```text
Custom
```

Contenido:

```json
[
  {"value":"ssh","text":"ssh"},
  {"value":"apache","text":"apache"},
  {"value":"fail2ban","text":"fail2ban"},
  {"value":"errors","text":"errors"},
  {"value":"kernel","text":"kernel"},
  {"value":"auth","text":"auth"},
  {"value":"system","text":"system"},
  {"value":"cron","text":"cron"},
  {"value":"clamav","text":"clamav"},
  {"value":"hestia","text":"hestia"},
  {"value":"rspamd","text":"rspamd"}
]
```

Orden mostrado:

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

## 7. Estrategia de consultas

Los servidores Linux convencionales utilizan:

```text
job="lab-servers"
```

`mqh01` utiliza:

```text
job="aws-mqh01"
```

Las consultas de los paneles combinan ambas fuentes mediante PromQL.

Conceptualmente:

```text
Node Exporter
     │
     ├──── OR ────► resultado
     │
YACE / CloudWatch
```

De esta forma Grafana recibe una única serie para el servidor seleccionado.

---

# MÉTRICAS

---

## 8. CPU

### Consulta

```promql
(
  100 - (
    avg by (server) (
      rate(node_cpu_seconds_total{
        mode="idle",
        server="$instance",
        job="lab-servers"
      }[5m])
    ) * 100
  )
)
or
(
  aws_ec2_cpuutilization_average{
    server="$instance",
    job="aws-mqh01"
  }
)
```

### Funcionamiento

Para:

```text
lab01
python
security
mail01
```

se calcula el porcentaje de CPU a partir de Node Exporter.

Para:

```text
mqh01
```

se utiliza:

```text
aws_ec2_cpuutilization_average
```

procedente de AWS CloudWatch mediante YACE.

Unidad recomendada:

```text
Percent (0-100)
```

---

## 9. RAM utilizada

### Consulta

```promql
(
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
)
or
(
  aws_cwagent_mem_used_percent_average{
    server="$instance",
    job="aws-mqh01",
    name="mqh01-memory"
  }
)
```

Para servidores Node Exporter se calcula:

```text
1 - RAM disponible / RAM total
```

Para `mqh01` se utiliza:

```text
aws_cwagent_mem_used_percent_average
```

Unidad:

```text
Percent (0-100)
```

---

## 10. Disco utilizado

Se monitoriza el filesystem raíz:

```text
/
```

### Consulta

```promql
(
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
)
or
(
  aws_cwagent_disk_used_percent_average{
    server="$instance",
    job="aws-mqh01",
    name="mqh01-disk",
    dimension_path="/"
  }
)
```

Unidad:

```text
Percent (0-100)
```

---

## 11. Load Average — 1 minuto

```promql
node_load1{
  server="$instance",
  job="lab-servers"
}
or
aws_cwagent_load_average_1m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

---

## 12. Load Average — 5 minutos

```promql
node_load5{
  server="$instance",
  job="lab-servers"
}
or
aws_cwagent_load_average_5m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

---

## 13. Load Average — 15 minutos

```promql
node_load15{
  server="$instance",
  job="lab-servers"
}
or
aws_cwagent_load_average_15m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

Las métricas de Load Average de `mqh01` son generadas mediante StatsD y
CloudWatch Agent.

---

## 14. Uptime

### Consulta

```promql
(
  node_time_seconds{
    server="$instance",
    job="lab-servers"
  }
  -
  node_boot_time_seconds{
    server="$instance",
    job="lab-servers"
  }
)
or
(
  aws_cwagent_uptime_seconds_average{
    server="$instance",
    job="aws-mqh01",
    name="mqh01-system"
  }
)
```

Para Node Exporter:

```text
hora actual - hora de arranque
```

Para `mqh01`:

```text
uptime_seconds
```

Unidad de Grafana:

```text
Duration → seconds
```

---

## 15. Estado del servidor

El panel muestra:

```text
ON
OFF
```

### Consulta

```promql
(
  up{
    server="$instance",
    job="lab-servers"
  }
)
or
(
  1 -
  clamp_max(
    aws_ec2_status_check_failed_average{
      server="$instance",
      job="aws-mqh01"
    },
    1
  )
)
```

### Value mappings

```text
1 → ON
0 → OFF
```

Colores:

```text
ON  → verde
OFF → rojo
```

Para Node Exporter se utiliza directamente `up`.

Para AWS se invierte `StatusCheckFailed`:

```text
StatusCheckFailed = 0 → ON
StatusCheckFailed = 1 → OFF
```

---

# RED

---

## 16. Tráfico de entrada

### Consulta

```promql
(
  sum(
    rate(node_network_receive_bytes_total{
      server="$instance",
      job="lab-servers",
      device!="lo"
    }[1m])
  )
)
or
(
  aws_ec2_network_in_sum{
    server="$instance",
    job="aws-mqh01"
  } / 300
)
```

Para Node Exporter se suma el tráfico recibido por todas las interfaces salvo:

```text
lo
```

Para `mqh01`, CloudWatch proporciona `NetworkIn` como suma durante 300
segundos.

Por ello:

```text
NetworkIn / 300
```

produce una media aproximada en bytes por segundo.

---

## 17. Tráfico de salida

### Consulta

```promql
(
  sum(
    rate(node_network_transmit_bytes_total{
      server="$instance",
      job="lab-servers",
      device!="lo"
    }[1m])
  )
)
or
(
  aws_ec2_network_out_sum{
    server="$instance",
    job="aws-mqh01"
  } / 300
)
```

Para `mqh01`:

```text
NetworkOut / 300
```

Unidad recomendada:

```text
bytes/sec
```

---

# INFORMACIÓN DEL SERVIDOR

---

## 18. Panel Información

El dashboard contiene un panel de información que muestra:

```text
nombre del servidor
hostname
dirección / plataforma
sistema operativo
kernel
CPU total
RAM total
disco total
```

El panel utiliza varias consultas Prometheus.

---

## 19. Consulta A — Información del sistema

```promql
node_uname_info{
  server="$instance",
  job="lab-servers"
}
or
label_replace(
  label_replace(
    label_replace(
      label_replace(
        aws_ec2_status_check_failed_average{
          server="$instance",
          job="aws-mqh01"
        },
        "nodename",
        "masqhosting",
        "",
        ""
      ),
      "sysname",
      "Linux",
      "",
      ""
    ),
    "release",
    "6.12.107+deb13-cloud-amd64",
    "",
    ""
  ),
  "instance",
  "AWS EC2 · eu-south-2",
  "",
  ""
)
```

Para servidores Node Exporter se utiliza:

```text
node_uname_info
```

Para `mqh01` se añaden mediante PromQL los datos necesarios para mantener el
mismo formato del panel.

---

## 20. Consulta B — CPU total

```promql
(
  count(
    count by (cpu) (
      node_cpu_seconds_total{
        server="$instance",
        job="lab-servers"
      }
    )
  )
)
or
(
  aws_ec2_status_check_failed_average{
    server="$instance",
    job="aws-mqh01"
  } * 0 + 2
)
```

`mqh01` dispone actualmente de:

```text
2 CPU
```

---

## 21. Consulta C — RAM total

```promql
round(
  (
    node_memory_MemTotal_bytes{
      server="$instance",
      job="lab-servers"
    } / 1024 / 1024 / 1024
  )
  or
  (
    aws_cwagent_mem_total_average{
      server="$instance",
      job="aws-mqh01",
      name="mqh01-memory"
    } / 1024 / 1024 / 1024
  ),
  0.1
)
```

Resultado expresado en:

```text
GB
```

---

## 22. Consulta D — Disco total

```promql
round(
  (
    node_filesystem_size_bytes{
      server="$instance",
      job="lab-servers",
      mountpoint="/",
      fstype!~"tmpfs|overlay"
    } / 1024 / 1024 / 1024
  )
  or
  (
    aws_cwagent_disk_total_average{
      server="$instance",
      job="aws-mqh01",
      name="mqh01-disk",
      dimension_path="/"
    } / 1024 / 1024 / 1024
  ),
  0.1
)
```

Resultado expresado en:

```text
GB
```

---

## 23. HTML del panel Información

```html
<div style="padding:20px 30px; max-width:950px; margin:auto;">
  <div style="text-align:center; font-size:32px; font-weight:700; margin-bottom:25px;">
    {{data.[0].[0].server}}
  </div>

  <div style="display:flex; justify-content:center; gap:20px;">
    <div style="flex:1; max-width:400px; padding:18px 22px; border:1px solid rgba(255,255,255,0.12); border-radius:8px; background:rgba(255,255,255,0.025);">
      <div style="font-size:22px; font-weight:700; margin-bottom:10px;">HOST</div>
      <div style="font-size:18px;">{{data.[0].[0].nodename}}</div>
      <div style="font-size:16px; opacity:0.70; margin-top:5px;">{{data.[0].[0].instance}}</div>
    </div>

    <div style="flex:1; max-width:400px; padding:18px 22px; border:1px solid rgba(255,255,255,0.12); border-radius:8px; background:rgba(255,255,255,0.025);">
      <div style="font-size:22px; font-weight:700; margin-bottom:10px;">SISTEMA</div>
      <div style="font-size:18px;">{{data.[0].[0].sysname}}</div>
      <div style="font-size:16px; opacity:0.70; margin-top:5px;">Kernel: {{data.[0].[0].release}}</div>
    </div>
  </div>

  <div style="max-width:820px; margin:20px auto 0 auto; padding:16px 20px; border:1px solid rgba(255,255,255,0.12); border-radius:8px; background:rgba(255,255,255,0.025); display:flex; justify-content:space-around; font-size:21px;">
    <div><strong>CPU:</strong> {{data.[1].[0].Value}} CPU</div>
    <div><strong>RAM:</strong> {{data.[2].[0].Value}} GB</div>
    <div><strong>Disco:</strong> {{data.[3].[0].Value}} GB</div>
  </div>
</div>
```

---

## 24. Información actual de mqh01

El panel muestra actualmente:

```text
mqh01

HOST
masqhosting
AWS EC2 · eu-south-2

SISTEMA
Linux
Kernel: 6.12.107+deb13-cloud-amd64

CPU: 2 CPU
RAM: 1.9 GB
Disco: 29.3 GB
```

---

# LOGS

---

## 25. Panel de logs

El mismo panel permite trabajar con diferentes tipos de logs mediante:

```text
$instance
$log_type
```

La arquitectura utiliza:

```text
Loki
CloudWatch Logs
```

---

## 26. Consulta Loki

Consulta:

```logql
{server="$instance", log_type="$log_type"}
```

Esto permite filtrar simultáneamente por:

```text
servidor
tipo de log
```

---

## 27. Logs de mqh01

`mqh01` utiliza AWS CloudWatch Logs.

Datasource:

```text
cloudwatch-1
```

Región:

```text
eu-south-2
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

## 28. Consulta CloudWatch Logs Insights

```text
fields @timestamp, @message
| filter @log like /$instance/
| filter @log like /$log_type/
| sort @timestamp desc
| limit 100
```

La consulta contiene dos filtros importantes.

### Servidor

```text
| filter @log like /$instance/
```

Evita que los registros de `mqh01` aparezcan cuando se selecciona otro
servidor.

### Tipo de log

```text
| filter @log like /$log_type/
```

Selecciona:

```text
ssh
apache
fail2ban
errors
kernel
```

según corresponda.

---

## 29. Logs disponibles en mqh01

Actualmente se contemplan:

| Tipo | Origen |
|---|---|
| `apache` | WordPress / Apache |
| `ssh` | OpenSSH |
| `fail2ban` | Fail2ban |
| `errors` | errores del sistema |
| `kernel` | kernel |

No se pretende recopilar indiscriminadamente todos los logs del servidor.

---

# DECISIONES TÉCNICAS

---

## 30. Un único dashboard

Se descartó mantener paneles específicos para cada plataforma.

No se utiliza:

```text
panel CPU Linux
panel CPU AWS
panel RAM Linux
panel RAM AWS
```

Se utiliza:

```text
CPU
RAM
Disco
Load
Uptime
Estado
Red
```

y cada consulta decide automáticamente qué métrica utilizar.

---

## 31. No duplicar paneles

El dashboard utiliza layout personalizado.

Las reglas automáticas de Show/Hide no proporcionaban el comportamiento
deseado en este diseño.

Por ello se descartó utilizar paneles duplicados ocultados mediante reglas.

La solución definitiva se basa en PromQL combinado mediante:

```text
or
```

---

## 32. No utilizar CloudWatch directamente para las métricas del dashboard

Aunque Grafana dispone de datasource CloudWatch, se decidió normalizar las
métricas de `mqh01` mediante:

```text
CloudWatch
    ↓
YACE
    ↓
Prometheus
    ↓
Grafana
```

Esto permite que los paneles de métricas utilicen una única datasource:

```text
Prometheus
```

CloudWatch continúa utilizándose directamente para:

```text
logs de mqh01
```

---

## 33. NetworkIn y NetworkOut

Una diferencia importante entre Node Exporter y CloudWatch es la forma de
representar el tráfico.

Node Exporter permite calcular directamente una tasa:

```promql
rate(...[1m])
```

CloudWatch entrega:

```text
Sum durante 300 segundos
```

Por tanto, para `mqh01`:

```text
bytes/segundo = Sum / 300
```

No eliminar esta división mientras YACE mantenga:

```text
period: 300
statistics:
  - Sum
```

---

## 34. Dimensión metric_type

Las métricas personalizadas StatsD de `mqh01` utilizan:

```text
InstanceId
metric_type
```

YACE inicialmente no descubría correctamente las métricas de Load Average y
uptime porque faltaba:

```text
metric_type
```

en los requisitos de dimensiones.

Este detalle debe conservarse en futuras modificaciones de YACE.

---

# OPERACIÓN

---

## 35. Flujo final

```text
                    ┌──────────────────────┐
                    │       Grafana        │
                    │      Fedora44        │
                    └──────────▲───────────┘
                               │
                  ┌────────────┴────────────┐
                  │                         │
             Prometheus                  Logs
                  ▲                         ▲
          ┌───────┴───────┐         ┌──────┴──────┐
          │               │         │             │
   Node Exporter         YACE      Loki      CloudWatch
          ▲               ▲         ▲             ▲
          │               │         │             │
 lab01/python/            │      servidores      mqh01
 security/mail01          │
                          │
                     CloudWatch
                          ▲
                          │
                        mqh01
```

---

## 36. Estado actual

```text
Dashboard                   OPERATIVO

Variables
  Servidor                   OPERATIVA
  Log                        OPERATIVA

Métricas
  CPU                        OPERATIVA
  RAM                        OPERATIVA
  Disco                      OPERATIVA
  Load 1m                    OPERATIVA
  Load 5m                    OPERATIVA
  Load 15m                   OPERATIVA
  Uptime                     OPERATIVA
  Estado                     OPERATIVA
  Red entrada                OPERATIVA
  Red salida                 OPERATIVA

Información
  lab01                      OPERATIVA
  python                     OPERATIVA
  security                   OPERATIVA
  mail01                     OPERATIVA
  mqh01                      OPERATIVA

Logs
  Loki                       OPERATIVO
  CloudWatch Logs mqh01      OPERATIVO
```

---

## 37. Reglas para futuras modificaciones

1. Mantener un único dashboard.
2. No crear paneles duplicados por servidor.
3. Mantener únicamente `Servidor` y `Log` como selectores visibles salvo que
   exista una necesidad real.
4. Mantener `$instance` como variable principal del servidor.
5. Mantener `$log_type` como variable de logs.
6. Mantener Prometheus como datasource común para las métricas.
7. Mantener CloudWatch directo únicamente donde sea necesario.
8. Si cambia el periodo de `NetworkIn` o `NetworkOut`, revisar la división
   entre 300.
9. Si cambian las dimensiones StatsD, revisar YACE.
10. No modificar el dashboard de producción sin conservar una forma de
    recuperación o exportación del JSON cuando los cambios sean importantes.

---

## 38. Documentación relacionada

```text
estado-actual.md
arquitectura.md
prometheus.md
loki.md
servidores.md
seguridad.md
recuperacion.md
```

`estado-actual.md` representa la referencia rápida del estado operativo.

`grafana.md` contiene la configuración lógica definitiva del dashboard.
