# Informe Técnico — Primera Entrega
## Proyecto: Gestión Inteligente de Tráfico Urbano
### Introducción a los Sistemas Distribuidos — 2026-30

---

## 1. Modelos del Sistema

### 1.1 Modelo Arquitectónico

El sistema sigue una arquitectura **cliente-servidor distribuida en tres niveles**, con comunicación orientada a mensajes vía ZeroMQ.

```
┌─────────────────────────────────────────────────────────────────────┐
│  PC1 — Capa de Adquisición                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ Sensor CAM  │  │ Sensor ESP  │  │ Sensor GPS  │  × N (25 int.)  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │   PUB (topic)  │                 │                         │
│         └────────────────┴─────────────────┘                         │
│                          │                                            │
│                 ┌─────────▼──────────┐                               │
│                 │   Broker ZeroMQ    │  XSUB/XPUB                   │
│                 │  (simple/multihilo)│                               │
│                 └─────────┬──────────┘                               │
└───────────────────────────┼─────────────────────────────────────────┘
                  SUB (todos los topics)
┌───────────────────────────┼─────────────────────────────────────────┐
│  PC2 — Capa de Procesamiento                                         │
│                 ┌─────────▼──────────┐                               │
│                 │  Servicio Analítica │◄── REP ── (Monitoreo PC3)   │
│                 └───┬──────┬──────┬──┘                               │
│           PUSH      │      │      │ PUSH                             │
│       ┌─────────────┘      │      └────────────────┐                │
│       ▼                    ▼ PUSH                   ▼               │
│  ┌─────────┐        ┌─────────────┐         (→ PC3 BD Principal)   │
│  │Semáforos│        │ BD Réplica  │                                  │
│  │  PULL   │        │   PULL/SQL  │                                  │
│  └─────────┘        └─────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  PC3 — Capa de Persistencia y Consulta                               │
│  ┌─────────────────────────┐      ┌────────────────────────┐        │
│  │    BD Principal         │      │  Servicio Monitoreo    │        │
│  │  PULL + REP + HB(PUB)  │      │     REQ → Analítica    │        │
│  └─────────────────────────┘      │     REQ → BD Principal │        │
│                                   └────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

**Patrones de comunicación utilizados:**

| Patrón | Conexión | Justificación |
|--------|----------|---------------|
| PUB/SUB | Sensores → Broker → Analítica | Desacoplamiento asíncrono; múltiples productores/consumidores |
| PUSH/PULL | Analítica → Semáforos, BD réplica, BD principal | Cola de trabajo asíncrona; no bloquea al productor |
| REQ/REP | Monitoreo ↔ Analítica, Monitoreo ↔ BD Principal | Consultas síncronas con respuesta garantizada |
| PUB/SUB | BD Principal (heartbeat) → Analítica | Detección de fallos sin polling activo |

---

### 1.2 Modelo de Interacción

#### Flujo normal: sensor detecta congestión

```
Sensor CAM      Broker ZMQ      Analítica       Semáforos       BD Réplica     BD Principal
    │                │               │               │               │               │
    │──PUB(evento)──►│               │               │               │               │
    │                │──SUB fwd─────►│               │               │               │
    │                │               │─clasificar()  │               │               │
    │                │               │  → CONGESTION │               │               │
    │                │               │──PUSH(cmd)───►│               │               │
    │                │               │               │─cambiarEstado()               │
    │                │               │               │──print(VERDE) │               │
    │                │               │──PUSH(reg)────┼───────────────►               │
    │                │               │──PUSH(reg)────┼───────────────┼──────────────►│
```

#### Flujo: usuario solicita paso de ambulancia

```
Monitoreo       Analítica       Semáforos       BD Réplica     BD Principal
    │               │               │               │               │
    │─REQ(AMBUL)───►│               │               │               │
    │               │──PUSH(PRIOR)─►│               │               │
    │               │               │─activarPrioridad()            │
    │               │               │──print(OLA VERDE)             │
    │               │──PUSH(reg)────┼───────────────►               │
    │               │──PUSH(reg)────┼───────────────┼──────────────►│
    │◄──REP(ok)─────│               │               │               │
