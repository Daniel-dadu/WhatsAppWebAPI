# Azure Function - Web API WhatsApp

This Azure Function app exposes several HTTP endpoints used by the chatbot system.

Endpoints
---------

1) `POST /login`
- Description: Simple login endpoint for testing.
- Body (JSON):
  - `username` (string)
  - `password` (string)
- Responses:
  - 200: JSON with a signed JWT access token
  - 401: "Login failed"

Sample responses:

Success (200):
```json
{
  "message": "Login successful",
  "access_token": "<JWT_TOKEN_HERE>",
  "token_type": "Bearer",
  "expires_in": 28800
}
```

Failure (401):
```json
"Login failed"
```

2) `POST /conversation-mode`
- Description: Cambia el modo de conversación (bot/agente) para un lead específico.
- Body (JSON):
  - `wa_id` (string) - identificador del lead (sin el prefijo `conv_`)
  - `mode` (string) - "bot" o "agente"
- Responses:
  - 200: JSON with success message and timestamp
  - 400: Missing/invalid parameters
  - 404: Conversation not found
  - 500: Internal server error

Sample success response (200):
```json
{
  "success": true,
  "message": "Conversation mode updated successfully",
  "new_mode": "agente",
  "timestamp": "2025-08-28T12:34:56Z"
}
```

3) `POST /send-agent-message`
- Description: Envía un mensaje del agente al lead invocando la función del chatbot.
- Body (JSON):
  - `wa_id` (string)
  - `message` (string)
- Responses:
  - 200: JSON confirming message was sent
  - 400: Missing parameters
  - 500: Error communicating with chatbot

Sample success response (200):
```json
{
  "success": true,
  "message": "Agent message sent successfully",
  "wa_id": "lead_98765",
  "message_sent": "Hola, te contacto desde soporte",
  "conversation_mode": "agente",
  "timestamp": "2025-08-28T12:35:00Z"
}
```

4) `POST /get-conversation`
- Description: Obtiene el estado completo de una conversación. Es de tipo POST y recibe `wa_id` en el body JSON.
- Body (JSON):
  - `wa_id` (string) - identificador del lead (sin el prefijo `conv_`)
- Responses:
  - 200: JSON with conversation details
  - 400: Missing `wa_id` or invalid JSON
  - 404: Conversation not found
  - 500: Internal server error

Sample success response (200):
```json
{
  "wa_id": "lead_98765",
  "conversation_mode": "bot",
  "lead_info": {
    "nombre": "María García López",
    "telefono": "521234567890"
  },
  "messages": [
    {
      "from": "user",
      "text": "Hola",
      "ts": "2025-08-24T18:00:00Z"
    }
  ],
  "completed": false,
  "updated_at": "2025-08-24T18:15:00Z"
}
```

5) `GET /conversations/recent`
- Description: Devuelve las últimas 10 conversaciones ordenadas por `updated_at` desc.
- Responses:
  - 200: JSON array, donde cada elemento tiene la siguiente forma:
    ```json
    {
      "id": "conv_12345",
      "lead_id": "lead_98765",
      "canal": "whatsapp",
      "created_at": "2025-08-24T18:00:00Z",
      "updated_at": "2025-08-24T18:15:00Z",
      "state": {
        "nombre": "María",
        "nombre_completo": "María García López",
        "telefono": "521234567890",
        "completed": false
      },
      "conversation_mode": "agente",
      "asignado_asesor": "asesor_ventas_001"
    }
    ```
  - 500: Internal server error
```

Notes
-----
- La conexión a Cosmos DB actualmente está configurada para pruebas locales en `get_cosmos_container()`; reemplázala por las variables de entorno `COSMOS_CONNECTION_STRING`, `COSMOS_DB_NAME` y `COSMOS_CONTAINER_NAME` para producción.
- La función `send-agent-message` actualmente usa una URL de chatbot codificada — cámbiala por una configuración a través de variables de entorno antes de desplegar.
- Autenticación: la Function App ahora emite un JWT al iniciar sesión con éxito. La secret del JWT debe configurarse en App Settings como `JWT_SECRET`.

Autenticación / notas de uso
---------------------------
- Tras un `POST /login` exitoso, la respuesta contiene `access_token` (un JWT). El frontend debe almacenar ese token (por ejemplo en `localStorage`) y enviarlo en las peticiones siguientes en el header `Authorization` así:

```
Authorization: Bearer <token>
```

Ejemplo de uso en cliente (browser JS):

```js
// Tras recibir la respuesta del login en `data`
localStorage.setItem('access_token', data.access_token);

// Más tarde, enviando una petición protegida
fetch('/conversation-mode', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ' + localStorage.getItem('access_token')
  },
  body: JSON.stringify({ wa_id: 'lead_123', mode: 'agente' })
});
```

Si el JWT falta, es inválido o ha expirado, los endpoints protegidos devolverán 401 Unauthorized.

Cómo ejecutar localmente
-----------------------
- Instala los requisitos de Python listados en `requirements.txt` dentro de tu entorno virtual.
- Inicia el host de Functions (hay una tarea de VS Code incluida): `func host start`.
- Usa `test.http` o `curl` para probar los endpoints.