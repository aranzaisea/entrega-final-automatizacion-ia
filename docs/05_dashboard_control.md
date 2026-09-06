# Dashboard de Control
## Ecosistema de Automatización IA — Triage de Tickets de Soporte

## 1. Enlaces públicos (solo lectura)

| Recurso | Enlace | Contenido |
|---|---|---|
| **Dashboard de Control (KPIs)** | https://airtable.com/appW1W1cT9qmTkf33/shrlyjtTmq7ep3sIG | Vista agrupada por `status` con conteo de tickets por estado y suma del tiempo hasta cierre. Pensada para monitoreo operativo del flujo. |
| **Base de datos — Tabla Tickets (solo lectura)** | https://airtable.com/appW1W1cT9qmTkf33/shrTjTjwegaH2pm45 | Todos los registros y campos de `Tickets` (email de cliente oculto por minimización de datos). |
| **Base de datos — Tabla Clientes (solo lectura)** | https://airtable.com/appW1W1cT9qmTkf33/shrgdurbLKAC5q9bf | Todos los registros de `Clientes`, incluyendo la relación `Tickets vinculados` (email oculto). |

Los tres enlaces son vistas compartidas de Airtable ("Shared View"): de solo lectura, sin necesidad de cuenta ni login, y de acceso público mediante el link.

*(Nota técnica: el Diseñador de Interfaces de Airtable —con gráficos de barras/circulares nativos— también se construyó dentro de la base como referencia visual interna ["Dashboard de Control — Triage de Tickets"], pero la publicación pública de Interfaces requiere el plan de pago Team de Airtable. Por eso el dashboard público final se implementó como una Vista de Cuadrícula agrupada, que sí se puede compartir públicamente en el plan gratuito y cumple el mismo propósito de mostrar KPIs en un enlace accesible sin cuenta.)*

## 2. KPIs incluidos

- **Total de tickets**: conteo total de registros en la tabla `Tickets` (mostrado como el número de filas de la vista y como grupo agregado).
- **Tickets por estado**: agrupación por el campo `status` (`Pendiente`, `Procesado por IA`, `Aprobado por Humano`, `Enviado`, `Error`), con conteo automático por grupo — esta es la fuente directa de la **tasa de error** (`Error` / total).
- **Tiempo hasta cierre**: columna `Tiempo hasta cierre (horas)` con una fila de resumen (suma) al pie de la vista, permitiendo evaluar la velocidad de resolución del flujo.
- **Desglose por categoría y prioridad**: columnas `category` y `priority` visibles en la misma vista para cruzar volumen de tickets por tipo de problema y urgencia.

## 3. Estado de los datos al momento de esta entrega

El dashboard es una vista **operativa en vivo**, no una foto fija: se actualiza automáticamente con cada ticket que el flujo procesa a través del webhook. Al momento de esta entrega contiene **36 registros**:

- **15 registros de demostración** (los datos semilla originales de la base, previos a la integración con n8n) — agrupados bajo `status` = `(Vacío)`, sin `category`/`priority` porque nunca pasaron por el flujo.
- **15 registros con `status = Enviado`**: tickets reales disparados vía webhook y procesados de punta a punta por el flujo (clasificación con IA, registro en Airtable y notificación en Slack), incluyendo los cuatro escenarios de evidencia exigidos:
  - Éxito directo sin aprobación (prioridad `Media`/`Baja`).
  - Aprobación humana (HITL) con prioridad **Alta** — ticket de Diego Fuentes, aprobado en `#soporte-aprobaciones` y notificado en `#soporte`.
  - Aprobación humana (HITL) con prioridad **Urgente** — tickets de Sofía Herrera y Mateo Vidal, y de Elena Vargas, aprobados vía el botón "Aprobar" de Slack y notificados con todos los campos (`Cliente`, `Correo`, `Asunto`, `Categoría`, `Prioridad`, `Descripción`) correctamente poblados.
- **6 registros con `status = Error`**: tickets que dispararon la rama de manejo de errores del flujo, incluyendo un error real de la API de Airtable (`INVALID_VALUE_FOR_COLUMN`, capturado en el campo `error_message`) al enviar un valor con tipo de dato inválido — confirmando que ninguna falla de nodo (Airtable, IA o Slack) detiene el flujo silenciosamente: siempre se registra el error y se notifica al equipo.

Durante esta ronda de pruebas se detectó y corrigió un bug de mapeo en dos nodos de Slack (`Solicitar Aprobación en Slack` y `Notificar Ticket en #soporte`): las expresiones referenciaban `item.json.customer_name` en lugar de `item.json.fields.customer_name` (el nodo de Airtable anida la salida bajo `fields`), lo que dejaba `Cliente`/`Correo`/`Asunto` vacíos en las notificaciones. Ambos nodos fueron corregidos y republicados; las capturas en `evidencia/` muestran el resultado ya corregido.

## 4. Minimización de datos en los enlaces públicos

Siguiendo el mismo principio del documento de seguridad, el campo `customer_email` está oculto en ambas vistas de "solo lectura" — los enlaces públicos no exponen direcciones de correo de clientes, aunque sí incluyen nombre, empresa y el resto de los campos operativos necesarios para auditar el funcionamiento del flujo.
