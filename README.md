# Ecosistema de Automatización IA — Triage de Tickets de Soporte

Entrega final del curso **"Ecosistema de Automatización IA Autónomo para Negocios"**. Sistema que clasifica tickets de soporte entrantes con IA (categoría y prioridad), los registra en una base de datos relacional, y notifica al equipo de soporte por Slack — con un punto de aprobación humana obligatorio para tickets de prioridad crítica.

## Stack (las 4 categorías obligatorias)

| Categoría | Herramienta |
|---|---|
| Orquestador | [n8n](https://n8n.io) (n8n.cloud) |
| Base de datos | [Airtable](https://airtable.com) — tablas relacionadas `Clientes` ↔ `Tickets` |
| Procesamiento IA | [OpenAI](https://openai.com) (GPT-4o-mini) con prompt estructurado de salida JSON |
| Canal de salida | [Slack](https://slack.com) — notificaciones y aprobación humana (HITL) |

## Cómo funciona (resumen)

1. Un webhook POST recibe un nuevo ticket (`customer_name`, `customer_email`, `subject`, `description`).
2. Se crea el registro en Airtable (tabla `Tickets`, vinculada a `Clientes`).
3. OpenAI clasifica el ticket en `category` y `priority` a partir de la descripción.
4. Si la prioridad es **Urgente**, el flujo se detiene y espera aprobación humana vía Slack (`sendAndWait`) antes de continuar — este es el punto de Human-in-the-Loop.
5. Se notifica al canal de soporte en Slack y se actualiza el estado final del ticket en Airtable.
6. Cualquier falla en un nodo (Airtable, OpenAI o Slack) se enruta a nodos de error dedicados que registran el fallo y notifican al equipo — el flujo nunca se detiene silenciosamente.

El diagrama completo está en [`docs/01_diagrama_arquitectura.pdf`](docs/01_diagrama_arquitectura.pdf).

## Contenido del repositorio

```
docs/
  01_diagrama_arquitectura.pdf     Diagrama de arquitectura (triggers, routers, APIs, nodos IA, destinos de datos)
  02_manual_datos.md               Esquema relacional de Airtable + esquemas JSON de cada integración, explicados
  03_matriz_costos.md              Comparativa de modelos de IA, justificación del elegido y ahorro estimado
  04_seguridad_resiliencia.md      Minimización de datos, manejo de errores y punto de HITL
  05_dashboard_control.md          Enlaces públicos al dashboard de KPIs y a la base de datos (solo lectura)
workflow/
  triage_tickets_soporte.json      Export técnico del flujo de n8n
evidencia/
  *.jpg / *.png                    Capturas de ejecución y del dashboard público
```

## Enlaces obligatorios

- **Base de datos (solo lectura)**: ver [`docs/05_dashboard_control.md`](docs/05_dashboard_control.md) — incluye el enlace a la vista pública de `Tickets` y de `Clientes`.
- **Dashboard de control (KPIs)**: ver el mismo documento — vista pública agrupada por estado, con tasa de error y tiempo de resolución.

## Las 5 entregas evaluadas

1. **Diagrama de arquitectura** — `docs/01_diagrama_arquitectura.pdf`
2. **Manual operativo de datos** — `docs/02_manual_datos.md`
3. **Matriz de costos / optimización de modelos IA** — `docs/03_matriz_costos.md`
4. **Documentación de seguridad y resiliencia** — `docs/04_seguridad_resiliencia.md`
5. **Dashboard de control** — `docs/05_dashboard_control.md`

## Estado operativo

El flujo está construido, publicado en n8n y fue probado de punta a punta con una ejecución real vía webhook, lo que permitió detectar y corregir errores de configuración. Dos ajustes de cuenta (fuera del alcance de esta automatización, pendientes de la propietaria del proyecto) son necesarios antes de que las 5 ejecuciones de evidencia con datos reales puedan capturarse: habilitar el acceso de la base de Airtable en el token (PAT) usado por n8n, y crear/anular la invitación del bot a los canales de Slack `#soporte` y `#soporte-aprobaciones`. La arquitectura, el manejo de errores y el punto de HITL ya están verificados en el diseño y en las capturas de la ejecución de prueba.
