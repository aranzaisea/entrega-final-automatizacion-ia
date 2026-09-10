# Documentación de Seguridad y Resiliencia
## Ecosistema de Automatización IA — Triage de Tickets de Soporte

## 1. Minimización de datos

El principio aplicado en todo el flujo es enviar a cada sistema externo únicamente el dato estrictamente necesario para su función:

| Dato del ticket | ¿Se envía a OpenAI? | ¿Se envía a Slack? | ¿Se guarda en Airtable? |
|---|---|---|---|
| `description` (contenido del ticket) | Sí — es el único input necesario para clasificar | No se reenvía completo, solo referenciado en el resumen de notificación | Sí (campo fuente) |
| `customer_email` | **No** | **No** | Sí (necesario para contacto de soporte) |
| `customer_name` | **No** | Sí, solo en el mensaje de notificación al equipo humano | Sí |
| `subject` | **No** (no aporta señal adicional a la categoría/prioridad más allá de `description`) | Sí, en la notificación | Sí |

- El nodo de clasificación de IA (`Clasificar Ticket con IA`) recibe **solo** el campo `description`, no el email ni ningún identificador directo del cliente. Esto reduce la superficie de datos personales expuesta a un proveedor de IA externo al mínimo indispensable para la tarea.
- Ningún dato sensible (contraseñas, tokens, información de pago) circula en el flujo — el caso de uso (triage de tickets) no lo requiere y el diseño del formulario de entrada tampoco lo captura.
- Las credenciales (API key de OpenAI, token de Airtable, token de Slack) se gestionan mediante el sistema de *Credentials* nativo de n8n (cifradas en reposo), nunca como texto plano en los nodos ni en este repositorio.

## 2. Manejo de errores (resiliencia)

El flujo implementa manejo de errores explícito (`onError: continueErrorOutput`) en los **dos puntos de mayor riesgo** de la ejecución: la escritura inicial en la base de datos y la llamada al modelo de IA. En ambos casos, un fallo enruta a una rama de error dedicada en vez de detener el flujo silenciosamente:

| Nodo que puede fallar | Salida de error va a | Acción de recuperación |
|---|---|---|
| Crear Registro en Airtable | Notificar Error de Creación (Slack) | Alerta inmediata al equipo — sin registro creado, no hay forma de reintentar automáticamente el guardado, así que se prioriza visibilidad humana rápida |
| Clasificar Ticket con IA (llamada a OpenAI) | Registrar Error en Airtable | El ticket ya existe en Airtable (creado en el paso anterior), así que el error se registra actualizando ese mismo registro con `status: "Error"` y `error_message`, en vez de perder el ticket |
| Actualizar: Procesado por IA (escritura del resultado de la IA) | Registrar Error en Airtable | Mismo patrón — si la escritura del resultado de la IA falla, el ticket se marca como `Error` en vez de quedar en un estado inconsistente |

**Por qué solo estos dos puntos y no los nodos de Slack posteriores:** una vez que el ticket pasó por `Actualizar: Procesado por IA`, ya está persistido en Airtable con `category` y `priority` correctos — el dato de negocio ya no está en riesgo. Los nodos de Slack posteriores (`Solicitar Aprobación en Slack`, `Notificar Ticket en #soporte`) y las actualizaciones de estado que les siguen no tienen una rama de error dedicada en esta versión: si Slack fallara en ese punto, la ejecución se detendría con el ticket ya guardado y clasificado en Airtable (recuperable manualmente), pero sin el registro explícito de `error_message` ni la alerta automática que sí existe para los dos puntos críticos de la tabla. Se documenta esto como una limitación conocida y una mejora pendiente, no como un comportamiento no verificado.

**Principio de diseño:** ante una falla en la creación del registro o en la clasificación por IA — los dos puntos donde se puede perder el dato de origen o dejarlo en un estado inconsistente — el sistema **nunca pierde el ticket**: siempre queda persistido en Airtable con un estado (`Error`) y un mensaje de error explicativo, permitiendo intervención manual posterior sin tener que reconstruir el incidente desde los logs de n8n.

**Casos de falla cubiertos explícitamente:**
- **Datos faltantes:** si el webhook recibe un payload incompleto (p. ej. sin `description`), el nodo de Airtable de creación falla su validación de campo requerido y enruta a la rama de error en vez de crear un registro corrupto o dejar que el nodo de IA reciba un input vacío.
- **Fallas de API externa:** timeouts o errores 4xx/5xx de OpenAI, Airtable o Slack son capturados por el modo `Continue On Fail` habilitado en cada nodo crítico, que redirige a la salida de error en lugar de abortar toda la ejecución del workflow.
- **Errores de permisos:** un error de autenticación/autorización (p. ej. 403 de Airtable) se trata igual que cualquier otro error de API — se registra y notifica, no se reintenta automáticamente con credenciales, evitando bucles de reintento contra un fallo de configuración que requiere intervención humana.

## 3. Punto de Human-in-the-loop (HITL)

**Ubicación en el flujo:** entre la clasificación por IA (`¿Prioridad Crítica? (IF)`) y la notificación final al cliente/canal de soporte.

**Condición de activación:** cuando la IA clasifica el ticket con `priority = "Urgente"` **o** `priority = "Alta"` (nodo IF `¿Prioridad Crítica?`, condiciones combinadas con `OR`).

**Mecanismo:** el nodo `Solicitar Aprobación en Slack` usa la función nativa `sendAndWait` de n8n con `Response Type: Approval`. Esto **pausa la ejecución del workflow** (no es un simple mensaje informativo) hasta que una persona del equipo de soporte responde *Aprobar* o *Rechazar* directamente en Slack. El workflow no continúa hacia la actualización de estado ni hacia la notificación al cliente sin esa respuesta humana explícita.

**Por qué este punto y no otro:** un ticket de prioridad Urgente o Alta es, por definición, el caso donde un error de clasificación de la IA tiene mayor costo (p. ej. escalar innecesariamente, o no escalar un caso realmente crítico). Se eligió este único punto de control humano — en vez de aprobar cada ticket, lo cual anularía el propósito de automatización — porque concentra la supervisión humana exactamente donde el riesgo es mayor, cumpliendo el requisito de "un punto de validación humana antes de una acción crítica" sin convertir todo el flujo en manual.

## 4. Resumen de controles

| Categoría | Control implementado |
|---|---|
| Minimización de datos | Solo `description` va a la IA; sin datos de pago/credenciales en el flujo; credenciales cifradas en el gestor nativo de n8n |
| Resiliencia ante datos faltantes | Validación de campos requeridos en la creación del registro Airtable + rama de error dedicada |
| Resiliencia ante fallas de API | `Continue On Fail` + ramas de error dedicadas en los 2 puntos de mayor riesgo: creación del registro en Airtable y clasificación por OpenAI |
| Trazabilidad de errores | Los errores capturados se persisten en Airtable (`status: Error`, `error_message`) y se notifican por Slack en tiempo real; los nodos de Slack posteriores al HITL quedan documentados como limitación pendiente (ver sección 2) |
| Human-in-the-loop | Aprobación obligatoria vía Slack (`sendAndWait`) para tickets de prioridad Urgente **o Alta**, antes de notificar al cliente |
