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

Los 15 registros de demostración de la base fueron generados antes de definir la taxonomía final en español (`category`/`priority`/`status`). Al limpiar las opciones de esos campos para dejar únicamente los valores que el flujo de n8n genera (ver `02_manual_datos.md`), los valores de demostración en inglés quedaron vacíos — es decir, el dashboard muestra actualmente 15 tickets sin clasificar, 0 en estado `Error`. Esto es esperado: el dashboard es una vista **operativa en vivo**, no una foto fija, y se poblará automáticamente con datos reales en cuanto el flujo procese tickets reales a través del webhook. La estructura de KPIs, agrupación y enlaces públicos ya está completamente funcional y verificada.

## 4. Minimización de datos en los enlaces públicos

Siguiendo el mismo principio del documento de seguridad, el campo `customer_email` está oculto en ambas vistas de "solo lectura" — los enlaces públicos no exponen direcciones de correo de clientes, aunque sí incluyen nombre, empresa y el resto de los campos operativos necesarios para auditar el funcionamiento del flujo.
