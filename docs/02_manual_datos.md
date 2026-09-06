# Manual Operativo de Datos
## Ecosistema de Automatización IA — Triage de Tickets de Soporte

Este documento describe el modelo de datos en Airtable y los esquemas JSON que viajan entre cada integración del flujo de n8n (`Triage de Tickets de Soporte`).

---

## 1. Modelo relacional en Airtable

**Base:** `Untitled Base` (`appW1W1cT9qmTkf33`)

El modelo evita datos aislados mediante una relación **uno-a-muchos** entre dos tablas vinculadas: un cliente puede tener varios tickets, y cada ticket pertenece a un único cliente.

```
┌────────────────────────┐          ┌──────────────────────────────┐
│   Clientes              │ 1 ── N   │   Tickets                     │
│  (tblQdWyGSLhaTf9hU)    │◄────────►│  (tbldsGQver1PYd0hA)           │
├────────────────────────┤          ├──────────────────────────────┤
│ Nombre (primary)        │          │ Ticket ID (primary/autonum)   │
│ Email                   │          │ Cliente (link → Clientes)     │
│ Empresa                 │          │ customer_name                 │
│ Tickets vinculados       │          │ customer_email                 │
│  (link ← Tickets)        │          │ subject                       │
└────────────────────────┘          │ description                    │
                                     │ category (select)              │
                                     │ priority (select)               │
                                     │ status (select)                │
                                     │ error_message                   │
                                     │ Created                         │
                                     │ Historial de estado             │
                                     │ Tiempo hasta cierre             │
                                     └──────────────────────────────┘
```

### 1.1 Tabla `Clientes`

| Campo | Tipo | Descripción |
|---|---|---|
| Nombre | Texto (primary) | Nombre del cliente que abre el ticket |
| Email | Email | Correo de contacto |
| Empresa | Texto | Empresa/organización del cliente |
| Tickets vinculados | Link (back-reference) | Referencia inversa automática a todos los registros de `Tickets` asociados a este cliente |

### 1.2 Tabla `Tickets`

| Campo | Tipo | Descripción |
|---|---|---|
| Ticket ID | Autonumber (primary) | Identificador único del ticket |
| Cliente | Link → Clientes | Relación con el cliente que generó el ticket |
| customer_name | Texto | Nombre del cliente (copiado del payload entrante, redundante por diseño para no depender de un lookup en tiempo de IA) |
| customer_email | Texto | Email del cliente (payload entrante) |
| subject | Texto | Asunto del ticket |
| description | Texto largo | Descripción completa enviada por el cliente — único campo que se envía a la IA para clasificación |
| category | Select | `Facturacion` / `Tecnica` / `Cuenta` / `Otro` — asignado por OpenAI |
| priority | Select | `Urgente` / `Alta` / `Media` / `Baja` — asignado por OpenAI |
| status | Select | `Pendiente` → `Procesado por IA` → (`Aprobado por Humano`) → `Enviado` / `Error` — refleja el estado del ticket dentro del flujo |
| error_message | Texto largo | Mensaje de error capturado si algún nodo falla (poblado solo por la rama de error) |
| Created | Fecha/hora | Timestamp de creación del registro |
| Historial de estado | Texto largo | Bitácora de transiciones de estado (auditoría) |
| Tiempo hasta cierre | Número decimal | Horas transcurridas entre creación y cierre del ticket (usado como KPI en el dashboard) |

**Taxonomía cerrada:** los tres campos `category`, `priority` y `status` son de tipo *Single Select* con opciones fijas en español — esto evita valores libres/inconsistentes y permite que el nodo IF del flujo tome decisiones deterministas sobre `priority`.

---

## 2. Esquemas JSON de transferencia entre integraciones

Cada punto de integración del flujo transfiere un JSON con una forma específica. A continuación se documenta cada uno, en el orden en que ocurren dentro del flujo.

### 2.1 Webhook de entrada → n8n (Trigger)

**Endpoint:** `POST https://aranzaisea.app.n8n.cloud/webhook/nuevo-ticket-soporte`

```json
{
  "customer_name": "María González",
  "customer_email": "maria.gonzalez@example.com",
  "subject": "No puedo acceder a mi cuenta",
  "description": "Intenté iniciar sesión varias veces y me da error de credenciales, ya reseteé la contraseña dos veces."
}
```
Este es el único punto donde entran datos externos al sistema (variables dinámicas, nada hardcodeado). Todos los nodos posteriores referencian estos campos mediante expresiones n8n, p. ej. `{{ $json.body.customer_name }}`.

