# Matriz de Costos y Optimización de Modelos de IA
## Ecosistema de Automatización IA — Triage de Tickets de Soporte

## 1. Tarea de IA en el flujo

El flujo tiene **una sola tarea de IA**: clasificar cada ticket entrante en `category` (4 valores posibles) y `priority` (4 valores posibles) a partir del campo `description`. Es una tarea de clasificación de texto corto con salida estructurada (JSON), **no** generación creativa ni razonamiento complejo — por lo tanto no requiere un modelo de frontera.

**Supuestos de volumen** (para estimar costos mensuales): 1,000 tickets/mes, ~180 tokens de entrada por llamada (prompt de sistema + `description` del cliente) y ~15 tokens de salida (JSON corto).

## 2. Modelos evaluados

| Modelo | Precio entrada (por 1M tokens) | Precio salida (por 1M tokens) | ¿Apto para esta tarea? | Costo estimado / mes (1,000 tickets) |
|---|---|---|---|---|
| **GPT-4o-mini** (elegido) | $0.15 | $0.60 | Sí — clasificación estructurada simple, sobra capacidad | **~$0.027 + $0.009 = ~$0.036** |
| GPT-5 Mini | $0.25 | $2.00 | Sí, pero más caro sin beneficio para esta tarea | ~$0.045 + $0.030 = ~$0.075 |
| GPT-4o (legacy) | $2.50 | $10.00 | Sobredimensionado — 16x más caro que 4o-mini | ~$0.450 + $0.150 = ~$0.600 |
| GPT-5 | $1.25 | $10.00 | Sobredimensionado para clasificación de 4 categorías | ~$0.225 + $0.150 = ~$0.375 |
| Claude Haiku 4.5 | $1.00 | $5.00 | Alternativa viable si se migrara de proveedor | ~$0.180 + $0.075 = ~$0.255 |
| Claude Sonnet 4.6 | $3.00 | $15.00 | Sobredimensionado, mismo caso que GPT-4o | ~$0.540 + $0.225 = ~$0.765 |

*(Precios de referencia consultados en la documentación pública de OpenAI y Anthropic, septiembre 2026.)*

## 3. Justificación del modelo elegido: GPT-4o-mini

- **Ajuste tarea–modelo:** clasificar texto corto en categorías cerradas es una tarea de baja complejidad de razonamiento. Los benchmarks públicos de OpenAI muestran que GPT-4o-mini iguala a modelos de gama alta en tareas de clasificación e instrucción estructurada simple, por lo que pagar por un modelo de frontera (GPT-5, GPT-4o, Sonnet) no se traduce en mejor calidad de salida para este caso de uso.
- **Salida estructurada corta:** al forzar un JSON de dos campos, el costo de salida (la parte más cara por token en casi todos los modelos) se mantiene mínimo independientemente del modelo — la diferencia de costo total está dominada por el precio de entrada, donde GPT-4o-mini es el más económico de la tabla.
- **Latencia:** GPT-4o-mini tiene menor latencia que los modelos más grandes, lo cual importa porque el ticket completo (incluida la posible espera de aprobación humana) debe resolverse dentro de una ejecución de webhook con timeout limitado.
- **Ya integrado:** se usa la misma cuenta/API key de OpenAI que ya está conectada en n8n, sin necesidad de gestionar credenciales adicionales de otro proveedor.

## 4. Ahorro estimado

Comparado contra la alternativa "segura por defecto" que muchos equipos eligen sin optimizar (GPT-4o o GPT-5 completo):

| Comparación | Costo/mes con GPT-4o-mini | Costo/mes alternativa | Ahorro mensual | Ahorro anual | Ahorro % |
|---|---|---|---|---|---|
| vs. GPT-4o (legacy) | $0.036 | $0.600 | $0.564 | $6.77 | ~94% |
| vs. GPT-5 | $0.036 | $0.375 | $0.339 | $4.07 | ~90% |
| vs. Claude Sonnet 4.6 | $0.036 | $0.765 | $0.729 | $8.75 | ~95% |

En términos absolutos el monto es bajo porque el volumen de prueba (1,000 tickets/mes) es modesto, pero el ahorro **porcentual (90–95%)** es el punto relevante: escala linealmente con el volumen. A 100,000 tickets/mes (una operación de soporte mediana), el mismo ahorro porcentual representa **~$56–75/mes** solo por la elección de modelo, sin ningún cambio en la calidad de clasificación.

## 5. Optimización adicional considerada (y descartada para este flujo)

- **OpenAI Batch API (-50% flat):** aplicable solo a cargas asíncronas que toleran hasta 24h de latencia. Se descarta aquí porque el flujo es **tiempo real** (dispara notificaciones Slack inmediatas y puede requerir aprobación humana en minutos) — usar Batch rompería el requisito de "sin intervención manual" en el flujo principal. Podría evaluarse en el futuro para un job nocturno de re-clasificación masiva/auditoría, no para el flujo transaccional.
- **Prompt caching:** el prompt de sistema es fijo y corto; con mayor volumen se recomienda anclarlo como prefijo cacheable para reducir aún más el costo de entrada repetido.
