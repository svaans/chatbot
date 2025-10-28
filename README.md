# Flujo n8n: Asistente Virtual Dorotea

Este documento describe el flujo `Asistente Virtual Dorotea` incluido en [`flows/dorotea_virtual_assistant.json`](../flows/dorotea_virtual_assistant.json). El flujo conecta el canal de mensajería con la base de datos de subastas alojada en Supabase (Postgres) y genera respuestas contextuales mediante un modelo Ollama.

## Descripción general

1. **Entrada Asistente (Webhook)**: expone un endpoint `POST /dorotea-assistant` para recibir los mensajes del cliente, junto con metadatos de sesión y canal.
2. **Normalizar Entrada (Function)**: homogeniza el payload, genera un `sessionId` y agrega una marca de tiempo ISO8601.
3. **Listar Subastas (Postgres)**: ejecuta una consulta parametrizada que devuelve hasta cinco subastas activas o próximas. Requiere la credencial `Supabase Postgres` configurada en n8n con el string de conexión de Supabase.
4. **Agrupar Subastas (Function)**: agrupa las filas devueltas por Postgres en un solo item para simplificar el manejo en nodos posteriores.
5. **Contexto Conversacional (Merge)**: combina el mensaje del usuario con la lista agregada de subastas para mantener ambos contextos disponibles en ejecuciones posteriores.
6. **Construir Prompt (Function)**: formatea la información de las subastas y construye el prompt que se enviará al modelo. Incluye un fallback cuando no hay subastas.
7. **Consultar Ollama (HTTP Request)**: invoca el endpoint `POST http://ollama:11434/api/chat` (ajustable según despliegue) con el modelo `dorotea-llm`. Se recomienda crear en Ollama un modelo derivado con conocimiento de la empresa.
8. **Combinar Respuesta (Merge)**: fusiona la respuesta de Ollama con el contexto original para conservar datos del cliente de cara a los nodos siguientes.
9. **Formatear Respuesta (Function)**: extrae el texto del asistente, mantiene el listado de subastas en metadatos y prepara la respuesta para el cliente.
10. **Responder al Cliente (Respond to Webhook)**: devuelve la respuesta generada al canal de entrada.
11. **Registrar Conversación (Postgres)**: persiste la interacción en la tabla `conversation_logs` de Supabase (crear con columnas `session_id`, `message`, `response`, `channel`, `created_at`). Usa la misma credencial `Supabase Postgres`.

## Campos esperados en el webhook

```json
{
  "message": "¿Qué subastas tienen esta semana?",
  "channel": "whatsapp",
  "session_id": "cliente-123",
  "metadata": {
    "lead_source": "campaña primavera"
  }
}
```

El flujo genera valores por defecto cuando faltan campos:

- `message`: cadena vacía si no llega.
- `channel`: `web` por defecto.
- `session_id`: si no se proporciona, se toma la cabecera `x-session-id` o se genera a partir del `execution.id`.

## Requisitos previos

- **Credenciales**: definir en n8n una credencial Postgres llamada `Supabase Postgres` con los datos del proyecto Supabase.
- **Tabla `conversation_logs`**: crearla en Supabase si aún no existe:

```sql
CREATE TABLE public.conversation_logs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id text NOT NULL,
  message text NOT NULL,
  response text NOT NULL,
  channel text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

- **Modelo Ollama**: levantar Ollama accesible desde n8n (por ejemplo en la misma red Docker). Ajustar el nombre del modelo si es distinto a `dorotea-llm`.

## Personalización

- **Filtrado avanzado**: modificar la consulta SQL en `Listar Subastas` para incorporar filtros por ciudad, rango de precio o estado.
- **Registro adicional**: añadir nodos para enviar métricas a Prometheus, crear tickets en CRM o disparar notificaciones internas.
- **Fallback sin contexto**: si no se desea mostrar la lista de subastas al cliente, ajustar el prompt en `Construir Prompt`.

## Pruebas

1. Importar el JSON en n8n (`Import From File`).
2. Configurar las credenciales de Postgres y probar la conexión.
3. Iniciar el webhook y enviar un `POST` con el cuerpo de ejemplo.
4. Verificar que se obtiene la respuesta de Ollama y que se insertan registros en `conversation_logs`.

## Observabilidad sugerida

- Activar logging de ejecuciones en n8n para monitorear la latencia de cada nodo.
- Extender el flujo con un nodo HTTP adicional que reporte métricas a Prometheus o a un sistema de monitoreo interno.