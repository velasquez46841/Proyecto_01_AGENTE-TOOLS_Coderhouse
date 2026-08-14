# Aura — Asistente Personal con IA (n8n)

Aura es un agente de IA conversacional construido en **n8n**, que funciona como asistente personal para consultar el **clima**, las **noticias del día** y la **agenda de calendario**, además de poder **crear nuevos eventos** por pedido del usuario.

## 🧠 ¿Qué hace Aura?

| Función | Descripción |
|---|---|
| ☀️ Clima | Da el pronóstico actual o por rango de fechas de cualquier ciudad, infiriendo automáticamente la ubicación. |
| 📰 Noticias | Resume titulares de distintos feeds RSS (mundo, tecnología, salud, gastronomía, etc.) según el tema consultado. |
| 📅 Ver agenda | Consulta y lista los eventos ya agendados en Google Calendar para el día (u otro rango) que se le pida. |
| ➕ Agendar eventos | Crea nuevos eventos en Google Calendar a partir de una instrucción en lenguaje natural (ej. "agendame una reunión mañana a las 15hs"). |
| 📧 Notificación | Al finalizar cada respuesta, envía un resumen por Gmail. |

## 🏗️ Arquitectura del workflow

```
When chat message received (Trigger)
        │
        ▼
  Agente Inmobiliario (AI Agent, modo Tools Agent)
        │  ├── Connect Gemini (Chat Model — Google Gemini 2.5 Flash)
        │  ├── Conversation Memory (buffer de 12 mensajes)
        │  ├── Get Weather (HTTP Request Tool → Open-Meteo API)
        │  ├── Get News (RSS Feed Read Tool)
        │  ├── Ver Eventos de Calendario (Google Calendar Tool — Get Many)
        │  └── Agendar Evento en Calendario (Google Calendar Tool — Create)
        ▼
  Send Email1 (Gmail — envía el resumen de la respuesta)
```

## 🛠️ Tecnologías y servicios usados

- **n8n** — orquestación del workflow y del agente.
- **Google Gemini 2.5 Flash** — modelo de lenguaje (vía credencial `googlePalmApi`).
- **Open-Meteo API** — pronóstico del clima (sin necesidad de API key).
- **RSS Feed Read** — lectura de noticias desde múltiples fuentes (BBC, Al Jazeera, CNN, TechCrunch, Hacker News, entre otras).
- **Google Calendar API** — lectura y creación de eventos (OAuth2).
- **Gmail API** — envío de notificaciones por correo (OAuth2).

## ⚙️ Configuración

1. Importar `Aura_Asistente_Personal.json` en tu instancia de n8n (**Workflows → Import from File**).
2. Configurar las credenciales de cada nodo:
   - **Connect Gemini** → credencial de Google Gemini (API key).
   - **Ver Eventos de Calendario** / **Agendar Evento en Calendario** → credencial OAuth2 de Google Calendar, y seleccionar el calendario deseado en el campo `calendar`.
   - **Send Email1** → credencial OAuth2 de Gmail, y reemplazar el campo `sendTo` por tu propio email.
3. Activar el workflow y abrir el chat público generado por el nodo **When chat message received**.

## 💬 Ejemplos de uso

- *"¿Cómo está el clima en Buenos Aires hoy?"*
- *"Dame las últimas noticias de tecnología."*
- *"¿Qué tengo agendado para hoy?"*
- *"Agendame una reunión con el equipo mañana a las 15hs."*

## 🧩 Diseño del System Message

El prompt del agente (nodo **Agente Inmobiliario**) define:
- **Rol y personalidad**: cercana, clara, directa, sin lenguaje inclusivo.
- **Reglas de uso de cada herramienta**, evitando que el modelo invente datos o confunda "consultar" con "crear" eventos.
- **Formato de salida** breve y conversacional, adaptado a cada tipo de respuesta (clima, noticias, eventos).
- **Límite de iteraciones** (`maxIterations: 8`) para acotar el razonamiento del agente por respuesta.

## ✅ Cumplimiento de la consigna

| # | Requisito | Estado |
|---|---|---|
| 1 | AI Agent en modo Tools Agent | ✅ |
| 2 | Chat Model nativo conectado | ✅ |
| 3 | Máximo de iteraciones (5-10) | ✅ (`maxIterations: 8`) |
| 4 | System Message modular, con exclusión de lenguaje inclusivo | ✅ |
| 5 | Conector nativo de Workspace/productividad (Google Calendar) | ✅ |
| 6 | Descripción semántica extensa en las herramientas acopladas | ✅ |
| 7 | Sin bloques rígidos de decisión antes del agente | ✅ |
| 8 | Nodo final de notificación (Gmail) | ✅ |
| 9 | Prueba manual vía Execute Workflow | ✅ |

---
*Proyecto desarrollado como práctica de agentes de IA con n8n.*
