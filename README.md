# claude-sdk-dev-flow

Orquestador de Claude Code con observabilidad OpenTelemetry: un harness propio, construido sobre el Claude Agent SDK, que ejecuta de forma determinística un flujo de orquestador + subagentes especializados, y expone esa ejecución como trazas/métricas/logs en una plataforma de observabilidad.

## Objetivos

1. **Ejecución determinística**: el orden de llamada a cada subagente (quién, con qué input, qué hacer con su output) vive en código propio, no en el criterio del modelo turno a turno.
2. **Observabilidad end-to-end**: instrumentar cada sesión con OpenTelemetry (trazas, métricas, logs) para ver origen/destino de cada tarea, jerarquía de llamadas, tiempos y costos.

## Decisión de arquitectura

Se evaluaron dos opciones de harness:

- **Dynamic Workflows** (nativo de Claude Code): rápido de adoptar, pero el script de orquestación lo sigue generando el modelo en el momento — no garantiza cumplimiento real del flujo.
- **Claude Agent SDK** (harness propio, Python/TypeScript): la secuencia de pasos se implementa como control de flujo real (`if`/`for`) en código propio; el modelo solo se invoca para ejecutar cada paso puntual. Da control determinístico y tiene instrumentación OpenTelemetry nativa (spans automáticos por conversación, turno, tool call y subagente, con jerarquía padre-hijo).

**Elegido: Claude Agent SDK** como núcleo del harness. Dynamic Workflows queda como opción secundaria para prototipar el flujo sin infraestructura propia.

## Flujo de referencia (v1 aprobada)

```
Contexto de Sesión → Clasificador de Intención → Orquestador (rutea)
  └─ Rama código: Planificador → Challenger (revisa plan) → loop [Coder ↔ Tester] por subtarea
     → Reviewer → Documentador → Challenger (control final) → Contexto de Sesión (cierra)
  └─ Ramas cortas (documentación / tarea de gestión / requerimientos nuevos): agente directo → Contexto de Sesión (cierra)
```

Subagentes: Contexto de Sesión, Clasificador de Intención, Planificador, Challenger, Coder, Tester, Reviewer, Documentador, Gestor de Tareas, Analista de Requerimientos. Cada uno recibe y devuelve un **objeto estandarizado de delegación** (trace id, span id, parent span id, agente, paso del flujo, complejidad, modelo asignado, estado, input/output resumido, artefactos, timestamps) — ese objeto es a la vez el contrato de ejecución del harness y la fuente de los spans en la plataforma de observabilidad.

La complejidad de cada subtarea (baja/media/alta) determina el modelo asignado (haiku/sonnet/opus) para evitar sobre-exigir un modelo grande en tareas simples.

## Capas de telemetría

Claude Code / Agent SDK → OpenTelemetry Collector local → backend de visualización (SigNoz o Langfuse). Variables clave: `CLAUDE_CODE_ENABLE_TELEMETRY`, `OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

## Estado actual

Proyecto en etapa inicial (solo scaffolding del repo). Próximos pasos según la spec: habilitar telemetría nativa, levantar el colector OpenTelemetry local, conectar backend de visualización, y construir el harness con el Claude Agent SDK.

Spec completa (fuente de este resumen): documento de proyecto en Claude Desktop, "Spec Orquestador Claude Code con Observabilidad OpenTelemetry" (24-09-2026).
