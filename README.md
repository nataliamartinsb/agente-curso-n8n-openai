# Agente de IA para Hipócrates

## 1. Qué hace el agente

Este workflow atiende consultas de Hipócrates, un emprendimiento de materiales educativos. Recibe mensajes por webhook, mantiene contexto, responde consultas y puede usar Gmail para comunicaciones formales o Slack para avisos internos urgentes; luego clasifica y registra cada ejecución.

## 2. Cómo se invoca

Se invoca mediante una petición `POST` a la URL de producción del nodo Webhook, con encabezado `Content-Type: application/json` y el siguiente cuerpo:

```json
{
  "chatInput": "¿Qué materiales ofrece Hipócrates?"
}
```

El nodo `Validar input` exige que `chatInput` sea texto, contenga entre 3 y 1000 caracteres y no incluya caracteres de control. Los mensajes inválidos reciben una respuesta HTTP 400 sin llegar a OpenAI.

## 3. Manejo de errores

El nodo `AI Agent` realiza hasta 3 intentos, con 2000 ms de espera, y detiene el workflow si todos fallan. En esta versión de n8n, Gmail Tool y Slack Tool no exponen `Retry On Fail`; sus errores se propagan al agente y quedan cubiertos por los reintentos del nodo padre. Los nodos de Google Sheets también tienen 3 intentos.

Los errores no quedan en silencio: el workflow separado `Manejador de errores - Hipócrates` utiliza `Error Trigger` y envía a Slack el workflow, nodo fallido, mensaje, identificador, fecha y enlace de la ejecución. Las evidencias están en `evidencias/manejo-errores.png` y `evidencias/workflow-produccion.png`.

## 4. Monitoreo

La opción `Save execution progress` está habilitada. Cada rama registra en Google Sheets el timestamp, ID de sesión, input, output, tokens, duración, resultado y rama; cuando n8n no expone el consumo real del modelo, el campo de tokens guarda una estimación basada en la longitud del input y output.

El flujo envía una alerta a Slack si una ejecución supera 15.000 ms o 500 tokens. Además, cada error genera una alerta inmediata, una política más estricta que esperar a acumular una tasa mínima de fallos.

## 5. Seguridad

Todas las credenciales están guardadas mediante el sistema Credentials de n8n y no existen API keys, tokens ni contraseñas escritos en el workflow. Gmail se usa únicamente con la operación de envío; Slack solo publica mensajes en el canal configurado; y el flujo no incorpora operaciones de lectura, borrado ni administración que no necesita.

El webhook valida estructura, longitud y caracteres antes de llamar al agente. Las consultas inválidas se rechazan, y las acciones ambiguas o sin destinatario, canal o contenido suficiente no deben ejecutar herramientas.

## 6. Mitigación de alucinaciones

El System Message ordena no inventar precios, productos, stock, promociones, pedidos, direcciones, correos ni políticas, y exige responder que no existe información suficiente cuando corresponda. El modelo usa temperatura `0.2` para priorizar consistencia y precisión.

El nodo `Validar salida IA` bloquea respuestas vacías, demasiado cortas o con frases de incertidumbre. Esas respuestas toman un fallback explícito de revisión, se registran en auditoría y no continúan hacia las ramas normales de negocio.

## Archivos principales

- `workflow.json`: workflow principal endurecido para producción.
- `manejador-errores.json`: workflow separado de errores y notificación.
- `evidencias/workflow-produccion.png`: vista general del flujo principal.
- `evidencias/manejo-errores.png`: subflujo de captura y aviso de errores.