```

#### Flujo: falla de PC3 (detección por heartbeat)

```
BD Principal    Analítica       BD Réplica
    │               │               │
    │──HB(PUB)─────►│               │
    │               │ (timeout 9s)  │
    ✕ (PC3 caído)   │               │
    │               │──⚠️ PC3 caído  │
    │               │ (solo réplica) │
    │               │──PUSH(reg)────►│  ← toda la persistencia aquí
    │               │               │
    │ (PC3 vuelve)  │               │
    │──HB(PUB)─────►│               │
    │               │──✅ PC3 activo │
    │               │──PUSH(reg)────►│  (réplica)
    │               │──PUSH(reg)────────────────────► (principal)
```

---

### 1.3 Modelo de Fallos

#### Tabla de fallos y estrategias de resiliencia

| Componente | Tipo de fallo | Detección | Estrategia de recuperación |
|------------|---------------|-----------|---------------------------|
| Sensor individual | Proceso termina | Ausencia de mensajes | Los otros sensores siguen publicando; el sistema opera con datos parciales |
| Broker ZMQ (PC1) | Proceso termina | Sensores no pueden conectar | Los sensores intentan reconectar automáticamente al reiniciarse el broker |
| Servicio Analítica (PC2) | Proceso termina | Sensores acumulan en buffer ZMQ | Al reiniciar, los mensajes en buffer se procesan |
| BD Réplica (PC2) | Proceso termina | Error PUSH en analítica | Analítica registra error; los datos se pierden hasta reinicio |
| BD Principal (PC3) | ★ **Fallo relevante** | **Heartbeat timeout (9s)** | **Analítica cambia a solo-réplica automáticamente** |
| Monitoreo (PC3) | Proceso termina | N/A | Usuario puede relanzar; el sistema sigue operando |

#### Implementación del health check (heartbeat)

- La BD principal en PC3 publica un mensaje `heartbeat` cada 3 segundos.
- La analítica en PC2 tiene un `MonitorPC3` que registra la última vez que recibió el heartbeat.
- Si pasan más de 9 segundos (= 3× el intervalo) sin heartbeat, `MonitorPC3.activo` retorna `False`.
- La analítica deja de hacer PUSH a PC3 y opera con la réplica.
- Cuando PC3 se recupera y el heartbeat llega de nuevo, la analítica reanuda el envío.

**Limitación conocida**: Los datos generados mientras PC3 estaba caído **no** se sincronizan automáticamente a la BD principal al recuperarse. Esto podría implementarse con un mecanismo de "catch-up" comparando timestamps.

---

### 1.4 Modelo de Seguridad

| Aspecto | Implementación actual | Mejora recomendada |
|---------|-----------------------|--------------------|
| **Autenticación** | No implementada (red interna) | ZeroMQ CURVE (clave pública/privada) |
| **Confidencialidad** | Mensajes en texto plano (JSON) | Cifrado TLS sobre ZMQ |
| **Integridad** | No verificada | HMAC en cada mensaje |
| **Control de acceso** | Cualquier cliente puede publicar | Lista blanca de IPs en el broker |
| **Auditoría** | Logs de todas las operaciones | Persistencia de logs con timestamps |

**Para el contexto del laboratorio** (red cerrada y controlada), no se implementan mecanismos criptográficos, pero la arquitectura es compatible con la adición de ZMQ CURVE sin modificar la lógica de negocio.

---

## 2. Diseño del Sistema

### 2.1 Diagrama de Despliegue

```
«dispositivo»                «dispositivo»                «dispositivo»
     PC1                          PC2                          PC3
192.168.X.101               192.168.X.102               192.168.X.103
─────────────               ─────────────               ─────────────
«proceso»                   «proceso»                   «proceso»
broker_zmq.py               base_datos_replica.py       base_datos_principal.py
  :5555 (XSUB)                :5558 (PULL)                :5559 (PULL)
  :5556 (XPUB)                trafico_replica.db          :5561 (PUB heartbeat)
                                                          trafico_principal.db
«proceso» × 75              «proceso»                   :5570 (REP consultas)
sensor_camara.py            servicio_semaforos.py
sensor_espira.py              :5557 (PULL)              «proceso»
sensor_gps.py                                           servicio_monitoreo.py
(25 intersecciones           «proceso»                   → CLI interactiva
 × 3 tipos)                 servicio_analitica.py        REQ → PC2:5560
                              :5560 (REP)                REQ → PC3:5570
                              SUB → PC1:5556
                              PUSH → :5557
                              PUSH → :5558
                              PUSH → PC3:5559
                              SUB → PC3:5561
