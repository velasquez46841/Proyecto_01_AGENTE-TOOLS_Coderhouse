# Neblina Forest — Sistema Multi-Agente

Sistema de agentes de IA construido en **n8n** para Neblina Forest, agencia de turismo especializada en observación de aves. El proyecto nació como ejercicio integrador de un curso (Coderhouse) y creció módulo a módulo hasta convertirse en un sistema real de atención de correo, ventas, memoria conversacional y sincronización con herramientas de negocio (CRM, Gmail, Slack).

Autor: Richard Velasquez Garcia — velasquez46841@estudiantes.untref.edu.ar

---

## 1. Arquitectura general

El sistema sigue un patrón **Manager–Worker**: un workflow central clasifica cada correo entrante y delega el trabajo pesado a workflows especializados ("Workers"), cada uno con un contrato de entrada/salida fijo y su propia lógica de validación. Un segundo eje, independiente del correo, es el **Asesor por Telegram**: un asistente conversacional para uso interno del equipo, con memoria persistente entre sesiones.

```
                    ┌─────────────────────────────┐
                    │  checkpoint4 (Manager 2.0)   │
                    │  Trigger: Gmail (real)       │
                    └──────────────┬──────────────┘
                                   │
                    ① IF ¿Es auto-reply? ──Sí──▶ (corta el bucle)
                                   │ No
                    AI Agent: Clasificar intención
                    (VUELOS · COMPRA · OTROS · REVISION)
                                   │
        ┌──────────────┬──────────┴──────────┬──────────────┐
        ▼              ▼                     ▼              ▼
   CH - Worker    ② Look up/Create      CH - Worker      Escala a
   de Vuelos      en HubSpot (CRM)      Otros            revisión humana
        │         antes de clasificar        │
        │         armado/personalizado       │
        │              │                     ▼
        │        CH - Worker · Clasificador  Aviso a asesor
        │        de Compra                   (si importancia alta)
        │              │
        │    ┌─────────┴─────────┐
        │    ▼ personalizado     ▼ armado
        │  Aviso interno    Leer catálogo (Google Sheets)
        │                   → Filtrar por país (Code, sin IA)
        │                   → Armar HTML con tabla dinámica
        │                   → ③ Gmail Create Draft (HITL)
        │                   → ④ Set limpio → Slack (aviso al equipo)
        ▼
   Registrar en Airtable (Vuelos), vinculado al Cliente
```

En paralelo, **CH - Asesor Telegram** es un chatbot para que el equipo humano consulte datos de clientes ya cargados en Airtable, con memoria de conversación y resúmenes automáticos (Módulo 3), y **CH - Confirmar Compra NF** atiende los formularios web de alta de cliente y confirmación de compra.

---

## 2. El Manager (`checkpoint4_velasquez_garcia_richard.json`)

Es el punto de entrada real del sistema: un **Gmail Trigger** que revisa la casilla de soporte cada un minuto. Este archivo es la entrega del **Checkpoint de Integraciones Avanzadas** y es completamente autocontenido (no depende de que otros workflows existan para poder importarse y evaluarse).

**Clasificación de intención** — un AI Agent (Google Gemini) clasifica cada correo entrante en una taxonomía cerrada de 4 categorías:

| Categoría | Cuándo aplica |
|---|---|
| `VUELOS` | El remitente transmite datos concretos de su propio itinerario aéreo |
| `COMPRA` | Interés (aunque sea general) en contratar o cotizar un tour |
| `OTROS` | Todo lo demás identificable con certeza (proveedores, reclamos, pagos, prensa) |
| `REVISION` | Vía de escape hacia un humano — ambigüedad, mensaje vacío o confuso |

Un nodo de código (`Parsear clasificación1`) valida esta salida con dos guardas: **taxonomía cerrada** (cualquier valor fuera de las 4 categorías se fuerza a `REVISION`) y **umbral de confianza** (confianza baja también degrada a `REVISION`, nunca se actúa "a medias"). El remitente real se extrae siempre de la cabecera `From` del correo — nunca de lo que la IA cree entender del texto, que es una fuente no confiable para un dato que se necesita exacto (email de contacto).

### Los 4 guardrails de seguridad (Checkpoint 4)

| # | Guardrail | Dónde vive | Qué evita |
|---|---|---|---|
| ① | IF *"¿Es auto-reply?"* inmediatamente después del trigger de Gmail | `¿Es auto-reply?1` | Bucle infinito de auto-respuestas (Out of Office, Undeliverable, no-reply@) |
| ② | Look up antes de Create en el CRM | `Search contacts1` → `Contacto existe?1` → `Update`/`Create contact` (HubSpot) | Error 409 por contactos duplicados |
| ③ | Create Draft, nunca envío automático | `③ Gmail: Create Draft` | Que la IA le conteste a un cliente sin que un humano lo revise (Human-in-the-loop) |
| ④ | Set de limpieza de payload | `④ Validar contacto para CRM` (antes de HubSpot) y `④ Limpiar payload para Slack` (antes de Slack) | Payloads pesados/mal formados hacia el CRM (error 400) y saturación del canal de Slack |

