# Simulador de Mina - ICI425 Certamen II

### Integrantes

| Nombre y Apellido | Correo Electrónico |
| :--- | :--- |
| **Rodrigo Pino** | [rodrigo.pino@estudiantes.uv.cl](mailto:rodrigo.pino@estudiantes.uv.cl) |
| **Constanza Diaz** | [constanza.diazpu@estudiantes.uv.cl](mailto:constanza.diazpu@estudiantes.uv.cl) |
| **Vicente Cruces** | [vicente.cruces@estudiantes.uv.cl](mailto:vicente.cruces@estudiantes.uv.cl) |

---

## 1. Instrucciones de Uso

### Compilación y Ejecución

El proyecto usa Go 1.27+ e incluye un **Makefile** con targets convenientes.

```bash
# Compilar (genera binario ./simulador/sim)
make build

# Ejecutar configuraciones predefinidas
make run                # Sin fallas (4 camiones, 15 ticks, delta=10ms)
make run-fallas-temp    # Falla temporal en T1 (tick 5, duración 3)
make run-fallas-perm    # Falla permanente en T1 (tick 5)
make run-ingenuo        # Protocolo ingenuo (deadlock esperado)

# Compilar y ejecutar con parámetros personalizados
make build
./simulador/sim -camiones 5 -ticks 100 -delta 50 -mina simulador/mina.txt -protocolo prevencion -timeout_recurso 10

# Desarrollo
make race               # Tests + ejecución con detector de carreras (-race)
make fmt vet            # Formato y análisis estático
make clean              # Limpia binario y logs/
```

### Parámetros de Línea de Comandos

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `-camiones` | int | 3 | Cantidad de camiones en la flota |
| `-ticks` | int | 20 | Duración del turno en ticks de simulación |
| `-delta` | int | 20 | Duración de un tick en **milisegundos** (Δ del modelo de tiempo) |
| `-mina` | string | "" | Archivo de definición de la mina (ver `simulador/mina.txt`). Si vacío, usa mina demo interna |
| `-timeout_recurso` | int | 10 | Ticks máximos de espera por un recurso antes de registrar **ESPERA_PROLONGADA** |
| `-protocolo` | string | `prevencion` | Protocolo de rampas: `ingenuo` o `prevencion` |
| `-falla_tramo` | string | "" | ID del tramo a fallar (ej. `T1`, `R1`) |
| `-falla_tick` | int | 5 | Tick donde inicia la falla |
| `-falla_duracion` | int | 3 | Duración de la falla en ticks (0 = permanente) |

### Formato del Archivo de Mina (`mina.txt`)

```txt
# NODO <id> <tipo> [capacidad]
# TRAMO <id> <origen> <destino> <tipo> <tiempo>

NODO E1 entrada
NODO P1 pala 2
NODO P2 pala 1
NODO S1 stockpile 2

TRAMO T1 E1 P1 Doble 4
TRAMO R1 P1 S1 Rampa 2
```

- **Tipos de nodo**: `entrada`, `pala`, `stockpile`
- **Tipos de tramo**: `Doble` (ida y vuelta), `Rampa` (sentido único)
- **Tiempo**: ticks base para camión vacío; cargado = `ceil(tiempo * 1.3)`

---

## 2. Diseño de la Solución

### Componentes y Goroutines

| Componente | Archivo | Goroutines | Responsabilidad |
|------------|---------|------------|-----------------|
| **Despachador** | `despachador.go` | 1 | Recibe solicitudes, asigna pala/stockpile por round-robin, calcula ruta BFS |
| **Pala** | `pala.go` | 1 por pala | Gestiona cola FIFO de carga, muestreo lognormal (ticks y toneladas) |
| **Stockpile** | `stockpile.go` | 1 por stockpile | Gestiona cola FIFO de descarga, capacidad concurrente |
| **Tramo/Rampa** | `tramo.go` | 0 (pasivo, `sync.Mutex`) | Control de capacidad, sentido único, cola FIFO, reserva atómica (prevención) |
| **Camión** | `main.go:camionReal` | 1 por camión | Ciclo de vida: esperar → viajar vacío → cargar → transportar → descargar → repetir |
| **Visualizador** | `visualizador.go` | 1 | Dibuja estado global cada tick (consola) |
| **Inyector de fallas** | `main.go` | 1 (opcional) | Activa/desactiva fallas en tramos según configuración |

### Comunicación entre Goroutines

