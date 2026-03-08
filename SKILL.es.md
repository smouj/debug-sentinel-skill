---
nombre: Debug Sentinel
slug: debug-sentinel
descripción: Monitoreo de seguridad en tiempo real y detección de anomalías para aplicaciones OpenClaw con análisis de amenazas en runtime
versión: 1.2.0
autor: Equipo de Seguridad (OpenClaw)
etiquetas: monitoring, threat-detection, runtime-analysis, security
requiere: [python3 >=3.8, psutil, scapy, elasticsearch==7.17.0, kibana, docker, libpcap-dev]
conflictos: [legacy-monitor, old-audit]
prioridad: 90
variables_entorno:
  - DEBUG_SENTINEL_CONFIG: Ruta al archivo YAML de configuración (default: ~/.config/debug-sentinel/config.yaml)
  - SENTINEL_LOG_LEVEL: DEBUG, INFO, WARNING, ERROR (default: INFO)
  - SENTINEL_DB_PATH: Cadena de conexión a Elasticsearch (default: http://localhost:9200)
  - SENTINEL_ALERT_WEBHOOK: Webhook de Slack/Discord para alertas críticas
---

# Debug Sentinel

Monitoreo de seguridad en runtime para aplicaciones OpenClaw con detección de amenazas en tiempo real, análisis de anomalías y registro forense.

## Propósito

**Casos de uso reales:**
- Detectar ataques de inyección SQL en endpoints de API de OpenClaw monitoreando patrones de consulta y firmas de payloads
- Identificar intentos de escalación de privilegios mediante árboles de procesos sospechosos y capacidades
- Monitorear conexiones de red anómalas desde aplicaciones containerizadas (Docker/K8s)
- Alertar sobre anomalías a nivel de kernel como syscalls inesperados desde procesos de aplicación
- Generación automatizada de líneas de tiempo forenses para respuesta a incidentes
- Registro de cumplimiento para requisitos SOC2/PCI con trails de auditoría inmutables
- Detección de anomalías de rendimiento correlacionando eventos de seguridad con picos de recursos

## Alcance

**Comandos exactos disponibles:**

### Comandos Principales
```
debug-sentinel start [--config PATH] [--daemon] [--pid-file PATH]
debug-sentinel stop [--graceful] [--timeout SECONDS]
debug-sentinel status [--json] [--verbose] [--metrics]
debug-sentinel restart [--config PATH]
```

### Comandos de Análisis
```
debug-sentinel analyze [--since TIMESPEC] [--until TIMESPEC] [--app APP_NAME] [--severity LEVEL]
                        [--format {json,table,csv}] [--output FILE]
debug-sentinel alert list [--active] [--resolved] [--severity {CRITICAL,HIGH,MEDIUM,LOW}]
debug-sentinel alert ack ALERT_ID [--comment TEXT]
debug-sentinel alert resolve ALERT_ID [--auto]
debug-sentinel threat-hunt [--ioc IOC_VALUE] [--type {ip,domain,hash,user}] [--hours N]
```

### Comandos de Configuración
```
debug-sentinel rules list [--enabled] [--category CATEGORY]
debug-sentinel rules add RULE_FILE [--category CATEGORY] [--priority 1-10]
debug-sentinel rules remove RULE_ID [--force]
debug-sentinel rules test RULE_FILE [--sample-traffic PATH]
debug-sentinel config show [--section SECTION]
debug-sentinel config set SECTION.KEY VALUE [--type {string,int,bool,list}]
debug-sentinel config validate
```

### Dashboard e Informes
```
debug-sentinel dashboard [--host HOST] [--port PORT] [--ssl] [--auth USER:PASS]
debug-sentinel report generate [--type {daily,weekly,incident}] [--since DATE] [--output PATH]
debug-sentinel export evidence ALERT_ID [--format {json,zip}] [--output PATH]
```

### Mantenimiento
```
debug-sentinel db cleanup [--older-than DAYS] [--dry-run]
debug-sentinel index rotate [--daily|--weekly] [--keep N]
debug-sentinel healthcheck [--verbose]
debug-sentinel self-test [--all] [--category {network,syscall,file,process}]
```

## Proceso de Trabajo Detallado

**Flujo operativo real:**

### 1. Configuración Inicial
```bash
# Crear directorio de configuración
mkdir -p ~/.config/debug-sentinel/

# Generar configuración de producción por defecto
debug-sentinel config generate --template production > ~/.config/debug-sentinel/config.yaml

# Editar secciones críticas:
# - elasticsearch.hosts: ["http://es-cluster:9200"]
# - monitoring.apps: ["rpg-claw-api", "rpg-claw-worker"]
# - rules.directories: ["/opt/debug-sentinel/rules/enabled", "/opt/debug-sentinel/rules/custom"]
# - alerting.webhooks: ["https://hooks.slack.com/services/..."]
# - network.interfaces: ["eth0", "docker0"]

# Validar sintaxis de configuración
debug-sentinel config validate
```

### 2. Despliegue de Reglas
```bash
# Descargar reglas de detección más recientes del repositorio de seguridad OpenClaw
git clone https://github.com/openclaw/security-rules /opt/debug-sentinel/rules

# Habilitar solo reglas probadas en producción
cd /opt/debug-sentinel/rules/enabled
ln -sf ../available/sql_injection.yaml .
ln -sf ../available/path_traversal.yaml .
ln -sf ../available/privilege_escalation.yaml .

# Probar sintaxis de reglas sin desplegar
debug-sentinel rules test /opt/debug-sentinel/rules/custom/my_rule.yaml

# Agregar regla personalizada para patrón de ataque conocido
debug-sentinel rules add /opt/debug-claw/rules/custom/rpg_claw_specific.yaml \
  --category "application" \
  --priority 8

# Verificar cantidad de reglas
debug-sentinel rules list | wc -l  # Debería mostrar 150+ reglas
```

### 3. Despliegue de Demonio
```bash
# Iniciar como servicio systemd (recomendado)
sudo cp debug-sentinel.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable debug-sentinel
sudo systemctl start debug-sentinel

# Verificar estado del servicio con métricas
debug-sentinel status --verbose --metrics

# Salida esperada:
# State: RUNNING (PID 2847)
# Uptime: 2h 15m
# Events processed: 1,247,893
# Rules loaded: 167
# Alerts (24h): CRITICAL:3, HIGH:12, MEDIUM:47, LOW:156
# Memory: 312MB, CPU avg: 2.4%
# Last heartbeat: 2025-03-08T10:45:22Z
```

### 4. Monitoreo Continuo
```bash
# Tail en tiempo real de eventos de seguridad (como tail -f)
debug-sentinel analyze --since 5m --format json | jq '.[] | select(.severity=="CRITICAL")'

# Verificar alertas activas que requieren atención
debug-sentinel alert list --active --severity HIGH

# Salida de ejemplo:
# ID: ALT-7f3a9b2c  Severity: HIGH  Rule: SQL_INJECTION_DETECTED
# App: rpg-claw-api  Source: 192.168.1.105  Time: 10:42:15
# Evidence: "SELECT * FROM users WHERE id='1' OR '1'='1'"
# [ACK] [RESOLVE]

# Threat hunt para IoC específico en 30 días
debug-sentinel threat-hunt --ioc "35.200.220.123" --type ip --hours 720

# Generar línea de tiempo forense para incidente ALT-7f3a9b2c
debug-sentinel export evidence ALT-7f3a9b2c --format json --output /tmp/incident_7f3a.json
```

### 5. Respuesta a Incidentes
```bash
# Cuando se activa una alerta CRÍTICA:
# 1. Verificación inmediata
debug-sentinel alert list --active --severity CRITICAL

# 2. Exportar toda evidencia con contexto
debug-sentinel export evidence ALT-xxxx --format zip --output /tmp/forensic_$(date +%s).zip

# 3. Analizar árbol de procesos al momento del incidente
debug-sentinel analyze --since "2025-03-08T10:40" --until "2025-03-08T10:45" \
  --format json | jq '.[] | select(.event_type=="process_exec")'

# 4. Generar informe de incidente
debug-sentinel report generate --type incident --since "2025-03-08" --output /tmp/report.pdf

# 5. Reconocer y rastrear
debug-sentinel alert ack ALT-xxxx --comment "Equipo IR investigando, aislando host afectado"
```

## Reglas de Oro

1. **Prueba de Reglas Obligatoria**: Todas las reglas personalizadas DEBEN pasar `debug-sentinel rules test` con 0 falsos positivos en 1000+ solicitudes legítimas de muestra antes del despliegue en producción

2. **Calibración de Umbrales**: Nunca usar umbrales de detección por defecto en producción. Ejecutar en modo `LEARN` durante 7 días para establecer baseline de tráfico legítimo, luego cambiar a `ENFORCE` con umbrales ajustados

3. **Prevención de Fatiga de Alertas**: Configurar filtros de severidad para que alertas HIGH+ activen pagerduty, MEDIUM vayan a Slack #security, LOW a digest diario. Requerida revisión semanal de falsos positivos

4. **Registro Inmutable**: Los índices de Elasticsearch deben tener `index.lifecycle.rollover_alias: debug-sentinel-logs` y `number_of_shards: 1`. Nunca eliminar logs manualmente; usar `debug-sentinel index rotate`

5. **Alcance de Captura de Red**: Solo monitorear interfaces explícitamente listadas en config. Nunca habilitar modo promiscuo en producción sin aprobación del equipo de red

6. **Límite de Automatización de Respuesta**: Auto-bloqueo (integración fail2ban) SÓLO para patrones de ataque verificados con tasa de falsos positivos <0.1% durante 30 días

7. **Cumplimiento de Privacidad**: Enmascarar PII en logs usando reglas de redacción antes del almacenamiento. Habilitado por defecto: `redaction.enabled: true` con patrones estándar para emails, SSNs, tarjetas de crédito

8. **Presupuesto de Rendimiento**: El demonio sentinel no debe exceder 5% CPU o 500MB RAM en hosts de producción. Muestrear apropiadamente: `sampling.rate: 0.1` para aplicaciones de alto tráfico (>10k RPS)

## Ejemplos

### Ejemplo 1: Detectar Inyección SQL en API RPGClaw
```bash
# Entrada de usuario: Atacante envía payload malicioso
curl -X POST https://rpg-claw.example.com/api/user/search \
  -H "Authorization: Bearer <valid-token>" \
  -d '{"query": "1' OR '1'='1"}'

# Detección de Debug Sentinel (15ms después):
$ debug-sentinel alert list --active --severity HIGH
ID: ALT-a1b2c3d4  Severity: HIGH  Rule: SQLI_PATTERN_COMMON
Time: 2025-03-08 10:42:15 UTC
App: rpg-claw-api
Source IP: 192.168.1.105 (attacker.example.com)
User: authenticated_user_45
Evidence:
  Request ID: req_7f8g9h0j
  Payload: {"query": "1' OR '1'='1'"}
  Matched pattern: OR '1'='1'
  Stack trace: /app/api/search.py:127
  Process tree: nginx(2847)#python3(2851)#handler(2852)

# Respuesta automatizada si está configurado:
# - IP bloqueada via fail2ban (sentinel.jail.conf)
# - Sesión de usuario revocada (callback API)
# - Alerta en Slack a #secops
```

### Ejemplo 2: Detección de Escalación de Privilegios de Proceso
```bash
# Evento: Proceso de aplicación intenta elevar privilegios
$ debug-sentinel analyze --since 1h --format json | jq '.[] | select(.event=="setuid")'
{
  "timestamp": "2025-03-08T09:15:33Z",
  "event_type": "setuid",
  "severity": "CRITICAL",
  "app": "rpg-claw-worker",
  "pid": 3456,
  "uid_before": 1001,
  "uid_after": 0,
  "process": "/usr/bin/python3 /opt/rpgclaw/worker/tasks.py",
  "parent_process": "celery(2890)",
  "rule_id": "PRIV_ESC_SETUID_0",
  "container_id": "a1b2c3d4e5f6"
}

# Respuesta: Matar proceso inmediatamente y aislar container
docker stop rpg-claw-worker-01
nsenter -t 3456 -m -u -i -n -p kill -9 3456
```

### Ejemplo 3: Anomalía de Red - Exfiltración de Datos
```bash
# Detectar conexiones salientes anómalas desde container de app
$ debug-sentinel alert list --severity HIGH --since 6h
ID: ALT-z9y8x7w6  Severity: HIGH  Rule: UNUSUAL_OUTBOUND
App: rpg-claw-api
Time: 2025-03-08 08:30:00 UTC
Evidence:
  External IP: 185.220.101.45 (Russia - región no aprobada)
  Port: 4444/TCP
  Bytes sent: 2.4MB en 5 minutos
  Process: python3(zip_export.py)
  Historical baseline: <50KB/day promedio para este proceso
  GDPR impact: HIGH (violación de ubicación de datos de clientes)

# Verificar base de datos geo IP cargada
debug-sentinel config show geoip
# Salida: geoip.enabled: true, database: /var/lib/geoip/GeoLite2-City.mmdb
```

### Ejemplo 4: Threat Hunt en Runtime
```bash
# Después de detectar usuario sospechoso "admin_temp" creado a las 04:00
$ debug-sentinel threat-hunt --ioc "admin_temp" --type user --hours 24

HUNT RESULTS for IoC: admin_temp (user)
========================================
Found 47 events across 3 hosts:

1. 2025-03-08 04:02:11 - Host: rpg-db-01
   Event: USER_ADDED (auditd)
   Details: uid=1002, gid=1002, home=/home/admin_temp
   Session: ssh from 10.0.5.23 (jumpbox)

2. 2025-03-08 04:03:45 - Host: rpg-claw-api-02
   Event: SUDO_EXEC
   Details: admin_temp -> root via ALL NOPASSWD
   Command: systemctl restart nginx

3. 2025-03-08 04:05:22 - Host: rpg-claw-api-02
   Event: FILE_ACCESS /etc/shadow
   Severity: CRITICAL

RECOMMENDACIÓN: Aislamiento inmediato de rpg-claw-api-02, rotación de credenciales para todas las claves SSH de admin_temp
```

### Ejemplo 5: Monitoreo en Entorno Kubernetes
```bash
# Desplegar como DaemonSet en cluster K8s
kubectl apply -f debug-sentinel-daemonset.yaml

# Verificar estado de pods
kubectl get pods -n security -l app=debug-sentinel

# Consultar alertas desde namespace específico
debug-sentinel alert list --filter 'kubernetes.namespace=="rpg-claw-prod"' \
  --since 1h --format table

# Extraer eventos de seguridad de contenedores
debug-sentinel analyze --app "rpg-claw-frontend" \
  --since 30m --format json | jq -r '.container_id' | sort -u
```

## Comandos de Rollback

**Rollback de emergencia inmediato:**
```bash
# Detener monitoreo completamente (usar solo para compromiso activo)
debug-sentinel stop --graceful --timeout 10  # Drenado graceful primero
# Si se cuelga:
pkill -f debug-sentinel
# Verificar detenido:
debug-sentinel status  # Debería mostrar "STOPPED"

# Eliminar todas las reglas iptables inyectadas por sentinel
debug-sentinel iptables flush  # Solo si integrado con fail2ban
# O manualmente:
iptables -D INPUT -p tcp --dport 22 -j DROP  # Eliminar IPs bloqueadas
```

**Rollback de configuración:**
```bash
# Restaurar configuración anterior desde auto-backup (creado en cada cambio de config)
cp ~/.config/debug-sentinel/config.backup.{timestamp}.yaml \
   ~/.config/debug-sentinel/config.yaml

# O revertir a versión conocida-good desde git
git -C /opt/debug-sentinel/rules checkout HEAD -- enabled/
debug-sentinel rules reload

# Reiniciar con baseline anterior
debug-sentinel restart --config /opt/debug-sentinel/configs/prod-baseline.yaml
```

**Rollback de limpieza de base de datos:**
```bash
# Si ejecutaste debug-sentinel db cleanup por error
# La próxima vez verificar qué se eliminaría primero:
debug-sentinel db cleanup --older-than 90 --dry-run  # Muestra conteo
# Si eliminaste índices y necesitas restaurar:
# 1. Encontrar snapshot previo a eliminación
curl -X GET "localhost:9200/_snapshot/debug-sentinel-backup/snap*/_all" | jq .
# 2. Restaurar índice específico
curl -X POST "localhost:9200/_snapshot/debug-sentinel-backup/snap_20250305/_restore" \
  -H 'Content-Type: application/json' -d '{"indices": "logs-2025.03.01"}'
```

**Deshabilitar detección específica temporalmente:**
```bash
# Suspender regla sin eliminar (para investigación de falso positivo)
debug-sentinel rules disable RULE_ID --comment "Falso positivo en procesamiento de pagos"
# Re-habilitar después de corrección:
debug-sentinel rules enable RULE_ID
```

**Desinstalación completa con limpieza de artefactos:**
```bash
# Detener servicio
sudo systemctl stop debug-sentinel
sudo systemctl disable debug-sentinel

# Eliminar configs (¡conservar backups primero!)
cp -r ~/.config/debug-sentinel ~/.config/debug-sentinel.bak
rm -rf ~/.config/debug-sentinel

# Eliminar reglas y base de datos (¡PRECAUCIÓN: borra todos los datos históricos!)
# ¡Conservar si necesitas archivos de cumplimiento!
# rm -rf /opt/debug-sentinel/
# curl -X DELETE "localhost:9200/debug-sentinel-*"  # Wipe Elasticsearch
```

**Checklist de rollback después de incidente:**
1. `debug-sentinel stop` - aislar monitoreo para prevenir más alertas
2. Revisar `/var/log/debug-sentinel/audit.log` para acciones propias de sentinel
3. Restaurar config desde backup timestamped: `config.backup.YYYYMMDD_HHMM.yaml`
4. `debug-sentinel start --config restored_config.yaml`
5. Validar: `debug-sentinel healthcheck --verbose`
6. Verificar manualmente que detecciones críticas aún funcionales: `debug-sentinel self-test --category network`
7. Documentar causa raíz en ~/.config/debug-sentinel/incidents/INC-YYYY-NNN.md

**Escenarios de rollback parcial:**

*Inundación de falsos positivos:*
```bash
# Deshabilitar regla específica causando problemas
debug-sentinel rules disable SQLI_PATTERN_COMMON --comment "Falso positivo en cliente legacy"
# O ajustar umbral
debug-sentinel config set detection.rules.sql_injection.threshold 0.95
debug-sentinel restart
```

*Degradación de rendimiento:*
```bash
# Reducir tasa de muestreo
debug-sentinel config set sampling.rate 0.05  # Desde 0.1
# Deshabilitar categorías de reglas no críticas
debug-sentinel config set rules.disabled_categories '["performance","compliance"]'
debug-sentinel restart
```

*Falla de canal de alerta:*
```bash
# Deshabilitar webhook temporalmente
debug-sentinel config set alerting.webhooks "[]"
# O enrutar a canal de backup
debug-sentinel config set alerting.webhooks '["https://backup-hooks..."]'
debug-sentinel restart
```
```