**Los 3 conectores externos reales, con OAuth2:**
- **Gmail** — casilla de soporte al cliente (lectura vía trigger, escritura vía Create Draft).
- **HubSpot** — CRM de la tienda (lectura vía Search Contact, escritura vía Create/Update Contact).
- **Slack** — canal del equipo de operaciones, avisado cada vez que se crea un borrador pendiente de revisión.

### Rama "COMPRA" → tour armado

Cuando el cliente pide ver tours ya armados (el caso más común, según el propio Worker de clasificación), el sistema:
1. Lee el catálogo completo de tours desde una **hoja de Google Sheets** (27 tours reales, 3 por cada uno de los 9 países que trabaja Neblina Forest: Argentina, Brasil, Chile, Colombia, Bolivia, Ecuador, Perú, Vietnam, Uganda).
2. Filtra esos tours por el/los países que mencionó el cliente usando un **nodo de código determinístico** (nunca un modelo de IA) — si no mencionó ningún país, se devuelven los 27.
3. Arma un correo HTML de marca Neblina Forest con una **tabla dinámica** que crece o se achica sola según cuántos tours haya que mostrar.
4. Crea el borrador en Gmail (guardrail ③) y avisa al equipo por Slack.

---

## 3. Los Workers (subworkflows invocados por Execute Workflow)

Cada Worker recibe un input fijo, nunca redacta nada por su cuenta, y siempre devuelve `{ status, data, meta }`. Ninguno "adivina": cuando faltan datos críticos o la confianza del modelo es baja, se marca explícitamente como pendiente de revisión en vez de guardar algo a medias.

### `CH - Worker de Vuelos`
Extrae itinerarios de vuelo (aerolínea, número de vuelo, aeropuertos, fechas, código de reserva) de un correo con un AI Agent. Antes de guardar nada, **busca al cliente por email en Airtable** — si no lo encuentra, no crea un vuelo huérfano: avisa al asesor por mail para vincularlo a mano. Si falta el número de vuelo o la fecha de salida, o la confianza es baja, el registro se guarda con `estado_validación: conflicto` en vez de `completo`.

### `CH - Worker · Clasificador de Compra`
Decide si el interés del cliente es por un tour **armado** (el catálogo existente — caso por defecto) o **personalizado** (excepción, requiere evidencia explícita: itinerario privado, fechas propias, grupo cerrado). Extrae además países mencionados, fechas, cantidad de personas y un resumen breve — esos datos son los que alimentan después el filtro del catálogo en el Manager.

### `CH - Worker Otros`
Triage de todo lo que no es vuelos ni compra: reclamos, reservas de alojamiento, pagos, correos del propio equipo, pedidos de información del cliente. Clasifica por `importancia` (alta/baja) y solo escala al asesor los casos de importancia alta.

---

## 4. El Asesor por Telegram (`CH - Asesor Telegram`)

Chatbot interno — **no atiende clientes, es una herramienta para que el equipo de Neblina Forest consulte datos** de clientes ya cargados en Airtable (pasaporte, alojamiento, salud y dieta, intereses, ventas, vuelos) sin tener que abrir la base manualmente.

- **Memoria persistente por sesión** (Módulo 3): cada chat de Telegram tiene un `Session_ID` propio. Un nodo `IF` (`¿Sesión existente?`) distingue si es la primera vez que ese chat interactúa o si ya tiene historial — si es nuevo, crea el registro inicial en `Resumenes_session` antes de guardar el primer mensaje; si ya existe, solo guarda el mensaje.
- **Resumen automático de contexto**: cada 5 mensajes (cadencia periódica, para controlar el costo de llamadas al modelo) se dispara `CH - Resumidor de Sesión`, que le pide a un modelo barato un resumen de 3 claves (`asunto_principal`, `puntos_clave`, `acción_requerida`) y lo guarda con upsert en Airtable — nunca duplica datos puntuales de un cliente en el resumen (esos siempre se vuelven a buscar en vivo con la herramienta, nunca se copian a la memoria de largo plazo).
- **Inyección de contexto compartido**: el resumen de la sesión anterior se inyecta en el `systemPrompt` del agente entre delimitadores rígidos (`[INICIO DE CONTEXTO COMPARTIDO]` / `[FIN DEL CONTEXTO COMPARTIDO]`), para que el modelo lo trate como contexto de referencia y no como una instrucción nueva.
- **Herramienta `Buscar Cliente (Tool)`**: antes de responder cualquier consulta sobre un cliente puntual, el agente está obligado (por prompt) a llamar a esta tool — nunca puede inventar un dato de cliente por su cuenta.
- **Saneo de salida**: Telegram interpreta `_` y `*` sueltos como formato y rompe si no cierran (por ejemplo, "Cliente_ID" abre una cursiva que nunca cierra) — un nodo de código limpia esos caracteres antes de responder.