```

### 2.2 Diagrama de Componentes

```
┌──────────────────────── PC1 ────────────────────────────┐
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │SensorCamara│  │SensorEspira│  │  SensorGPS │        │
│  │            │  │            │  │            │        │
│  │«PUB»camara │  │«PUB»espira │  │«PUB»gps    │        │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘        │
│         └───────────────┴───────────────┘               │
│                         │ tcp:5555                       │
│                ┌────────▼────────┐                       │
│                │   BrokerZMQ     │                       │
│                │ «XSUB» «XPUB»  │                       │
│                │  simple/multi  │                       │
│                └────────┬────────┘                       │
└─────────────────────────┼────────────────────────────────┘
                          │ tcp:5556 SUB
┌─────────────────────────┼────────────── PC2 ─────────────┐
│                ┌────────▼────────┐                        │
│                │ ServicioAnalitica│                        │
│                │                 │◄─── REP tcp:5560 ──────┼──┐
│                │ clasificar()    │                        │  │
│                │ reglas_trafico()│                        │  │
│                └──┬──────┬───┬───┘                        │  │
│           PUSH    │      │   │ PUSH(BD principal)         │  │
│       tcp:5557    │   tcp:5558  tcp:5559                  │  │
│    ┌──────────────┘      │   └──────────────────────────┐ │  │
│    ▼                     ▼                              │ │  │
│  ┌────────────┐  ┌───────────────┐                      │ │  │
│  │  Servicio  │  │  BD Réplica   │                      │ │  │
│  │ Semáforos  │  │ (SQLite)      │                      │ │  │
│  │ PULL:5557  │  │ PULL:5558     │                      │ │  │
│  └────────────┘  └───────────────┘                      │ │  │
└──────────────────────────────────────────────────────────┼─┘  │
                                                           │     │
┌──────────────────────────────────────────────────────────┼─────┼────── PC3 ──┐
│                                                          ▼     │             │
│  ┌──────────────────────────────────┐   ┌───────────────────────────────┐   │
│  │     BD Principal (SQLite)        │   │  Servicio Monitoreo (CLI)     │   │
│  │  PULL:5559 ← datos analítica     │   │  REQ → Analítica PC2:5560     │   │
│  │  PUB:5561  → heartbeat           │   │  REQ → BD Principal:5570      │   │
│  │  REP:5570  ← consultas monitor  │   │  Consultas históricas         │   │
│  └──────────────────────────────────┘   │  Comandos directos            │   │
│                                          └───────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Diagrama de Clases (clases principales)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  common/config_loader.py                                                 │
├─────────────────────────────────────────────────────────────────────────┤
│ + cargar_config(path:str) : dict                                         │
│ + obtener_intersecciones(config:dict) : list[str]                       │
│ + obtener_ip(config:dict, pc:str) : str                                 │
│ + obtener_puerto(config:dict, nombre:str) : int                         │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│  EstadoInterseccion              │  │  GestorSemaforos                 │
├──────────────────────────────────┤  ├──────────────────────────────────┤
│ - interseccion: str              │  │ - _semaforos: dict[str,dict]     │
│ - Q: float                       │  │ - _lock: threading.Lock          │
│ - Vp: float                      │  │ - _running: bool                 │
│ - Cv: float                      │  │ - _timer_thread: Thread          │
│ - nivel_gps: str                 │  ├──────────────────────────────────┤
│ - estado_actual: str             │  │ + cambiar_estado(int,est,modo)   │
│ - ts_ultima_actualizacion: str   │  │ + extender_verde(int,seg)        │
├──────────────────────────────────┤  │ + activar_prioridad(vias)        │
│ + actualizar_camara(evento:dict) │  │ + consultar(int) : dict          │
│ + actualizar_espira(evento:dict) │  │ + consultar_todos() : dict       │
│ + actualizar_gps(evento:dict)    │  │ - _bucle_timer()                 │
│ + to_dict() : dict               │  │ + detener()                      │
└──────────────────────────────────┘  └──────────────────────────────────┘

┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│  MonitorPC3                      │  │  ClienteAnalitica                │
├──────────────────────────────────┤  ├──────────────────────────────────┤
│ - _timeout: float                │  │ - _config: dict                  │
│ - _ultimo_heartbeat: float       │  │ - _ctx: zmq.Context              │
│ - _pc3_activo: bool              │  │ - _sock: zmq.Socket (REQ)        │
│ - _lock: threading.Lock          │  ├──────────────────────────────────┤
├──────────────────────────────────┤  │ + enviar(payload:dict) : dict    │
│ + registrar_heartbeat()          │  │ + cerrar()                       │
│ + verificar() : bool             │  └──────────────────────────────────┘
│ + activo : bool (property)       │
└──────────────────────────────────┘  ┌──────────────────────────────────┐
                                      │  ClienteBD                       │
 Funciones de clasificación:          ├──────────────────────────────────┤
 ┌────────────────────────────────┐   │ - _ctx: zmq.Context              │
 │ clasificar_trafico(Q,Vp,Cv,    │   │ - _sock_main: zmq.Socket (REQ)   │
 │   config) : str                │   │ - _sock_replica: zmq.Socket(REQ) │
 │ determinar_accion_semaforo(    │   │ - _pc3_activo: bool              │
 │   estado, interseccion) : dict │   ├──────────────────────────────────┤
 │ clasificar_congestion(vel)     │   │ + consultar(payload:dict): dict  │
 │   : str                        │   │ - _reconectar()                  │
 └────────────────────────────────┘   └──────────────────────────────────┘
```

### 2.4 Diagrama de Secuencia — Detección de congestión y cambio de semáforo

```
SensorCAM    Broker    Analítica   Semáforos   BD-Replica   BD-Principal
   │            │           │           │           │            │
   │─PUB(cam)──►│           │           │           │            │
   │            │─forward──►│           │           │            │
   │            │           │           │           │            │
   │            │           │ clasificar(Q=12,Vp=12,Cv=22)       │
   │            │           │ → "CONGESTION"        │            │
   │            │           │           │           │            │
   │            │           │─PUSH({accion:CAMBIAR, │            │
   │            │           │  interseccion:INT-C5, ►           │
   │            │           │  nuevo_estado:VERDE,  │            │
   │            │           │  modo:CONGESTION})    │            │
   │            │           │           │           │            │
   │            │           │           │ cambiar_estado()        │
   │            │           │           │ estado=VERDE, t=30s    │
   │            │           │           │ print("[CAMBIO]...")   │
   │            │           │           │           │            │
   │            │           │─PUSH(reg)─────────────►           │
   │            │           │           │           │ INSERT     │
   │            │           │─PUSH(reg)─────────────────────────►│
   │            │           │           │           │ INSERT     │