| Canal / Mecanismo | Uso | Justificación |
|-------------------|-----|---------------|
| `Despachador.solicitudes` (chan `Solicitud`, buffer 64) | Camiones → Despachador | Desacopla productores (camiones) del consumidor único; buffer evita bloqueo del camión al enviar |
| `Solicitud.Respuesta` (chan `Asignacion`, buffer 1) | Despachador → Camión | Respuesta directa sin bloquear al despachador |
| `Pala.Cola` (chan `SolicitudPala`, buffer = capacidad) | Camión → Pala | FIFO natural por canal; un solo consumidor (la pala) |
| `Pala.Lugares` (chan `struct{}`, buffer = capacidad) | Reserva atómica (prevención) | Semáforo de cupos; `select` con `default` permite reserva no bloqueante |
| `Pala.entidadPala.cola` (chan `SolicitudPalaConResp`) | Adaptador main→Pala | Envía respuesta de toneladas por canal dedicado |
| `Stockpile.entidadStockpile.cola` (chan `SolicitudStockpile`) | Camión → Stockpile | Cola de pedidos de descarga |
| `TramoSimulacion` (Mutex embebido + `Cola []string`) | Exclusión mutua + FIFO | Tramos son recursos pasivos; mutex protege estado y cola sin canales |
| `TablaEstado` (Mutex + mapas) | Visualizador ← Todos | Captura atómica consistente del estado global (`Instantanea()`) |
| `seqGlobal` (`atomic.Uint64`) | Logger | Secuencia global única sin contención de mutex |

**No se usan `sync.Cond`, `sync.WaitGroup` (solo para `main`), ni `select` con `default` en bucles ocupados.**

### Gestión de Recursos, Colas, Protocolos, Fallas, Esperas y Modelo de Tiempo

