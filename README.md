# IA Support Triage & Human-in-the-Loop Workflow

Un flujo de trabajo automatizado construido en **n8n** para la gestión inteligente de tickets de soporte técnico. El sistema procesa correos entrantes, utiliza Inteligencia Artificial (**Google Gemini**) para clasificar la severidad y redactar respuestas, gestiona bases de datos de seguimiento y permite la aprobación humana (**Human-in-the-Loop**) vía **Telegram** antes de responder al cliente.

---

## 📐 Arquitectura del Workflow

```
[Gmail Trigger] ➔ [Filtro Anti-Bucles] ➔ [Airtable: Registrar Ticket]
                                                    │
                                         [Motor IA (Gemini)]
                                                    │
                                     [Switch: Human-in-the-Loop]
                                      ┌─────────────┴─────────────┐
                         (Requiere Aprobación)            (Soporte Estándar)
                                      │                           │
                         [Telegram: Notificación]        [Airtable: Registrar Error]
                                      │
                         [Wait: Esperar Clic]
                                      │
                        [Gmail: Responder en Thread]
                                      │
                        [Airtable: Marcar Resuelto]
```

---

## 🛠️ Tecnologías y Herramientas

* **n8n**: Orquestador principal de la automatización.
* **Gmail API**: Disparador de eventos de correo e hilo de respuestas.
* **Google Gemini (`gemini-3.1-flash-lite`)**: Modelo de lenguaje (LLM) para análisis contextual y generación de respuestas.
* **LangChain Integration (n8n)**: Módulos de *Chain LLM* y *Structured Output Parser* para forzar salidas JSON estructuradas.
* **Telegram Bot API**: Canal para alertas críticas e interacción *Human-in-the-Loop*.
* **Airtable / n8n Data Tables**: Base de datos para registro de estados, severidad, logs de errores y trazabilidad de tickets.

---

## ⚙️ Componentes y Lógica del Flujo

1. **Gmail Trigger (`subject:"TICKETPRUEBA:" is:unread`)**:
   Monitorea la bandeja de entrada buscando correos no leídos que contengan el patrón de asunto especificado.
2. **Filtro Anti-Bucles**:
   Evita bucles infinitos validando que el remitente no contenga `no-reply` y que el asunto no contenga `Re:`.
3. **Airtable: Registrar Ticket**:
   Crea una entrada inicial en la base de datos con estado **Pendiente**, registrando el asunto y mensaje original.
4. **Motor de IA (Clasificador LLM + Structured Parser)**:
   Procesa el mensaje mediante Gemini y garantiza un formato JSON estricto:
   ```json
   {
     "severidad": "Baja" | "Media" | "Alta" | "Critica",
     "respuesta_sugerida": "Texto de la respuesta..."
   }
   ```
5. **Switch: ¿Human-in-the-Loop?**:
   Evalúa la severidad devuelta por la IA. Si es **Alta** o **Critica**, enruta hacia la rama de aprobación. Si falla o genera un error, lo registra en la tabla de errores.
6. **Telegram & Wait**:
   Envía una alerta estructurada con la respuesta sugerida al bot de Telegram (`Valent_ia_bot`). Pausa la ejecución hasta que el agente presione el enlace de aprobación (*Webhook*).
7. **Gmail: Responder en Thread**:
   Responde directamente en el hilo original del correo del cliente utilizando la respuesta validada.
8. **Airtable: Marcar Resuelto**:
   Actualiza la fila del ticket con la severidad final, la respuesta enviada y el estado **Resuelto**.

---

## 🚀 Instalación y Configuración

### Prerrequisitos
1. Instancia de **n8n** activa (Cloud o Self-hosted).
2. Credenciales configuradas en n8n:
   * **Gmail OAuth2 API**
   * **Google Gemini (PaLM/Gemini) API Key**
   * **Telegram Bot API Token**
3. Base de datos configurada (Airtable o Data Tables internas) con las siguientes columnas:
   * `Asunto` (String)
   * `Mensaje_Original` (String)
   * `Estado` (String)
   * `Severidad` (String)
   * `Respuesta_Sugerida` (String)
   * `Error_Log` (String)

### Pasos de Importación
1. Clona este repositorio o copia el contenido del archivo JSON del workflow.
2. En tu interfaz de n8n, crea un nuevo flujo y selecciona **Import from JSON** (o usa `Ctrl + V`).
3. Asigna las credenciales correspondientes a cada nodo (Gmail, Gemini, Telegram).
4. Activa el workflow.

---

## 📑 Código JSON del Workflow

<details>
<summary>Ver JSON completo de n8n</summary>

