# Registro de Uso de Inteligencia Artificial

**Proyecto:** Simulador de Operaciones Mineras (ICI425 - Cruces, Díaz, Pino)
**Fecha de creación:** 2026-11-21

---

## 1. Alcance del Uso de IA

El uso de herramientas de inteligencia artificial en este proyecto se limitó **exclusivamente** a los siguientes casos permitidos según el enunciado:

| Caso Permitido | Descripción |
|----------------|-------------|
| **Comprensión del problema / Inicio del trabajo** | Análisis de requerimientos, entendimiento de la arquitectura propuesta, definición de estructuras de datos iniciales, planificación de la concurrencia |
| **Interpretación de errores de compilación o ejecución** | Diagnóstico de errores del compilador Go, deadlocks en tiempo de ejecución, race conditions, panics, lógica de sincronización |

**Lo que NO se hizo con IA:**
-  El código del simulador (goroutines, canales, mutex, lógica de negocio) fue escrito íntegramente por el grupo
-  La integración entre módulos (main, despachador, palas, stockpiles, tramos, camiones) fue desarrollada por el grupo
-  La adaptación de la simulación a los requerimientos específicos (protocolos ingenuo/prevención, fallas, logging CSV) fue implementada por el grupo

---

## 2. Planificador de Rutas

> **Declaración:** No se importó código externo ni generado por IA para el planificador de rutas; fue implementado por el grupo.

| Aspecto | Detalle |
|---------|---------|
| **Archivo** | `simulador/main.go` (struct `planificadorReal` + método `Ruta`) |
| **Algoritmo** | BFS (Breadth-First Search) sobre grafo dirigido con pesos |
| **Autores** | Grupo (Cruces, Díaz, Pino) |
| **Descripción** | El planificador recorre `GrafoMina.Tramos` respetando `EnFalla`, calcula pesos con `Tramo.Peso(cargado)` (×1.3 redondeado arriba), y retorna la ruta mínima en ticks. |

---

## 3. Registro de Uso de IA (Log de Consultas)

> **Formato requerido:** Fecha/Semana | Integrante | Tipo de uso | Qué se consultó y cómo ayudó

| # | Fecha / Semana | Integrante | Tipo de Uso | Qué se consultó y cómo ayudó |
|---|----------------|------------|-------------|------------------------------|
| 1 | 2026-10-22 / Semana 0 | [Rodrigo Pino] | **Explicación de conceptos / Cómo empezar** | Entender la estructura general del enunciado: entidades (camión, pala, stockpile, tramo), concurrencia requerida, y flujo de simulación por ticks y la arquitectura de goroutines. |
| 2 | 2026-10-26 / Semana 0-1 | [Rodrigo Pino] | **Explicación de Lenguaje Go** | Para comprender bien las diferencias que tiene relacionado a los demás lenguajes anteriormente vistos en la universidad, como funciona o interactua con el tema de los hilos y mini curso de aprendizaje medio con ejercicios de programación en Go|
| 3 | 2026-11-12 / Semana 2-3 | [Rodrigo Pino] | **Explicación de Codigo de los demas integrantes** | Para entender bien lo que se realizo los demas integrantes para poder realizar la integracion de los componentes para poder llevar a cabo la ejecucion correcta del certamen 2 |

---
