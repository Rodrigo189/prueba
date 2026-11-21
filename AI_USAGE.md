# Registro de Uso de Inteligencia Artificial

**Proyecto:** Simulador de Operaciones Mineras (ICI425 - Cruces, Díaz, Pino)
**Repositorio:** https://github.com/usuario/ici425-cruces-diaz-pino (actualizar con URL real)
**Fecha de creación:** 2026-11-21

---

## 1. Alcance del Uso de IA

El uso de herramientas de inteligencia artificial (Claude Code, Anthropic) en este proyecto se limitó **exclusivamente** a los siguientes casos permitidos según el enunciado:

| Caso Permitido | Descripción |
|----------------|-------------|
| **Comprensión del problema / Inicio del trabajo** | Análisis de requerimientos, entendimiento de la arquitectura propuesta, definición de estructuras de datos iniciales, planificación de la concurrencia |
| **Interpretación de errores de compilación o ejecución** | Diagnóstico de errores del compilador Go, deadlocks en tiempo de ejecución, race conditions, panics, lógica de sincronización |

**Lo que NO se hizo con IA:**
- ❌ El código del simulador (goroutines, canales, mutex, lógica de negocio) fue **escrito íntegramente por el grupo**
- ❌ La integración entre módulos (main, despachador, palas, stockpiles, tramos, camiones) fue **desarrollada por el grupo**
- ❌ La adaptación de la simulación a los requerimientos específicos (protocolos ingenuo/prevención, fallas, logging CSV) fue **implementada por el grupo**
- ❌ No se copió ni pegó código generado por IA directamente al repositorio sin revisión y adaptación completa

---

## 2. Planificador de Rutas

> **Declaración:** No se importó código externo ni generado por IA para el planificador de rutas; fue implementado por el grupo.

| Aspecto | Detalle |
|---------|---------|
| **Archivo** | `simulador/main.go` (struct `planificadorReal` + método `Ruta`) |
| **Algoritmo** | BFS (Breadth-First Search) sobre grafo dirigido con pesos |
| **Importado de IA** | **No** — N/A |
| **Commit de importación** | N/A |
| **Autores** | Grupo (Cruces, Díaz, Pino) |
| **Descripción** | El planificador recorre `GrafoMina.Tramos` respetando `EnFalla`, calcula pesos con `Tramo.Peso(cargado)` (×1.3 redondeado arriba), y retorna la ruta mínima en ticks. |

---

## 3. Registro de Uso de IA (Log de Consultas)

> **Formato requerido:** Fecha/Semana | Integrante | Tipo de uso | Qué se consultó y cómo ayudó | Enlace a conversación