```json
{
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [{}]
        },
        "mode": "everyMinute",
        "simple": false,
        "filters": {},
        "q": "subject:\"TICKETPRUEBA:\" is:unread",
        "readStatus": "unread",
        "options": {}
      },
      "id": "1361bfa4-cc23-4642-9b60-56105cbec28c",
      "name": "Gmail Trigger (Nuevos Tickets)",
      "type": "n8n-nodes-base.gmailTrigger",
      "typeVersion": 1,
      "position": [1, -464]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.from.value[0].address }}",
              "operation": "notContains",
              "value2": "no-reply"
            },
            {
              "value1": "={{ $json.subject }}",
              "operation": "notContains",
              "value2": "Re:"
            }
          ]
        }
      },
      "id": "c0977c02-33f8-4e8b-abc6-9dde3a7a1762",
      "name": "Filtro Anti-Bucles",
      "type": "n8n-nodes-base.filter",
      "typeVersion": 1,
      "position": [-256, 304]
    },
    {
      "parameters": {
        "prompt": "=Analiza el siguiente correo de soporte técnico:\n\nAsunto: {{ $json.Asunto }}\nMensaje: {{ $json.Mensaje_Original }}\n\nResponde strictly en formato JSON válido con la siguiente estructura:\n{\n \"severidad\": \"Baja\" | \"Media\" | \"Alta\" | \"Critica\",\n \"respuesta_sugerida\": \"Redacta una respuesta profesional, empática y técnica para el cliente.\"\n}",
        "messages": {}
      },
      "id": "75b6da3c-706e-4ad5-bcb9-57f8f321a0c6",
      "name": "Motor de IA (Clasificador)",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [192, 304],
      "onError": "continueErrorOutput"
    },
    {
      "parameters": {
        "rules": {
          "values": [
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 1
                },
                "conditions": [
                  {
                    "leftValue": "={{ $json.output.severidad }}",
                    "rightValue": "Alta",
                    "operator": { "type": "string", "operation": "equals" }
                  },
                  {
                    "leftValue": "={{ $json.output.severidad }}",
                    "rightValue": "Critica",
                    "operator": { "type": "string", "operation": "equals" }
                  }
                ],
                "combinator": "or"
              },
              "renameOutput": true,
              "outputKey": "Requiere_Aprobacion"
            }
          ]
        }
      },
      "id": "b12b7903-7e30-46c0-ab40-91959d1911f6",
      "name": "Switch: ¿Human-in-the-Loop?",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3,
      "position": [640, 288]
    },
    {
      "parameters": {
        "resume": "webhook",
        "options": {}
      },
      "id": "d3216d52-f26f-4384-a832-89975ccdc3d0",
      "name": "Wait (Esperar Clic en Telegram)",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [1088, 192]
    },
    {
      "parameters": {
        "chatId": "Valent_ia_bot",
        "text": "=⚠️ *ALERTA DE TICKET DE ALTA SEVERIDAD*\n\n*Asunto:* {{ $('Airtable: Registrar Ticket').item.json.Asunto }}\n*Severidad Detectada:* {{ $json.text.severidad }}\n\n*Respuesta Sugerida por IA:*\n_{{ $json.text.respuesta_sugerida }}_\n\n👉 *Presiona para aprobar el envío al cliente:*",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "ffe1cd1c-a091-491f-87de-c2e12b7b48cf",
      "name": "Telegram: Enviar Notificación",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.1,
      "position": [864, 192]
    },
    {
      "parameters": {
        "sendTo": "={{ $('Gmail Trigger (Nuevos Tickets)').item.json.from.value[0].address }}",
        "subject": "=Re: {{ $('Gmail Trigger (Nuevos Tickets)').item.json.subject }}",
        "message": "={{ $('Motor de IA (Clasificador)').item.json.text.respuesta_sugerida }}",
        "options": {}
      },
      "id": "e641dab5-75dd-4869-9476-6d4ee5019a6c",
      "name": "Gmail: Responder en Thread",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [1296, 288]
    },
    {
      "parameters": {
        "modelName": "models/gemini-3.1-flash-lite",
        "options": {}
      },
      "id": "f455145e-232f-486e-b1b0-168ec800010d",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1.1,
      "position": [192, 544]
    },
    {
      "parameters": {
        "dataTableId": {
          "value": "W6zQAeA3aodPQIni"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Asunto": "={{ $json.subject }}",
            "Mensaje_Original": "={{ $json.text }}",
            "Estado": "Pendiente"
          }
        }
      },
      "id": "f31ebc31-b594-429c-a55b-bd4d76f2ff5e",
      "name": "Airtable: Registrar Ticket",
      "type": "n8n-nodes-base.dataTable",
      "typeVersion": 1.1,
      "position": [-32, 304]
    },
    {
      "parameters": {
        "operation": "upsert",
        "dataTableId": {
          "value": "W6zQAeA3aodPQIni"
        },
        "filters": {
          "conditions": [
            {
              "keyValue": "={{ $('Airtable: Registrar Ticket').item.json.id }}"
            }
          ]
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Estado": "Error",
            "Error_Log": "={{ $json.error.message }}"
          }
        }
      },
      "id": "4ae02857-fd87-4724-b68f-bc1c30ebf506",
      "name": "Airtable: Registrar Error",
      "type": "n8n-nodes-base.dataTable",
      "typeVersion": 1.1,
      "position": [512, 480]
    },
    {
      "parameters": {
        "operation": "update",
        "dataTableId": {
          "value": "W6zQAeA3aodPQIni"
        },
        "filters": {
          "conditions": [
            {
              "keyValue": "={{ $('Airtable: Registrar Ticket').item.json.id }}"
            }
          ]
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Severidad": "={{ $('Motor de IA (Clasificador)').item.json.text.severidad }}",
            "Respuesta_Sugerida": "={{ $('Motor de IA (Clasificador)').item.json.text.respuesta_sugerida }}"
          }
        }
      },
      "id": "310641f0-d85e-4a1a-9cbe-0fc402d129e5",
      "name": "Airtable: Marcar Resuelto",
      "type": "n8n-nodes-base.dataTable",
      "typeVersion": 1.1,
      "position": [1520, 288]
    },
    {
      "parameters": {
        "jsonSchemaExample": "{\n \"severidad\": \"Baja\",\n \"respuesta_sugerida\": \"Redacta una respuesta profesional, empática y técnica para el cliente.\"\n}"
      },
      "id": "a79cfd08-68d1-4080-8c99-729c213a2e58",
      "name": "Structured Output Parser",
      "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
      "typeVersion": 1.3,
      "position": [336, 512]
    }
  ]
}
```
</details>