### `CH - Tool Buscar Cliente`
Subworkflow-herramienta: recibe una consulta libre (Cliente_ID, email o nombre), busca al cliente en Airtable y trae **las 7 tablas relacionadas** en una sola llamada (Datos personales, Pasaporte, Alojamiento, Salud y Dieta, Intereses, Ventas, Vuelos), devolviendo un único JSON combinado. Si no encuentra al cliente, lo dice explícitamente en vez de devolver datos parciales.

### `CH - Resumidor de Sesión`
Junta todos los mensajes de una sesión de Telegram, le pide a un modelo el resumen de 3 claves, y hace upsert (`Actualizar` si ya existe una fila para ese `Session_ID`, `Crear` si no) en la tabla `Resumenes_session`.

---

## 5. `CH - Confirmar Compra NF`

Workflow basado en **webhooks** (no en trigger de correo) que atiende dos flujos separados de alta de datos:

- **Alta de cliente + confirmación de compra**: recibe los datos de una compra ya decidida, crea el registro en `Ventas` vinculado al `Cliente` (buscándolo primero por email; si no existe, lo crea con un `Cliente_ID` generado automáticamente y estado `documentación pendiente`).
- **Formulario "Guest Information" (Karen)**: sirve un formulario HTML completo de datos del pasajero (pasaporte, alojamiento, salud, contacto de emergencia, intereses puntuados del 1 al 5) directamente desde n8n vía `Respond to Webhook`, y procesa su envío buscando de nuevo al cliente por email antes de guardar nada.

---

## 6. Base de datos (Airtable — "Neblina Forest DATA")

Todas las tablas relacionales viven en una única base de Airtable, con `Cliente_ID`/`Email` como claves de vinculación entre tablas:

`Clientes` · `Datos_personales` · `Pasaporte` · `Alojamiento` · `Salud_y_Dieta` · `Intereses` · `Ventas` · `Vuelos` · `Memoria_conversaciones` (log crudo de Telegram) · `Resumenes_session` (memoria de largo plazo, una fila por sesión)

## 7. Catálogo de tours (Google Sheets)

Hoja externa con 27 tours reales (invención de datos con formato realista para la entrega): País, Nombre del Tour, Fecha de Salida, Duración, Costo (USD), Cupos Disponibles, Descripción. La leen `checkpoint4` para armar la oferta de tour armado, filtrando siempre con código determinístico — nunca dejando que la IA decida qué tour existe o cuánto cuesta.

---

## 8. Credenciales necesarias para correr el proyecto

| Servicio | Uso |
|---|---|
| Gmail OAuth2 (x2: casilla de soporte + casilla de logs internos) | Trigger de entrada, envío de avisos internos, Create Draft |
| HubSpot OAuth2 | CRM — Search/Create/Update Contact |
| Slack OAuth2 | Aviso al equipo de operaciones |
| Google Sheets OAuth2 | Catálogo de tours |
| Airtable OAuth2 | Base de datos relacional del proyecto |
| Telegram Bot API | Asesor interno |
| OpenAI / Google Gemini / Anthropic (Claude) | Modelos de lenguaje de los distintos agentes — el proyecto usa varios proveedores según el workflow |

---

## 9. Mapa de archivos

| Archivo | Rol |
|---|---|
| `checkpoint4_velasquez_garcia_richard.json` | **Entregable del checkpoint** — Manager con los 4 guardrails de integración (autocontenido) |
| `CH - Worker de Vuelos.json` | Worker: extracción de itinerarios de vuelo |
| `CH - Worker · Clasificador de Compra.json` | Worker: clasifica armado vs. personalizado |
| `CH - Worker Otros.json` | Worker: triage de importancia para todo lo demás |
| `CH - Asesor Telegram.json` | Chatbot interno con memoria persistente (Módulo 3) |
| `CH - Resumidor de Sesión.json` | Subworkflow de resumen automático de contexto |
| `CH - Tool Buscar Cliente.json` | Herramienta de consulta relacional de clientes |
| `CH - Confirmar Compra NF.json` | Webhooks de alta de cliente, compra y formulario de datos del pasajero |

---

*Proyecto integrador desarrollado a lo largo de los módulos del curso de Coderhouse, evolucionando el mismo workflow base checkpoint a checkpoint hasta el Proyecto Final.*
