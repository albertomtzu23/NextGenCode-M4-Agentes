# 0005. Id por petición, log JSON y métricas por proceso

## Contexto

Hay que poder responder qué pasó con cualquier petición sin guardar estado entre peticiones (ADR 0001) y sin
exponer datos de las personas. Las pruebas deben ser deterministas aunque el log lleve hora y latencia.

## Decisión

`observability.py` concentra tres piezas que `create_app()` recibe inyectadas: un reloj, un `JsonLogger` que
escribe una línea JSON por evento en la salida estándar (`request`, `agent_run`, `audit`, `error`) y `Metrics`
en memoria del proceso. Un middleware toma `X-Request-ID` o genera un UUID y lo propaga a log, respuesta y
trazas. Los eventos llevan conteos, nunca el texto de la pregunta o la respuesta, ni mensajes de error
completos. LangSmith se importa solo si `LANGSMITH_TRACING=true` y hay clave.

## Consecuencias

- `/metrics` describe solo su proceso; para el total se suman las copias. No contradice el ADR 0001: no es
  estado de conversación.
- Las pruebas fijan el reloj y capturan el log en memoria.
- Los eventos de costo de la sesión 3 (`model`, `route`, `cost_usd`) se agregan a `agent_run` sin cambiar el
  formato.
