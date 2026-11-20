# Seguridad y controles

La tabla completa, con el archivo y la prueba de cada regla, está en
[`hr-assistant/doc/compliance-checklist.md`](../hr-assistant/doc/compliance-checklist.md). Este es el resumen
por requisito del reto.

| # | Control | Cómo está implementado | Dónde |
| --- | --- | --- | --- |
| 1 | Validación de entradas | Pydantic valida el cuerpo de `/ask`: `question` tiene un largo máximo y el historial tiene roles y forma fijos. Si algo no cumple, la API responde 422 | `app/main.py` (`AskRequest`, `HistoryMessage`) |
| 2 | Validación de salidas | El modelo debe contestar con el protocolo JSON del agente. Lo que no se puede leer, o una herramienta desconocida o con argumentos inválidos, se rechaza (`ToolCallError`) y no llega al usuario. La respuesta sale con el esquema `AskResponse`, y la página pinta el texto con `textContent`, sin HTML | `app/agent.py`, `app/main.py`, `app/static/index.html` |
| 3 | Gestión de secretos | Ningún secreto está en el código. Se leen de variables de entorno, `.env` está en `.gitignore` y Render los guarda con `sync: false`. La configuración no los muestra en `repr` y los errores solo dicen la posición y el largo | `app/config.py`, `app/auth.py`, `render.yaml` |
| 4 | Autenticación por cliente | `X-API-Key` por cliente, comparada con `hmac.compare_digest` contra todas las claves. Sin clave válida, la API responde 401 | `app/auth.py` |
| 5 | Límite de peticiones | Ventana de 60 s por cliente. Al pasarse, la API responde 429 con `Retry-After` | `app/ratelimit.py` |
| 6 | Registro de auditoría | El evento `audit` guarda `request_id`, `client`, la herramienta y sus argumentos; cada petición deja también su evento `request` | `app/main.py`, `app/agent.py` |
| 7 | Protección de información sensible | La pregunta no se guarda en los logs, solo su largo. En producción, las trazas de LangSmith llevan obligatoriamente `HIDE_INPUTS` y `HIDE_OUTPUTS`. Las claves nunca se registran | `app/observability.py`, `app/compliance.py` |
| 8 | Mínimo privilegio | Las herramientas del agente son de solo lectura (`get_employee`, `search_by_department`). El contenedor corre con el usuario `appuser`, que no es root. El token de Cloudflare solo tiene permiso de Workers AI. En la fábrica, cada subagente tiene sus herramientas restringidas y un hook bloquea la lectura de `.env` | `app/directory.py`, `Dockerfile`, `.claude/` |
| 9 | Compliance al arrancar | Con `ENVIRONMENT=production`, el servicio no arranca si usa el modelo simulado, si no tiene claves, si una clave tiene menos de 16 caracteres o si las trazas no ocultan datos | `app/compliance.py` |
| 10 | Errores sin filtrar detalles | Si el proveedor falla, la API responde 502 "model unavailable" y solo registra el tipo de error, nunca el cuerpo ni el token | `app/main.py`, `app/llm.py` |

## Detección de PII

La protección actual es estructural:

- El agente solo puede leer el directorio. No escribe ni consulta nada fuera de él.
- El texto de la pregunta y el de la respuesta no queda ni en el log ni en la traza.
- Solo los clientes autenticados pueden preguntar.

Falta un detector que revise el contenido de la pregunta y de la respuesta y enmascare datos como RFC, CURP,
correo, teléfono o tarjeta antes de llamar al modelo. Está planeado como la siguiente mejora, con expresiones
regulares en `app/pii.py` y un evento `pii_detected` en la auditoría.

## Modelo de amenazas resumido

| Riesgo | Mitigación |
| --- | --- |
| Robo de la clave de la API | Clave por cliente y revocable; límite de peticiones; la página la guarda en `sessionStorage`, nunca en `localStorage` |
| Prompt injection que intente sacar datos | Las herramientas son de solo lectura y con argumentos validados, y el bucle tiene tope de pasos. Lo peor que puede pasar es leer el directorio, que ya es visible para el cliente autenticado |
| Fuga de secretos en logs o trazas | No se muestran en `repr`, los errores solo dicen el largo, y compliance obliga a ocultar el contenido en las trazas |
| Publicar código roto | El CI corre style, unit, e2e y docker, y Render solo publica con `checksPass` |