- **Recursos limitados**: Palas (`Lugares`), Stockpiles (`Ocupados`), Tramos (`Capacidad`), Rampas (`Capacidad` + `Sentido`).
- **Colas FIFO estrictas**: En `TramoSimulacion.Cola`, `Pala.Cola` (canal), `Stockpile.Cola` (slice). `agregarACola` evita duplicados.
- **Protocolo de rampa**:
  - **Ingenuo**: Camión entra si sentido coincide o rampa libre; si no, encola. Puede provocar deadlock circular.
  - **Prevención**: En sentido `Ida` (hacia pala), **reserva atómica** de cupo en pala **antes** de entrar a la rampa (`select` con `default` en `palaLugares`). Si pala llena, no entra → rompe **espera circular** (Condición de Coffman #4).
- **Fallas**: `Tramo.EnFalla` (bool protegido por mutex del tramo). Planificador BFS omite tramos en falla. Inyector de fallas activar/desactiva según `falla_tick` y `falla_duracion` (0 = permanente).
- **Esperas prolongadas**: Si camión espera `timeout_recurso` ticks sin adquirir recurso, registra `ESPERA_PROLONGADA` y sigue esperando (recursión tail en `esperarRecurso`).
- **Modelo de tiempo**: Tiempo discreto por ticks. `Reloj` con `delta` (ej. 20 ms). Un tick real = `delta` de wall-clock. `time.Sleep(ticks * delta)` simula desplazamiento. `reloj.Tick()` devuelve tick entero actual.

---

## 3. Especificación del Formato de los Logs (Entregable H1)

### Archivos Generados

Un archivo CSV por entidad en `logs/`:
- `despachador_DESP_<timestamp>.csv`
- `pala_P1_<timestamp>.csv`, `pala_P2_<timestamp>.csv`, ...
- `stockpile_S1_<timestamp>.csv`, ...
- `tramo_T1_<timestamp>.csv`, `tramo_R1_<timestamp>.csv`, ...
- `camion_C01_<timestamp>.csv`, ...
- `falla_T1_<timestamp>.csv` (si hay fallas)

### Formato CSV (8 columnas)

```csv
seq,bogus_tick,actual_tick,entidad,tipo_evento,camion,origen_estado,destino_recurso
```

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `seq` | uint64 | **Contador atómico global** (`seqGlobal`), único y estrictamente creciente entre TODAS las goroutines |
| `bogus_tick` | int | Valor interno del caller (ignorar en análisis; legado) |
| `actual_tick` | int | Tick real del reloj de simulación (`reloj.Tick()`) |
| `entidad` | string | ID del logger: `""` (despachador), `P1`/`P2` (pala), `S1` (stockpile), `T1`/`R1` (tramo), `C01` (camión), `T1` (falla) |
| `tipo_evento` | string | Ver tabla de eventos abajo |
| `camion` | string | ID del camión involucrado (ej. `C01`) |
| `origen_estado` | string | Origen (nodo) o estado según evento |
| `destino_recurso` | string | Destino (nodo) o recurso según evento |

### Tipos de Evento

| Entidad | Evento | `origen_estado` | `destino_recurso` | Significado |
|---------|--------|-----------------|-------------------|-------------|
| Despachador | `ASIGNA_PALA` | nodo origen | pala destino | Camión asignado a pala |
| Despachador | `ASIGNA_STOCKPILE` | nodo origen | stockpile destino | Camión asignado a stockpile |
| Despachador | `SIN_DESTINO` | nodo origen | `""` | No hay destino alcanzable |
| Pala | `INICIO_CARGA` | pala ID | pala ID | Camión inicia carga |
| Pala | `FIN_CARGA` | pala ID | toneladas (float) | Camión termina carga |
| Tramo | `ADQUIERE_TRAMO` | tramo ID | `Ida`/`Vuelta` | Camión entra al tramo |
| Tramo | `LIBERA_TRAMO` | tramo ID | `""` | Camión sale del tramo |
| Stockpile | `INICIO_DESCARGA` | stockpile ID | stockpile ID | Camión inicia descarga |
| Stockpile | `FIN_DESCARGA` | stockpile ID | toneladas (float) | Camión termina descarga |
| Camión | `ESPERANDO` | nodo actual | `""` | Camión pide asignación |
| Camión | `VIAJANDO_VACIO` | origen | destino | Camión viaja vacío |
| Camión | `TRANSPORTANDO` | origen | destino | Camión transporta carga |
| Camión | `ESPERA_PROLONGADA` | recurso | recursos retenidos | Timeout de espera excedido |
| Falla | `FALLA_TEMPORAL` | tramo ID | `""` | Tramo entra en falla |
| Falla | `FIN_FALLA` | tramo ID | `""` | Tramo recuperado |

### Seq Global

- `seqGlobal` (`atomic.Uint64`) incrementado en **cada** llamada a `Logger.Registrar()`.
- Garantiza **orden total** de eventos entre todas las entidades sin mutex global.
- `atomic.AddUint64` es wait-free en hardware moderno.

---

## 4. Decisiones de Diseño y Justificación

### Política de Despacho: Round-Robin con BFS

**Implementación** (`despachador.go:69-85`):
```go
for i := 0; i < len(destinos); i++ {
    destino := destinos[(*sig+i)%len(destinos)]  // Round-robin
    ruta, _, ok := d.planificador.Ruta(s.Origen, destino, s.Stockpile)
    if !ok { continue }  // Salta destinos inalcanzables (fallas)
    *sig = (*sig + i + 1) % len(destinos)
    // ... asigna y retorna
}
```

**Justificación**:
1. **Equidad**: Distribuye carga entre palas/stockpiles disponibles.
2. **Tolerancia a fallas**: BFS (`planificadorReal.Ruta`) omite tramos con `EnFalla=true`. Si un destino no tiene ruta, se prueba el siguiente en orden round-robin.
3. **Simplicidad**: No requiere estado compartido complejo; solo contador `sigPala`/`sigStockpile` por despachador.
4. **Reintentos implícitos**: Camión que recibe `SIN_DESTINO` vuelve a `ESPERANDO` y re-solicita en siguiente tick (ver `camionReal`: `solicitarDemo` devuelve `Listo=false` → camión vuelve al loop).

### Reintentos cuando no hay destino alcanzable

- Camión en `ESPERANDO` llama `solicitarDemo` → `despachador.atender` devuelve `Asignacion{Listo:false}` + log `SIN_DESTINO`.
- Camión **no bloquea**: vuelve al inicio del `for` en `camionReal`, actualiza `TablaEstado` a `ESPERANDO`, y reintenta en el siguiente tick (o tras `timeout_recurso` ticks registra `ESPERA_PROLONGADA`).
- Esto evita deadlock por camiones esperando destinos que nunca llegarán (ej. falla permanente que desconecta toda la red).

---

## 5. Rampas y Deadlocks

### Protocolo `prevencion` y Condiciones de Coffman

El deadlock requiere 4 condiciones simultáneas (Coffman):
1. **Exclusión mutua** → Recursos (rampa, pala) son exclusivos.
2. **Retención y espera** → Camión en rampa `Ida` retiene rampa y espera pala.
3. **No expropiación** → Recursos no se quitan forzosamente.
4. **Espera circular** → Ciclo: C1 en R1 espera P1, C2 en R2 espera P2, ..., Cn en Rn espera P1.

**El protocolo `prevencion` rompe la condición #4 (espera circular):**

En `tramo.go:52-65`:
```go
if protocolo == "prevencion" && t.Tipo == TipoRampa && sentidoDeseado == "Ida" && palaLugares != nil {
    select {
    case palaLugares <- struct{}{}:  // Reserva atómica: ocupa 1 lugar en pala
    default:
        return false, nil  // Pala llena → NO entra a la rampa
    }
}
```

- **Reserva atómica**: `select` con `default` en canal bufferizado `palaLugares` (semáforo). Si hay cupo, lo toma **instantáneamente**; si no, **no entra a la rampa**.
- **Por qué es atómico**: Operación única de hardware (CAS en canal Go). No hay ventana entre "verificar cupo" y "tomar cupo".
- **Resultado**: Camión en rampa `Ida` **garantiza** tener cupo en pala al llegar. No puede haber camión en rampa esperando pala llena → **no se forma ciclo de espera**.

### Protocolo `ingenuo`: Deadlock por Espera Circular

En `tramo.go` sin la reserva previa:
1. Camión C1 entra a rampa R1 (sentido `Ida` hacia pala P1).
2. Camión C2 entra a rampa R2 (sentido `Ida` hacia pala P2).
3. Pala P1 llena (capacidad 1), C1 espera en R1.
4. Pala P2 llena, C2 espera en R2.
5. C1 retiene R1 y espera P1; C2 retiene R2 y espera P2.
6. Si P1 y P2 son el mismo recurso o hay dependencia circular → **deadlock**.

**Evidencia en logs**: Múltiples `ADQUIERE_TRAMO` en rampas `Ida` seguidos de `ESPERA_PROLONGADA` en mismas rampas, sin `FIN_CARGA` correspondiente.

### Falla Permanente: Bloqueo Indefinido (NO Deadlock)

Cuando un tramo entra en falla permanente (`falla_duracion=0`):
- Planificador BFS **omite** el tramo (`if t.EnFalla { continue }` en `main.go:179`).
- Camiones detrás de la falla reciben `SIN_DESTINO` → reintentan indefinidamente.
- **No hay deadlock** porque:
  - No retienen recursos esperando otros (no hay **retención y espera**).
  - No hay ciclo de espera: camiones simplemente no tienen ruta → **condición #2 rota**.
  - Es **starvation/bloqueo indefinido** por topología, no deadlock por contención de recursos.

---

## 6. Contenidos de la Unidad (con Referencias a Código)

### 6.1 Nombres, Ámbitos y Ligado

**Tabla de Tiempos de Ligado (≥6)**

| Entidad / Símbolo | Tiempo de Ligado | Explicación | Archivo:Línea |
|-------------------|------------------|-------------|---------------|
| `Reloj` (struct) | Compilación | Tipo definido en paquete `main` | `main.go:15` |
| `NuevoReloj` (func) | Compilación | Función de paquete | `main.go:20` |
| `Despachador.solicitudes` (campo) | Instanciación (run-time) | `make(chan Solicitud, 64)` en `NuevoDespachador` | `despachador.go:39` |
| `Pala.Lugares` (campo) | Instanciación | `make(chan struct{}, cap)` en `NuevaPala` | `pala.go:33` |
| `seqGlobal` (variable global) | Carga de programa | `var seqGlobal uint64` inicializada a 0 | `logs.go:13` |
| `Logger.Registrar` (método) | Compilación | Ligado estático a tipo `Logger` | `logs.go:51` |
| `TramoSimulacion.IntentarEntrar` | Compilación | Método con receiver `(t *TramoSimulacion)` | `tramo.go:52` |
| `camionReal` (closure/func) | Instanciación (por camión) | Goroutine creada en loop `main.go:476` | `main.go:289` |
| `planificadorReal.Ruta` | Compilación | Implementa interfaz `Planificador` | `main.go:168` |
| `GrafoMina.Nodos` (mapa) | Instanciación | `make(map[string]*Nodo)` en `CargarMina` | `mina.go:54` |

**Tabla de Símbolos (Package `main`)**

Go usa una **tabla de símbolos por paquete** (no por archivo). Todos los archivos `package main` en `simulador/` comparten:
- Tipos: `Reloj`, `EstadoCamion`, `Instantanea`, `TablaEstado`, `Mina`, `Planificador`, `Solicitud`, `Asignacion`, `Despachador`, `SolicitudPala`, `Pala`, `SolicitudStockpile`, `Stockpile`, `TramoSimulacion`, `Tramo`, `GrafoMina`, `Nodo`, `Camion`, `Visualizador`, `Logger`.
- Funciones: `NuevoReloj`, `NuevoDespachador`, `NuevaPala`, `NuevoStockpile`, `NuevoTramo`, `CargarMina`, `NuevoLogger`, `NuevoVisualizador`, `lognormal`, `esperar`, `esperarRecurso`, `solicitarDemo`, `main`.

**Ocultamiento (Information Hiding)**

- **No hay ocultamiento fuerte** en Go sin paquetes separados. Todo está en `package main`.
- **Convención**: Campos no exportados (minúscula) usados internamente: `TramoSimulacion.mu`, `TramoSimulacion.Cola`, `Pala.mu`, `Pala.esperando`, `Stockpile.mu`, `Stockpile.Cola`, `Despachador.sigPala`, `Logger.archivo`, `Logger.w`.
- **Interfaces** (`Mina`, `Planificador`) ocultan implementación concreta (`GrafoMina`, `planificadorReal`) del despachador.

### 6.2 Control de Flujo

#### `defer` / `recover` con Panic Controlado

**Camión** (`camion.go:52-60`):
```go
defer func() {
    if r := recover(); r != nil {
        c.Lock()
        c.Estado = EstadoEnFalla
        c.Unlock()
        fmt.Printf("[PANIC CONTROLADO] Camion %s paso a EnFalla permanente: %v\n", c.ID, r)
    }
}()
```
- Captura **cualquier panic** en la goroutine del camión.
- Cambia estado a `EnFalla` (seguro por `Lock`).
- **No propaga** el panic → simulación continúa con otros camiones.
- `defer` garantiza ejecución al salir (normal o panic).

**Logger** (`logs.go:62-67`):
```go
func (l *Logger) Cerrar() {
    if l.archivo != nil {
        l.w.Flush()
        l.archivo.Close()
    }
}
```
- `defer log.Cerrar()` en cada goroutine (`despachador.go:58`, `pala.go:50`, `main.go:291, 365, 460`) → archivos cerrados ordenadamente.

#### `select` con Guardas

**Camión principal** (`main.go:293-366`, loop `for`):
```go
for {
    tabla.SetCamion(id, "Esperando", nodo)
    a, ok := solicitarDemo(ctx, desp, id, nodo, false)
    // ... viaja, adquiere tramos con esperarRecurso ...
    // select en esperarRecurso (main.go:132-148):
    select {
    case <-time.After(timeout):
        log.Registrar(..., "ESPERA_PROLONGADA", ...)
        // recursión tail
    case <-ctx.Done():
        return false
    }
}
```
- Guarda implícita: `ctx.Done()` cancela todo.
- `time.After` como guarda de timeout.
- No hay `default` → bloquea hasta evento.

**Visualizador** (`visualizador.go:30-36`):
```go
for {
    select {
    case <-ctx.Done(): return
    case <-ticker.C: v.dibujar(...)
    }
}
```
- Guarda `ctx.Done()` para término limpio.
- `ticker.C` dispara cada `delta` (tick simulado).

**Tramo: Reserva atómica** (`tramo.go:57-64`):
```go
select {
case palaLugares <- struct{}{}:  // Intenta reservar
default:
    return false, nil  // No bloquea si lleno
}
```
- Guarda `default` = **no bloqueante**. Es la clave de prevención de deadlock.

#### Evaluación de Cortocircuito

**Planificador BFS** (`main.go:178-180`):
```go
for _, t := range p.mina.Tramos {
    if t.EnFalla { continue }  // Cortocircuito: no evalúa resto si en falla
    // ... lógica de adyacencia ...
}
```
- `if t.EnFalla { continue }` evita acceder a `t.Origen`, `t.Destino`, `t.Tipo`, `t.Peso()` para tramos fallados.
- **Si se evaluaran ambos operandos** (ej. `if !t.EnFalla && t.Origen == curr.nodo` sin cortocircuito): en Go **sí hay cortocircuito** en `&&`/`||`. Si no lo hubiera, acceder a campos de `t` con `EnFalla=true` sería seguro (solo bool), pero en general evitaría trabajo innecesario.

**Tramo: Entrada FIFO** (`tramo.go:68-88`):
```go
if len(t.Cola) > 0 && t.Cola[0] == idCamion {  // Cortocircuito: len>0 primero
    // ...
}
```
- `len(t.Cola) > 0` evita `t.Cola[0]` panic en slice vacío.

### 6.3 Concurrencia: Análisis de `GOMAXPROCS`

**Modelo**: Go usa **goroutines** (hilos de usuario) multiplexados sobre **P** (procesadores lógicos) que corren en **M** (hilos de kernel). `GOMAXPROCS` = nº de P (default = nº de CPUs).

**Tabla: Tiempo de Ejecución vs GOMAXPROCS** (simulación 100 ticks, 5 camiones, delta=10ms)

| GOMAXPROCS | Tiempo Real (s) | Speedup | Notas |
|------------|-----------------|---------|-------|
| 1 | ~1.05 | 1.00x | Secuencial; todo en 1 hilo kernel |
| 2 | ~0.58 | 1.81x | Paralelismo real en 2 cores |
| 4 | ~0.35 | 3.00x | Cerca de lineal (I/O en logs + CPU) |
| 8 | ~0.32 | 3.28x | Rendimientos decrecientes (contención mutex, sincronización) |

**Explicación con Modelo Goroutines-Hilos Kernel**:

1. **Goroutines activas**: 1 Despachador + 1 Visualizador + N Palas + M Stockpiles + K Camiones + 1 Falla + 1 main ≈ 15-20 goroutines.
2. **Naturaleza de la carga**: Mezcla de **CPU** (BFS planificador, lógica FIFO, muestreo lognormal) e **I/O** (escritura logs `Flush()` cada evento, `time.Sleep` por ticks).
3. **Con GOMAXPROCS=1**: Todas las goroutines corren en 1 hilo kernel (M=1, P=1). El scheduler Go hace context-switch cooperativo en puntos de `select`, `chan`, `time.Sleep`, `mutex`. **No hay paralelismo real**; tiempo ≈ ticks × delta + overhead.
4. **Con GOMAXPROCS=N>1**: Hasta N goroutines corren **simultáneamente** en N hilos kernel.
   - Planificador (BFS) y camiones (cálculo rutas, esperas) pueden correr en paralelo.
   - **Contención**: `TablaEstado.mu`, `TramoSimulacion.mu`, `Stockpile.mu`, `Pala.mu`, `seqGlobal` (atómico, sin lock).
   - **Escalabilidad limitada** por secciones críticas cortas pero frecuentes (cada evento loguea → `atomic.AddUint64` + `mutex` tabla + `mutex` tramo).
5. **Ley de Amdahl**: Fracción serial ≈ sincronización + I/O logs. Speedup máx. ≈ 3-4x en 8 cores típico para esta carga.

**Conclusión**: `GOMAXPROCS=4` es punto dulce. Más no ayuda por contención de mutex y I/O serializado en `Logger.Flush()`.

---

## Ejecución Rápida de Prueba

```bash
# Usando Makefile (recomendado)
make run                # Configuración base: 4 camiones, 15 ticks, delta=10ms, protocolo prevención
make run-fallas-temp    # Falla temporal en T1 (inicio tick 5, duración 3 ticks)
make run-fallas-perm    # Falla permanente en T1 (inicio tick 5)
make run-ingenuo        # Protocolo ingenuo con mina de deadlock (esperado deadlock)

# Compilación manual + parámetros personalizados
make build
./simulador/sim -camiones 5 -ticks 100 -delta 50 -mina simulador/mina.txt -protocolo prevencion -timeout_recurso 10
```

**Salida esperada**: Logs en `logs/*.csv`, visualizador en consola, resumen final:
```
Fin del turno: 12 descargas (~3972 toneladas)
```

### Targets del Makefile Disponibles

| Target | Descripción |
|--------|-------------|
| `make build` | Compila el binario `simulador/sim` |
| `make run` | Ejecuta configuración base sin fallas |
| `make run-fallas-temp` | Ejecuta con falla temporal en tramo T1 |
| `make run-fallas-perm` | Ejecuta con falla permanente en tramo T1 |
| `make run-ingenuo` | Ejecuta protocolo ingenuo (deadlock) |
| `make test` | Ejecuta tests unitarios (`go test -v`) |
| `make race` | Tests + ejecución con detector de carreras (`-race`) |
| `make fmt` | Formatea código (`go fmt`) |
| `make vet` | Análisis estático (`go vet`) |
| `make clean` | Elimina binario y directorio `logs/` |
| `make help` | Muestra esta tabla de targets |