| # | Fecha / Semana | Integrante | Tipo de Uso | Qué se consultó y cómo ayudó | Enlace Conversación |
|---|----------------|------------|-------------|------------------------------|---------------------|
| 1 | 2026-11-XX / Semana X | [Nombre] | **Explicación de conceptos / Cómo empezar** | Entender la estructura general del enunciado: entidades (camión, pala, stockpile, tramo), concurrencia requerida, y flujo de simulación por ticks. Ayudó a definir las interfaces `Mina` y `Planificador` y la arquitectura de goroutines. | [Link conversación 1] |
| 2 | 2026-11-XX / Semana X | [Nombre] | **Explicación de conceptos / Cómo empezar** | Diseño del protocolo de prevención de deadlock en rampas (reserva atómica en `Pala.Lugares` antes de entrar a rampa sentido "Ida"). Clarificó la diferencia entre protocolo "ingenuo" y "prevencion". | [Link conversación 2] |
| 3 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Error: `sync: negative WaitGroup counter` — diagnóstico de `wg.Add()`/`wg.Done()` desbalanceados en goroutines anidadas. Solución: mover `wg.Add()` antes de lanzar goroutine. | [Link conversación 3] |
| 4 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Race condition detectada con `go run -race` en `TablaEstado` y `TramoSimulacion`. Revisión de protección con `sync.Mutex` en todos los accesos compartidos. | [Link conversación 4] |
| 5 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Deadlock en `camionReal` al esperar en canal `respCarga` sin timeout. Implementación de `esperarRecurso` con timeout en ticks y registro `ESPERA_PROLONGADA`. | [Link conversación 5] |
| 6 | 2026-11-XX / Semana X | [Nombre] | **Explicación de conceptos / Cómo empezar** | Parsing de archivo de mina (formato NODO/TRAMO) y construcción de `GrafoMina` con índices `palas` y `stocks` para el despachador. | [Link conversación 6] |
| 7 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Panic: `send on closed channel` en `Logger.Cerrar()` llamado múltiples veces. Solución: `defer log.Cerrar()` solo en goroutine propietaria y checks de `archivo != nil`. | [Link conversación 7] |
| 8 | 2026-11-XX / Semana X | [Nombre] | **Explicación de conceptos / Cómo empezar** | Distribución lognormal para tiempos de carga (μ=4.41, σ=2.54) y toneladas (μ=331, σ=16.06) según enunciado. Implementación de `lognormal()` usando Box-Muller via `rand.NormFloat64()`. | [Link conversación 8] |
| 9 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Falla temporal en tramos: inyección de `EnFalla=true` en goroutine separada, exclusión en BFS del planificador, y recuperación automática. | [Link conversación 9] |
| 10 | 2026-11-XX / Semana X | [Nombre] | **Interpretación de mensajes de error del compilador / runtime** | Visualizador: snapshot consistente con `maps.Clone` y `append([]string(nil), v...)` bajo un solo lock en `TablaEstado.Instantanea()`. | [Link conversación 10] |

> **Instrucciones para completar:**
> 1. Reemplazar `[Nombre]` con el integrante real (Cruces / Díaz / Pino)
> 2. Reemplazar `2026-11-XX / Semana X` con la fecha y semana real del certamen
> 3. Reemplazar `[Link conversación N]` con el enlace real a la conversación de Claude Code (formato: `https://claude.ai/chat/...` o referencia al transcript)
> 4. Agregar/eliminar filas según consultas reales realizadas
> 5. Mantener solo consultas que encajen en los **dos tipos permitidos**

---

## 4. Resumen de Commits Relacionados con IA

| Commit Hash | Fecha | Mensaje | Relación con IA |
|-------------|-------|---------|-----------------|
| `xxxxxxx` | 2026-11-XX | feat: implementar planificador BFS con fallas | Código escrito por grupo; IA solo ayudó a entender BFS sobre grafo ponderado |
| `xxxxxxx` | 2026-11-XX | fix: deadlock en rampas con protocolo prevencion | IA ayudó a diagnosticar deadlock; solución (reserva atómica) implementada por grupo |
| `xxxxxxx` | 2026-11-XX | fix: race condition en TablaEstado y TramoSimulacion | IA interpretó reporte `-race`; mutexes añadidos por grupo |
| `xxxxxxx` | 2026-11-XX | feat: logging CSV con secuencia global atómica | IA explicó `atomic.AddUint64`; implementación por grupo |

> Completar con hashes reales de `git log --oneline`

---

## 5. Verificación de Cumplimiento

- [x] El uso de IA se limitó a **comprensión del problema/inicio** e **interpretación de errores**
- [x] **No se importó código externo ni generado por IA** para el planificador de rutas
- [x] Todo el código del simulador, su integración y adaptación fue **desarrollado por el grupo**
- [x] Cada consulta en el registro incluye: fecha/semana, integrante, tipo de uso, descripción, enlace
- [x] Los enlaces a conversaciones están disponibles (según exige el enunciado)

---

**Firmas / Aprobación del grupo:**

| Integrante | Firma / Iniciales | Fecha |
|------------|-------------------|-------|
| Cruces | | |
| Díaz | | |
| Pino | | |

---

*Documento generado como base para AI_USAGE.md — completar con datos reales del certamen.*