```

---

## 3. Sección A — Inicialización de Recursos

### ¿Cómo obtienen los procesos la definición inicial de los recursos?

Todos los procesos carganel archivo `config/config.json` al iniciar, usando `common/config_loader.py`. Este archivo define:

#### Parámetros de la ciudad

```json
"ciudad": {
    "filas": ["A", "B", "C", "D", "E"],   ← 5 filas
    "columnas": [1, 2, 3, 4, 5]            ← 5 columnas
}
```

Esto genera **25 intersecciones** (INT-A1 a INT-E5). La función `obtener_intersecciones()` construye la lista completa automáticamente.

#### Parámetros de sensores

```json
"sensores": {
    "intervalo_camara_seg": 5,
    "intervalo_espira_seg": 30,
    "intervalo_gps_seg": 10
}
```

Cada sensor puede **sobreescribir** el intervalo por argumentos de línea de comandos (`--intervalo`), lo que permite los escenarios de prueba del experimento.

#### Proceso de inicialización por componente

| Componente | Parámetros recibidos | Fuente |
|------------|---------------------|--------|
| Sensor (cualquier tipo) | `--interseccion INT-XX`, `--intervalo N` | Args CLI + config.json |
| Broker ZMQ | `--modo simple/multihilo` | Args CLI + config.json |
| Analítica | Ninguno (todo de config.json) | config.json |
| Semáforos | Ninguno (inicializa de config.json) | config.json |
| BD (réplica/principal) | Ninguno | config.json |
| Monitoreo | Ninguno | config.json |

#### Determinación del número de semáforos

El `GestorSemaforos` (en `servicio_semaforos.py`) llama `obtener_intersecciones(config)` al inicializarse, creando automáticamente **un semáforo por intersección**. Con la cuadrícula 5×5 esto da **25 semáforos** en estado inicial ROJO.

#### Determinación del número de sensores

Para la primera entrega, cada proceso sensor cubre **una intersección**. El script `lanzar_pc1.sh` lanza 3 procesos por intersección (CAM + ESP + GPS), dando **75 procesos** para la cuadrícula completa. Los grupos pueden reducir esto para pruebas.

---

## 4. Sección B — Reglas, Consultas e Indicaciones Directas

### 4.1 Reglas de tráfico

Las reglas se aplican en `servicio_analitica.py::clasificar_trafico()` usando las métricas:
- **Q** = longitud de cola (vehículos esperando, del sensor cámara)
- **Vp** = velocidad promedio en km/h (de cámara o GPS)
- **Cv** = vehículos contados en 30 segundos (de espira inductiva)

#### Regla NORMAL (tráfico fluye con normalidad)
```
Q < 5 AND Vp > 35 AND Cv < 10
```
→ Semáforos en ciclo estándar de 15 segundos verde / 15 segundos rojo.

**Ejemplo**: INT-B2 con Q=3, Vp=40, Cv=7 → NORMAL → semáforo RESETEAR a ciclo estándar.

#### Regla CONGESTIÓN (tráfico severamente afectado)
```
Q >= 10 OR Vp < 15 OR Cv >= 20
```
→ Semáforo de la vía afectada pasa a VERDE por 30 segundos (modo CONGESTION). La roja es 10 segundos.

**Ejemplo**: INT-C5 con Q=15, Vp=8, Cv=25 → CONGESTION → semáforo CAMBIAR a VERDE 30s.

#### Regla MODERADO (zona intermedia)
```
(no NORMAL) AND (no CONGESTIÓN)
```
→ Extender fase verde +5 segundos para aliviar acumulación sin priorizar completamente.

**Ejemplo**: INT-A3 con Q=7, Vp=22, Cv=14 → MODERADO → EXTENDER +5s.

#### Regla PRIORIDAD (manual — ola verde)
```
Comando directo del usuario via monitoreo
```
→ Todas las intersecciones indicadas pasan a VERDE por 60 segundos; el resto a ROJO.

**Ejemplo de activación**: paso de ambulancia por ruta A1→A2→A3→A4.

### 4.2 Tipos de consulta disponibles en el servicio de monitoreo

| Tipo | Descripción | Canal |
|------|-------------|-------|
| `ESTADO_INTERSECCION` | Estado actual de Q, Vp, Cv y clasificación de tráfico para una intersección específica | REQ/REP → Analítica |
| `ESTADO_SISTEMA` | Estado de todas las intersecciones simultáneamente + estado de PC3 | REQ/REP → Analítica |
| `HISTORICO` | Registros de sensores para una intersección en un rango de fechas | REQ/REP → BD Principal |
| `CONGESTION` | Log histórico de todas las congestiones detectadas | REQ/REP → BD Principal |
| `ESTADISTICAS` | Conteos totales de eventos, congestiones, prioridades por intersección | REQ/REP → BD Principal |

### 4.3 Indicaciones directas del monitoreo a la analítica

| Indicación | Efecto | Ejemplo de payload |
|------------|--------|-------------------|
| `AMBULANCIA` | Activa ola verde en lista de intersecciones | `{"tipo":"AMBULANCIA","vias":["INT-A1","INT-A2","INT-A3"]}` |
| `CAMBIAR_SEMAFORO` | Fuerza el estado de un semáforo específico | `{"tipo":"CAMBIAR_SEMAFORO","interseccion":"INT-C5","nuevo_estado":"VERDE"}` |
| `HEARTBEAT` | Confirma que el monitoreo está activo (usado internamente) | `{"tipo":"HEARTBEAT"}` |

**Ejemplo de flujo ambulancia completo**:
1. Operador ingresa opción `6` en el monitoreo.
2. Ingresa ruta: `INT-D1, INT-D2, INT-D3, INT-D4, INT-D5`.
3. El monitoreo envía `{"tipo":"AMBULANCIA","vias":["INT-D1","INT-D2","INT-D3","INT-D4","INT-D5"]}` a la analítica.
4. La analítica envía `{"accion":"PRIORIDAD","vias":[...]}` al servicio de semáforos.
5. Los 5 semáforos de la ruta D cambian a VERDE por 60 segundos; los 20 restantes a ROJO.
6. El evento se registra en ambas BDs con tipo `"prioridad"`.

---

## 5. Protocolo de Pruebas

### 5.1 Pruebas funcionales

#### PF-01: Comunicación básica sensores → broker → analítica
- **Precondición**: PC1, PC2 y PC3 ejecutándose.
- **Pasos**: Iniciar 1 sensor de cada tipo en INT-C5. Verificar logs de la analítica.
- **Criterio de éxito**: La analítica imprime eventos de los 3 tipos de sensores en menos de 15 segundos.

#### PF-02: Detección de congestión y cambio de semáforo
- **Pasos**: Forzar valores altos de Q y bajos de Vp en el sensor simulado (modificar rangos en config.json temporalmente).
- **Criterio de éxito**: La analítica imprime "CONGESTION" y el servicio de semáforos imprime "[CAMBIO] INT-XX: ROJO → VERDE".

#### PF-03: Paso de ambulancia
- **Pasos**: Desde monitoreo opción 6, indicar ruta INT-A1,INT-A2,INT-A3.
- **Criterio de éxito**: Los 3 semáforos indicados cambian a VERDE y el evento aparece en la BD.

#### PF-04: Consulta histórica
- **Pasos**: Dejar correr el sistema 2 minutos. Usar opción 3 del monitoreo.
- **Criterio de éxito**: Se retornan registros ordenados por timestamp para la intersección consultada.

#### PF-05: Persistencia en ambas BDs
- **Pasos**: Contar registros en `trafico_replica.db` y `trafico_principal.db`.
- **Criterio de éxito**: Ambas BDs tienen la misma cantidad de registros (misma información).

### 5.2 Pruebas de tolerancia a fallos

#### PT-01: Falla del PC3 — detección automática
- **Pasos**: Con sistema funcionando, hacer `kill $(pgrep -f base_datos_principal)` en PC3.
- **Criterio de éxito**: En menos de 9 segundos, la analítica imprime "⚠️ PC3 no responde — usando réplica en PC2".

#### PT-02: Falla del PC3 — continuidad del servicio
- **Pasos**: Con PC3 caído, continuar operación 1 minuto.
- **Criterio de éxito**: Los sensores siguen generando, la analítica clasifica, los semáforos cambian, la réplica sigue acumulando registros.

#### PT-03: Recuperación del PC3
- **Pasos**: Reiniciar `base_datos_principal.py` en PC3.
- **Criterio de éxito**: La analítica imprime "✅ PC3 recuperado" y reanuda el envío a la BD principal.

#### PT-04: Falla de un sensor individual
- **Pasos**: Terminar el proceso `sensor_camara.py` de INT-C5.
- **Criterio de éxito**: El sistema sigue operando con datos de espira y GPS; la analítica no se detiene.

### 5.3 Pruebas de rendimiento (experimento Tabla 1)

#### Escenario E1: 1 sensor de cada tipo, intervalo 10s

```bash
# Iniciar broker + 3 sensores en una intersección
python pc1/broker_zmq.py --modo simple &
python pc1/sensor_camara.py --interseccion INT-C5 --intervalo 10 &
python pc1/sensor_espira.py --interseccion INT-C5 --intervalo 10 &
python pc1/sensor_gps.py    --interseccion INT-C5 --intervalo 10 &
```

**Variable dependiente 1**: Registros en BD en 2 minutos.

```sql
-- Ejecutar en PC3 después de 2 minutos
SELECT COUNT(*) FROM eventos_sensores
WHERE timestamp >= datetime('now', '-2 minutes');
```

**Variable dependiente 2**: Tiempo desde comando usuario hasta cambio de semáforo.
- Registrar `t1 = time.time()` antes de `enviar()` en el monitoreo.
- Registrar `t2` en el log del semáforo al imprimir `[CAMBIO]`.
- `Δt = t2 - t1`

#### Escenario E2: 2 sensores de cada tipo, intervalo 5s

```bash
python pc1/broker_zmq.py --modo simple &
for INT in C5 D3; do
  python pc1/sensor_camara.py --interseccion INT-$INT --intervalo 5 &
  python pc1/sensor_espira.py --interseccion INT-$INT --intervalo 5 &
  python pc1/sensor_gps.py    --interseccion INT-$INT --intervalo 5 &
