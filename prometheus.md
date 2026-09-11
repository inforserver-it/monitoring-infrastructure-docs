# Prometheus — Monitorización Centralizada

**Nodo central:** Fedora44  
**Última actualización:** 11 de septiembre de 2026

---

## 1. Objetivo

Prometheus es el sistema central de recopilación y consulta de métricas de la
plataforma de monitorización.

Actualmente recibe métricas desde dos arquitecturas:

```text
Node Exporter
AWS CloudWatch mediante YACE
```

El objetivo es proporcionar a Grafana una fuente común de métricas,
independientemente del origen real de los datos.

---

## 2. Arquitectura

```text
lab01 ──────┐
python ─────┤
security ───┼── Node Exporter ──┐
mail01 ─────┘                    │
                                │
                                ▼
                           Prometheus
                                │
                                ▼
                             Grafana
                                ▲
                                │
mqh01 ── CloudWatch ── YACE ────┘
```

Esto permite que Grafana utilice Prometheus para todos los paneles de métricas.

---

## 3. Ubicación

Stack:

```text
/datos/stacks/prometheus/
```

Configuración principal:

```text
/datos/stacks/prometheus/prometheus.yml
```

Docker Compose:

```text
/datos/stacks/prometheus/compose.yaml
```

---

## 4. Contenedor

Imagen utilizada:

```text
prom/prometheus:v3.14.0
```

Puerto publicado en Fedora44:

```text
9091
```

Puerto interno:

```text
9090
```

Conceptualmente:

```text
Fedora44:9091
      ↓
Prometheus:9090
```

---

## 5. Red Docker

Prometheus participa en la red Docker:

```text
monitoring
```

Esta red permite comunicación mediante nombres DNS internos con otros
componentes.

Ejemplo:

```text
yace:5000
```

No debe dependerse permanentemente de la IP interna de un contenedor Docker,
ya que puede cambiar al recrearlo.

---

# NODE EXPORTER

---

## 6. Job lab-servers

Los servidores monitorizados mediante Node Exporter utilizan:

```text
job_name: lab-servers
```

Actualmente:

```text
lab01
python
security
mail01
```

Cada servidor tiene además una etiqueta:

```text
server
```

---

## 7. Targets

Configuración lógica actual:

```yaml
scrape_configs:

  - job_name: "lab-servers"

    static_configs:

      - targets:
          - "192.168.1.101:9100"
        labels:
          server: "lab01"

      - targets:
          - "192.168.1.103:9100"
        labels:
          server: "security"

      - targets:
          - "192.168.1.104:9100"
        labels:
          server: "python"

      - targets:
          - "5.189.147.4:9100"
        labels:
          server: "mail01"
```

La etiqueta `server` es fundamental porque Grafana utiliza:

```text
$instance
```

para seleccionar el servidor.

---

## 8. Servidores Node Exporter

| Servidor | Dirección | Exporter | Job |
|---|---|---|---|
| lab01 | 192.168.1.101:9100 | Node Exporter | lab-servers |
| security | 192.168.1.103:9100 | Node Exporter | lab-servers |
| python | 192.168.1.104:9100 | Node Exporter | lab-servers |
| mail01 | 5.189.147.4:9100 | Node Exporter | lab-servers |

---

## 9. Métricas principales Node Exporter

Entre las métricas utilizadas por Grafana se encuentran:

```text
node_cpu_seconds_total

node_memory_MemAvailable_bytes
node_memory_MemTotal_bytes

node_filesystem_avail_bytes
node_filesystem_size_bytes

node_load1
node_load5
node_load15

node_time_seconds
node_boot_time_seconds

node_network_receive_bytes_total
node_network_transmit_bytes_total

node_uname_info

up
```

---

# AWS / YACE

---

## 10. mqh01

`mqh01` no utiliza Node Exporter.

Es una instancia AWS EC2 de producción y sus métricas siguen este flujo:

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

Esto evita tener que exponer:

```text
TCP 9100
```

desde la instancia EC2.

---

## 11. Job aws-mqh01

Prometheus consulta YACE mediante:

```text
job_name: aws-mqh01
```

Target:

```text
yace:5000
```

Intervalo:

```text
60 segundos
```

Etiqueta:

```text
server="mqh01"
```

Configuración lógica:

```yaml
- job_name: "aws-mqh01"

  scrape_interval: 60s

  static_configs:
    - targets:
        - "yace:5000"
      labels:
        server: "mqh01"
```

---

## 12. Estado de YACE

El estado normal esperado es:

```promql
up{
  job="aws-mqh01"
}
```

Resultado:

```text
1
```

Interpretación:

```text
1 → Prometheus puede consultar YACE
0 → problema de scrape o conectividad
```

---

## 13. YACE

YACE se ejecuta también en Fedora44.

Configuración:

```text
/datos/stacks/yace/config.yml
```

Endpoint interno:

```text
http://yace:5000
```

YACE consulta AWS CloudWatch y transforma las métricas en formato Prometheus.

---

## 14. AWS EC2

Datos relevantes de `mqh01`:

```text
Servidor:      mqh01
Hostname:      masqhosting
AWS Region:    eu-south-2
Instance ID:   i-0d4beab843b2d2f95
CPU:           2
RAM:           ~1.9 GB
Disco raíz:    ~29.3 GB
```

---

# MÉTRICAS AWS

---

## 15. CPU

CloudWatch:

```text
Namespace: AWS/EC2
Metric: CPUUtilization
Statistic: Average
Period: 300
```

Métrica expuesta por YACE:

```text
aws_ec2_cpuutilization_average
```

Grafana utiliza:

```promql
aws_ec2_cpuutilization_average{
  server="$instance",
  job="aws-mqh01"
}
```

---

## 16. Estado de EC2

CloudWatch:

```text
Namespace: AWS/EC2
Metric: StatusCheckFailed
Statistic: Average
Period: 60
```

Métrica Prometheus:

```text
aws_ec2_status_check_failed_average
```

CloudWatch utiliza:

```text
0 → comprobación correcta
1 → fallo
```

Grafana transforma este comportamiento para mostrar:

```text
1 → ON
0 → OFF
```

---

## 17. NetworkIn

CloudWatch:

```text
Namespace: AWS/EC2
Metric: NetworkIn
Statistic: Sum
Period: 300
```

YACE expone:

```text
aws_ec2_network_in_sum
```

Como el valor representa la suma de 300 segundos:

```text
bytes/segundo = NetworkIn / 300
```

Consulta utilizada:

```promql
aws_ec2_network_in_sum{
  server="$instance",
  job="aws-mqh01"
} / 300
```

---

## 18. NetworkOut

CloudWatch:

```text
Namespace: AWS/EC2
Metric: NetworkOut
Statistic: Sum
Period: 300
```

YACE expone:

```text
aws_ec2_network_out_sum
```

Consulta:

```promql
aws_ec2_network_out_sum{
  server="$instance",
  job="aws-mqh01"
} / 300
```

Si se modifica el periodo de YACE, esta división debe revisarse.

---

# CLOUDWATCH AGENT

---

## 19. Memoria

Namespace:

```text
mqh01-memory
```

Métricas:

```text
mem_used_percent
mem_total
```

Prometheus:

```text
aws_cwagent_mem_used_percent_average
aws_cwagent_mem_total_average
```

RAM utilizada:

```promql
aws_cwagent_mem_used_percent_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-memory"
}
```

RAM total:

```promql
aws_cwagent_mem_total_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-memory"
}
```

`mem_total` se expresa originalmente en bytes.

---

## 20. Disco

Namespace:

```text
mqh01-disk
```

Métricas:

```text
disk_used_percent
disk_total
```

Para el filesystem raíz:

```text
path="/"
```

Métricas Prometheus:

```text
aws_cwagent_disk_used_percent_average
aws_cwagent_disk_total_average
```