### 2.2 n8n → Airtable (Crear Registro)

El nodo `Crear Registro en Airtable` mapea el body del webhook a un nuevo registro en `Tickets`:

```json
{
  "customer_name": "{{ $json.body.customer_name }}",
  "customer_email": "{{ $json.body.customer_email }}",
  "subject": "{{ $json.body.subject }}",
  "description": "{{ $json.body.description }}",
  "status": "Pendiente"
}
```
Respuesta de Airtable (usada por nodos posteriores para el `Columns to match on` en updates):
```json
{
  "id": "recXXXXXXXXXXXXXX",
  "fields": { "...": "..." },
  "createdTime": "2026-09-05T18:00:00.000Z"
}
```

### 2.3 n8n → OpenAI (Clasificación)

**Prompt estructurado** enviado al nodo OpenAI (solo el campo `description`, minimizando datos personales — ver documento de seguridad):

```
Sistema: Eres un clasificador de tickets de soporte. Devuelve ÚNICAMENTE un JSON válido
con esta forma exacta, sin texto adicional:
{"category": "Facturacion|Tecnica|Cuenta|Otro", "priority": "Urgente|Alta|Media|Baja"}

Usuario: {{ $json.body.description }}
```

**Respuesta de OpenAI** (estructura real accedida vía `$json.output[0].content[0].text`, que contiene un string JSON parseado en el nodo siguiente):

```json
{
  "category": "Tecnica",
  "priority": "Alta"
}
```

### 2.4 n8n → Airtable (Actualizar: Procesado por IA)

```json
{
  "id": "{{ $('Crear Registro en Airtable').item.json.id }}",
  "category": "{{ $json.output[0].content[0].text.category }}",
  "priority": "{{ $json.output[0].content[0].text.priority }}",
  "status": "Procesado por IA"
}
```
`Columns to match on: id` — asegura que se actualiza el mismo registro creado en 2.2, no uno nuevo.

### 2.5 n8n → Slack (Solicitud de aprobación — HITL)

Se activa solo si `priority == "Urgente"` (rama `true` del nodo IF). Usa el nodo `sendAndWait` con `Response Type: Approval`.

```json
{
  "channel": "#soporte-aprobaciones",
  "text": ":rotating_light: Ticket de prioridad *Urgente* requiere aprobación\n\n*Cliente:* {{ $json.body.customer_name }}\n*Asunto:* {{ $json.body.subject }}\n*Categoría:* {{ $json.category }}",
  "approvalOptions": { "approve": "Aprobar", "reject": "Rechazar" }
}
```
**Respuesta de Slack** (el flujo se pausa hasta que un humano responde en el hilo):
```json
{
  "data": { "approved": true },
  "workflowId": "VXyZNBepXVSp9GHd"
}
```

### 2.6 n8n → Airtable (Actualizar: Aprobado por Humano)

```json
{
  "id": "{{ $('Crear Registro en Airtable').item.json.id }}",
  "status": "Aprobado por Humano"
}
```

### 2.7 n8n → Slack (Notificación final al canal de soporte)

```json
{
  "channel": "#soporte",
  "text": "✅ Ticket #{{ $('Crear Registro en Airtable').item.json.fields['Ticket ID'] }} clasificado como *{{ $json.category }}* / prioridad *{{ $json.priority }}* — {{ $json.body.subject }}"
}
```

### 2.8 n8n → Airtable (Actualizar: Enviado)

```json
{
  "id": "{{ $('Crear Registro en Airtable').item.json.id }}",
  "status": "Enviado"
}
```

### 2.9 Esquema de error (cualquier nodo → rama de error)

Cuando cualquier nodo del flujo principal falla, su salida `Error` alimenta dos nodos dedicados:

```json
{
  "id": "{{ $('Crear Registro en Airtable').item.json.id }}",
  "status": "Error",
  "error_message": "{{ $json.error.message }}"
}
```
y en paralelo, una notificación a Slack:
```json
{
  "channel": "#soporte",
  "text": ":x: Error en el flujo de triage — nodo: {{ $node.name }} — {{ $json.error.message }}"
}
```

---

## 3. Trazabilidad de variables dinámicas

Ningún nodo del flujo contiene valores de negocio hardcodeados: nombres de cliente, asuntos, descripciones, categorías y prioridades siempre se referencian mediante expresiones n8n (`{{ $json... }}`, `{{ $('Nodo').item.json... }}`). Los únicos valores literales en el flujo son nombres de canal de Slack y nombres de tabla/base de Airtable, que son configuración de infraestructura, no datos de negocio.