done
```

#### Repetición de E1 y E2 con broker multihilo

Reemplazar `--modo simple` por `--modo multihilo` y repetir las mediciones.

---

## 6. Estrategias para obtener las métricas de rendimiento

### 6.1 Variable 1: Cantidad de solicitudes en BD en 2 minutos

**Método**: Consulta SQL directa a la BD SQLite al inicio y fin del período.

```python
import sqlite3, time

conn = sqlite3.connect("trafico_replica.db")

t0 = time.time()
# ... esperar 2 minutos ...
time.sleep(120)
t1 = time.time()

ts0 = datetime.fromtimestamp(t0).isoformat()
ts1 = datetime.fromtimestamp(t1).isoformat()

cur = conn.cursor()
cur.execute("""
    SELECT COUNT(*) FROM eventos_sensores
    WHERE timestamp BETWEEN ? AND ?
""", (ts0, ts1))
print(f"Registros en 2 minutos: {cur.fetchone()[0]}")
```

**Herramienta alternativa**: SQLite Browser para visualización.

### 6.2 Variable 2: Tiempo de respuesta usuario → semáforo cambia

**Método**: Instrumentación con timestamps en código.

En `servicio_monitoreo.py`, antes del `enviar()`:
```python
import time
t_inicio = time.perf_counter()
resp = analitica.enviar({"tipo": "AMBULANCIA", "vias": vias})
t_respuesta_analitica = time.perf_counter() - t_inicio
```

En `servicio_semaforos.py`, al procesar el comando `PRIORIDAD`:
```python
t_cambio = time.perf_counter()
# ... realizar cambio ...
print(f"[TIMING] Cambio en: {time.perf_counter() - t_cambio:.4f}s")
```

**Herramienta de medición**: `time.perf_counter()` (Python, resolución microsegundos). Registrar en archivo CSV para análisis posterior.

### 6.3 Representación gráfica de resultados

Usar Python con `matplotlib` o una hoja de cálculo:

```python
import matplotlib.pyplot as plt
import numpy as np