Disco utilizado:

```promql
aws_cwagent_disk_used_percent_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-disk",
  dimension_path="/"
}
```

Disco total:

```promql
aws_cwagent_disk_total_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-disk",
  dimension_path="/"
}
```

---

# MÉTRICAS PERSONALIZADAS

---

## 21. Sistema

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

Estas métricas se generan en `mqh01` y se envían mediante StatsD a CloudWatch
Agent.

---

## 22. Load Average 1m

Métrica Prometheus:

```text
aws_cwagent_load_average_1m_average
```

Consulta:

```promql
aws_cwagent_load_average_1m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

---

## 23. Load Average 5m

```promql
aws_cwagent_load_average_5m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

---

## 24. Load Average 15m

```promql
aws_cwagent_load_average_15m_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

---

## 25. Uptime

```promql
aws_cwagent_uptime_seconds_average{
  server="$instance",
  job="aws-mqh01",
  name="mqh01-system"
}
```

La métrica representa segundos desde el arranque.

---

# STATSD

---

## 26. Listener StatsD

CloudWatch Agent escucha en:

```text
127.0.0.1:8125/UDP
```

Configuración activa:

```text
[[inputs.statsd]]
interval = "60s"
parse_data_dog_tags = true
service_address = "127.0.0.1:8125"

[inputs.statsd.tags]
  "aws:AggregationInterval" = "60s"
```

---

## 27. Script de métricas

Script:

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

Ejecución mediante:

```text
/etc/cron.d/mqh01-system-metrics
```

Frecuencia:

```text
cada minuto
```

---

## 28. Dimensiones StatsD

Las métricas personalizadas tienen las dimensiones:

```text
InstanceId
metric_type
```

Valor observado:

```text
metric_type=gauge
```

Este punto es especialmente importante.

YACE debe incluir:

```text
InstanceId
metric_type
```

en los requisitos de dimensiones para las métricas de:

```text
mqh01-system
```

Si falta `metric_type`, las métricas pueden existir en CloudWatch pero no ser
descubiertas por YACE.

---

# CONFIGURACIÓN YACE

---

## 29. Métricas EC2

La configuración debe contemplar:

```yaml
- name: CPUUtilization
  statistics:
    - Average
  period: 300

- name: StatusCheckFailed
  statistics:
    - Average
  period: 60

- name: NetworkIn
  statistics:
    - Sum
  period: 300

- name: NetworkOut
  statistics:
    - Sum
  period: 300
```

---

## 30. Namespace mqh01-memory

Requisito principal de dimensión:

```text
InstanceId
```

Métricas:

```text
mem_used_percent
mem_total
```

---

## 31. Namespace mqh01-disk

Dimensiones necesarias:

```text
InstanceId
device
fstype
path
```

Métricas:

```text
disk_used_percent
disk_total
```

---

## 32. Namespace mqh01-system

Dimensiones necesarias:

```text
InstanceId
metric_type
```

Métricas:

```text
load_average_1m
load_average_5m
load_average_15m
uptime_seconds
```

---

# PROMQL COMBINADO

---

## 33. Principio

Grafana utiliza Prometheus como fuente común.

Las consultas combinan:

```text
Node Exporter
OR
YACE
```

Ejemplo conceptual:

```promql
node_metric{
  server="$instance",
  job="lab-servers"
}
or
aws_metric{
  server="$instance",
  job="aws-mqh01"
}
```

---

## 34. CPU combinada

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

---

## 35. RAM combinada

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

---

## 36. Disco combinado

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

---

## 37. Load 1m combinado

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

## 38. Load 5m combinado

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

## 39. Load 15m combinado

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

---

## 40. Uptime combinado

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

---

## 41. Estado combinado

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

---

## 42. Red entrada combinada

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

---

## 43. Red salida combinada

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

---

# OPERACIÓN

---

## 44. Reinicio de Prometheus

Desde Fedora44:

```bash
cd /datos/stacks/prometheus
sudo docker compose restart prometheus
```

Debe evitarse reiniciar componentes que no hayan sido modificados.

---

## 45. Reinicio de YACE

Cuando únicamente se modifica YACE:

```bash
cd /datos/stacks/yace
sudo docker compose restart yace
```

No es necesario reiniciar:

```text
Grafana
Loki
WordPress
Apache
Docker completo
```

por un cambio normal en YACE.

---

## 46. DNS interno después de reiniciar YACE

Después de recrear o reiniciar YACE puede existir un breve intervalo durante
el cual Prometheus todavía no resuelva:

```text
yace
```

La configuración permanente debe continuar utilizando:

```text
yace:5000
```

y no una IP Docker fija.

---

## 47. Seguridad

No deben mostrarse ni documentarse:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
tokens
contraseñas
claves privadas
```

