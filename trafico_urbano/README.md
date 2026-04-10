# Gestión Inteligente de Tráfico Urbano
## Introducción a Sistemas Distribuidos — 2026-30

---

## 📋 Contenido del proyecto

```
trafico_urbano/
├── config/
│   └── config.json            ← Configuración global del sistema
├── common/
│   ├── config_loader.py       ← Carga y utilidades de configuración
│   └── logger.py              ← Logger estándar para todos los servicios
├── pc1/
│   ├── broker_zmq.py          ← Broker ZMQ (simple y multihilo)
│   ├── sensor_camara.py       ← Sensor cámara (EVENTO_LONGITUD_COLA)
│   ├── sensor_espira.py       ← Sensor espira inductiva (EVENTO_CONTEO)
│   └── sensor_gps.py          ← Sensor GPS (EVENTO_DENSIDAD_TRAFICO)
├── pc2/
│   ├── servicio_analitica.py  ← Analítica, detección de congestión, control
│   ├── servicio_semaforos.py  ← Control de estados de semáforos
│   └── base_datos_replica.py  ← BD réplica (SQLite, backup de PC3)
├── pc3/
│   ├── base_datos_principal.py← BD principal + heartbeat publisher
│   └── servicio_monitoreo.py  ← CLI de consultas y comandos directos
├── lanzar_pc1.sh              ← Script para iniciar todos los procesos de PC1
├── lanzar_pc2.sh              ← Script para iniciar todos los procesos de PC2
├── lanzar_pc3.sh              ← Script para iniciar todos los procesos de PC3
└── README.md                  ← Este archivo
```

---

## ⚙️ Prerequisitos

**Python 3.9+** y las siguientes librerías:

```bash
pip install pyzmq
```

SQLite3 viene incluido con Python (no requiere instalación adicional).

---

## 🔧 Configuración inicial

**PASO 1**: Editar `config/config.json` con las IPs reales de los computadores:

```json
"red": {
    "PC1_IP": "192.168.X.Y",   ← IP real del PC1
    "PC2_IP": "192.168.X.Z",   ← IP real del PC2
    "PC3_IP": "192.168.X.W"    ← IP real del PC3
}
```

**PASO 2**: Copiar el proyecto completo a los tres computadores en el **mismo
directorio**. Los scripts usan rutas relativas.

---

## 🚀 Orden de inicio (IMPORTANTE)

Los servicios deben iniciarse en el siguiente orden para evitar errores de
conexión:

```
1. PC3 (BD Principal + Heartbeat)
2. PC2 (BD Réplica + Semáforos + Analítica)
3. PC1 (Broker + Sensores)
```

---

## 🖥️ PC1 — Broker y Sensores

### Opción A: Script automático (recomendado)

```bash
# Inicia broker (simple) + sensores en todas las intersecciones
bash lanzar_pc1.sh

# Inicia broker en modo multihilo (para el experimento de rendimiento)
bash lanzar_pc1.sh --multihilo

# Solo un subconjunto de intersecciones (pruebas rápidas)
bash lanzar_pc1.sh --intersecciones=A1,B2,C3
```

### Opción B: Manual (para pruebas individuales)

```bash
# Terminal 1: Broker ZeroMQ
python pc1/broker_zmq.py                    # modo simple
python pc1/broker_zmq.py --modo multihilo   # modo multihilo

# Terminal 2+: Sensores (uno por intersección)
python pc1/sensor_camara.py  --interseccion INT-C5
python pc1/sensor_espira.py  --interseccion INT-C5
python pc1/sensor_gps.py     --interseccion INT-C5

# Cambiar intervalo de generación (para escenarios de prueba)
python pc1/sensor_camara.py  --interseccion INT-C5 --intervalo 10
python pc1/sensor_camara.py  --interseccion INT-C5 --intervalo 5
```

---

## 🖥️ PC2 — Analítica, Semáforos y BD Réplica

### Opción A: Script automático

```bash
bash lanzar_pc2.sh
```

### Opción B: Manual (terminales separadas)

```bash
# Terminal 1: BD Réplica (iniciar primero)
python pc2/base_datos_replica.py

# Terminal 2: Servicio de Semáforos
python pc2/servicio_semaforos.py

# Terminal 3: Servicio de Analítica (iniciar último)
python pc2/servicio_analitica.py
```

Los logs de cada servicio muestran en pantalla las operaciones en tiempo real.

---

## 🖥️ PC3 — BD Principal y Monitoreo

### Opción A: Script automático

```bash
bash lanzar_pc3.sh
```

### Opción B: Manual

```bash
# Terminal 1: BD Principal + Heartbeat (iniciar primero)
python pc3/base_datos_principal.py

# Terminal 2: Monitoreo y consulta (CLI interactiva)
python pc3/servicio_monitoreo.py
```

---

## 📊 Uso del servicio de monitoreo (PC3)

Al ejecutar `servicio_monitoreo.py`, aparece el menú:

```
╔══════════════════════════════════════════════════════════════╗
║          SISTEMA DE GESTIÓN INTELIGENTE DE TRÁFICO           ║
║                   Monitoreo y Consulta                        ║
╠══════════════════════════════════════════════════════════════╣
║  1. Estado actual de una intersección                         ║
║  2. Estado de TODO el sistema                                 ║
║  3. Historial de tráfico por intersección y período           ║
║  4. Historial de congestiones                                 ║
║  5. Estadísticas generales de la BD                           ║
║  6. 🚨  Activar paso de ambulancia (ola verde)               ║
║  7. Forzar cambio de semáforo en intersección                 ║
║  8. Verificar estado del sistema (ping a analítica)           ║
║  0. Salir                                                     ║
╚══════════════════════════════════════════════════════════════╝
```

### Ejemplo: Activar ambulancia (opción 6)

```
Opción > 6
Ingrese las intersecciones de la ruta de la ambulancia.
Ejemplo: INT-A1, INT-A2, INT-A3
Intersecciones (separadas por coma): INT-A1, INT-A2, INT-A3, INT-A4
🚨 ¡Prioridad activada! Ola verde en: ['INT-A1', 'INT-A2', 'INT-A3', 'INT-A4']
```

---

## 🔴 Simulación de falla del PC3

Para simular la falla del PC3:

```bash
# En PC3: detener la BD principal (Ctrl+C o kill)
# El servicio de analítica detectará la falla automáticamente
# (timeout de heartbeat: 9 segundos)

# Verificar en los logs de PC2 (analitica.log):
# "⚠️  PC3 no responde desde hace Xs — usando réplica en PC2"
```

La falla es transparente para el sistema: la BD réplica continúa recibiendo
todos los datos. Al restaurar PC3, la analítica reconecta automáticamente.

---

## 🧪 Escenarios de prueba (Tabla 1 del enunciado)

### Escenario 1: 1 sensor de cada tipo, intervalo 10 segundos

```bash
# PC1 — un solo juego de sensores con intervalo 10s
python pc1/broker_zmq.py &
python pc1/sensor_camara.py --interseccion INT-C5 --intervalo 10 &
python pc1/sensor_espira.py --interseccion INT-C5 --intervalo 10 &
python pc1/sensor_gps.py    --interseccion INT-C5 --intervalo 10 &
```

**Variable a medir**: Registros en BD en 2 minutos → máximo esperado: `2*60/10 * 3 = 36 eventos`.

### Escenario 2: 2 sensores de cada tipo, intervalo 5 segundos

```bash
python pc1/broker_zmq.py &
python pc1/sensor_camara.py --interseccion INT-C5 --intervalo 5 &
python pc1/sensor_camara.py --interseccion INT-D3 --intervalo 5 &
python pc1/sensor_espira.py --interseccion INT-C5 --intervalo 5 &
python pc1/sensor_espira.py --interseccion INT-D3 --intervalo 5 &
python pc1/sensor_gps.py    --interseccion INT-C5 --intervalo 5 &
python pc1/sensor_gps.py    --interseccion INT-D3 --intervalo 5 &
```

**Variable a medir**: Registros en BD en 2 minutos → máximo esperado: `2*60/5 * 3 * 2 = 144 eventos`.

### Medición del tiempo de respuesta (semáforo)

Para medir el tiempo desde que el usuario envía un comando hasta que el semáforo cambia:

1. Registrar timestamp antes de enviar comando desde monitoreo.
2. Revisar logs de `servicio_semaforos.py` para el timestamp del cambio.
3. La diferencia es el tiempo de respuesta end-to-end.

---

## 🏗️ Arquitectura de puertos

| Puerto | Protocolo | Origen → Destino | Descripción |
|--------|-----------|------------------|-------------|
| 5555   | XSUB      | Sensores → Broker PC1 | Sensores publican eventos |
| 5556   | XPUB      | Broker PC1 → Analítica PC2 | Broker reenvía a analítica |
| 5557   | PULL      | Analítica PC2 → Semáforos PC2 | Comandos de control |
| 5558   | PULL      | Analítica PC2 → BD Réplica PC2 | Persistencia réplica |
| 5559   | PULL      | Analítica PC2 → BD Principal PC3 | Persistencia principal |
| 5560   | REP       | Monitoreo PC3 → Analítica PC2 | Consultas y comandos |
| 5561   | PUB       | BD Principal PC3 → Analítica PC2 | Heartbeat |

---

## 📋 Reglas de tráfico implementadas

| Estado | Condición | Acción semáforo |
|--------|-----------|-----------------|
| NORMAL | Q < 5 AND Vp > 35 AND Cv < 10 | RESETEAR (ciclo estándar 15s) |
| MODERADO | Intermedio | EXTENDER verde +5s |
| CONGESTIÓN | Q ≥ 10 OR Vp < 15 OR Cv ≥ 20 | CAMBIAR a verde 30s (modo CONGESTIÓN) |
| PRIORIDAD | Comando manual | Ola verde 60s en ruta indicada |

---

## 📦 Dependencias

```
pyzmq >= 25.0
Python >= 3.9
sqlite3 (incluido en Python)
```

Instalación:
```bash
pip install pyzmq
```