escenarios = ["E1-Simple", "E1-Multihilo", "E2-Simple", "E2-Multihilo"]
registros_bd = [36, 36, 144, 144]          # esperados (reemplazar con reales)
tiempo_respuesta_ms = [12.5, 8.3, 15.2, 9.1]  # reemplazar con reales

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
ax1.bar(escenarios, registros_bd, color=["#4472C4","#70AD47","#FFC000","#FF0000"])
ax1.set_title("Registros en BD en 2 minutos")
ax1.set_ylabel("Cantidad de registros")

ax2.bar(escenarios, tiempo_respuesta_ms, color=["#4472C4","#70AD47","#FFC000","#FF0000"])
ax2.set_title("Tiempo de respuesta usuario → semáforo (ms)")
ax2.set_ylabel("Tiempo (ms)")

plt.tight_layout()
plt.savefig("resultados_experimento.png", dpi=150)
plt.show()
```

---

## 7. Implementación de los Servicios PC1 y PC2

Ver código fuente adjunto en:

| Archivo | Componente | Estado |
|---------|------------|--------|
| `pc1/broker_zmq.py` | Broker ZMQ (simple + multihilo) | ✅ Implementado |
| `pc1/sensor_camara.py` | Sensor cámara (EVENTO_LONGITUD_COLA) | ✅ Implementado |
| `pc1/sensor_espira.py` | Sensor espira (EVENTO_CONTEO_VEHICULAR) | ✅ Implementado |
| `pc1/sensor_gps.py` | Sensor GPS (EVENTO_DENSIDAD_TRAFICO) | ✅ Implementado |
| `pc2/servicio_analitica.py` | Analítica + clasificación + control | ✅ Implementado |
| `pc2/servicio_semaforos.py` | Control de estados de semáforos | ✅ Implementado |
| `pc2/base_datos_replica.py` | BD réplica SQLite (PULL) | ✅ Implementado |
| `pc3/base_datos_principal.py` | BD principal + heartbeat | ✅ Implementado |
| `pc3/servicio_monitoreo.py` | CLI monitoreo y consultas | ✅ Implementado |

**Actualización de BD principal en PC3**: implementada mediante socket PUSH desde la analítica (PC2) al PULL de la BD principal (PC3). El servicio también expone un socket REP para consultas directas del monitoreo.

---

*Informe generado para la Primera Entrega — Sustentación viernes 10 de abril de 2026*