Especialmente debe evitarse utilizar comandos de Docker Compose que impriman
la configuración expandida cuando existen secretos cargados mediante
variables de entorno.

Las credenciales AWS de YACE deben mantenerse fuera de esta documentación.

---

## 48. IAM

YACE utiliza credenciales con permisos de lectura para consultar CloudWatch.

La cuenta utilizada para esta integración dispone de permisos de lectura de
CloudWatch y recursos necesarios para el descubrimiento.

Un aviso relacionado con:

```text
iam:ListAccountAliases
```

puede aparecer sin impedir el funcionamiento de la monitorización.

No deben ampliarse permisos IAM únicamente para eliminar un warning si la
funcionalidad necesaria ya está operativa.

---

## 49. Decisiones importantes

### mqh01 no utiliza Node Exporter

La solución definitiva es:

```text
CloudWatch → YACE → Prometheus
```

### Métricas del dashboard

Todas las métricas del dashboard pasan por:

```text
Prometheus
```

incluidas las procedentes de AWS.

### Logs

Los logs de AWS no pasan por Prometheus.

Se consultan desde Grafana mediante CloudWatch Logs.

### Red AWS

Mientras YACE utilice:

```text
period: 300
statistics:
  - Sum
```

Grafana debe dividir:

```text
NetworkIn / 300
NetworkOut / 300
```

### StatsD

Las métricas de sistema requieren:

```text
InstanceId
metric_type
```

---

## 50. Estado actual

```text
Prometheus
  Servicio                     OPERATIVO
  Docker                       OPERATIVO
  Red monitoring               OPERATIVA

job lab-servers
  lab01                        UP
  python                       UP
  security                     UP
  mail01                       UP

job aws-mqh01
  YACE                         UP
  mqh01                        OPERATIVO

Métricas mqh01
  CPU                          OPERATIVA
  RAM                          OPERATIVA
  Disco                        OPERATIVA
  Load 1m                      OPERATIVA
  Load 5m                      OPERATIVA
  Load 15m                     OPERATIVA
  Uptime                       OPERATIVA
  Estado                       OPERATIVA
  NetworkIn                    OPERATIVA
  NetworkOut                   OPERATIVA
```

---

## 51. Regla de mantenimiento

Cuando se añada un nuevo servidor se debe decidir primero qué arquitectura
utilizará.

### Servidor accesible mediante Node Exporter

```text
Servidor → Node Exporter → Prometheus
```

### Servidor AWS donde no se desea exponer exporter

```text
Servidor → CloudWatch → YACE → Prometheus
```

Después se asignará siempre una etiqueta:

```text
server="<nombre>"
```

para mantener la integración con el dashboard de Grafana.

---

## 52. Documentación relacionada

```text
README.md
estado-actual.md
arquitectura.md
grafana.md
loki.md
node-exporter.md
servidores.md
seguridad.md
recuperacion.md
```

Este documento contiene la arquitectura y consultas definitivas de Prometheus.

`grafana.md` contiene la configuración de los paneles.

`estado-actual.md` representa la referencia rápida del estado operativo.
