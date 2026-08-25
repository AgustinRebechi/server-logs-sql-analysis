# Plan de Acción: Análisis de Logs de Servidor Web

**Período analizado (post-limpieza):** 2024-01-01 al 2026-08-22
**Total de requests:** 5,587 | **Usuarios únicos:** 2,451 | **Endpoints:** 12 (incluyendo `unknown_endpoint`)

---

## 0. Calidad de datos (previo a cualquier conclusión de negocio)

### Resumen
Antes de poder confiar en cualquier métrica de error o performance, el dataset requirió una capa de staging porque venía con corrupción real, no cosmética:

- **103 registros (1.81%)** tenían timestamps de fechas futuras — se excluyeron de `logs_clean` con un criterio documentado (`timestamp <= CURRENT_DATE`).
- **1,321 registros (23.65%)** no tenían `status_code` válido (NULL tras el cast).
- **1,324 registros (23.7%)** no tenían un `endpoint` identificable, agrupados como `unknown_endpoint` en vez de descartarse.
- El campo `timestamp` venía en **5 formatos distintos**, resueltos con un parseo en cascada (`TRY_STRPTIME`) en la vista `logs_parse`.
- Los formatos solo-fecha (sin hora) quedan con **00:00:00 por defecto**, lo que infla artificialmente el bucket de "hora 0" — se excluye explícitamente de los análisis de tendencia horaria (sección 3).
- Las latencias (`response_time_ms`) tienen una **distribución bimodal**: la mayoría de las requests caen en un rango normal (p95 < 500ms en todos los endpoints), pero un pequeño número de outliers extremos (hasta ~433,000ms) dispara el promedio y el p99. Por eso todo el análisis de performance se reporta en percentiles, no en promedios simples.

### Acciones recomendadas
- [ ] **No usar el promedio de latencia como métrica principal en ningún reporte** — usar p50/p95 y reservar el promedio solo junto con el máximo, para que se note el efecto de los outliers.
- [ ] **Investigar el origen de los `status_code` NULL** (23.65% del tráfico): si es un problema del logger de origen o de la etapa de ingesta, antes de asumir que son errores no registrados.
- [ ] **Definir una política formal para `unknown_endpoint`**: si sigue creciendo en futuros batches, evaluar si es un bug del router/proxy que no está seteando la ruta, y no solo un problema de logging.

---

## 1. Análisis de Errores

### Resumen
El servidor presenta una **tasa de errores 5xx del 16.4%** (916 de 5,587 requests), muy por encima del umbral aceptable (<1%). Los errores se distribuyen de forma relativamente pareja entre endpoints (16%-23%), sin un único punto de falla dominante, lo que sugiere un problema sistémico (infraestructura compartida, timeouts, DB) más que un bug aislado en un endpoint.

**Concentración relativa:** `/api/orders` (23.02%), `/metrics` (21.87%) y `/api/users` (21.64%) tienen las tasas de error más altas. Curiosamente `/api/cart`, que suele ser el endpoint más sensible en e-commerce, tiene la tasa más baja de los endpoints conocidos (16.25%) — vale la pena confirmar que esto no es un artefacto de la limpieza de datos antes de descartarlo como "no crítico".

### Acciones recomendadas
- [ ] **Priorizar `/api/orders` y `/metrics`:** son los endpoints con mayor error rate relativo; revisar si comparten dependencias (DB, servicio de métricas interno) que puedan estar saturadas.
- [ ] **Investigar por qué `/metrics` tiene tanto error 5xx:** no es un endpoint transaccional de negocio, por lo que un 21.87% de error ahí probablemente indica un problema de infraestructura de monitoreo, no de lógica de aplicación — vale la pena separarlo del análisis de endpoints de negocio.
- [ ] **Confirmar la tasa real de `/api/cart` con una muestra manual:** su bajo error rate relativo podría ser real o un efecto de cómo quedó after la limpieza de `unknown_endpoint`.

---

## 2. Análisis de Tráfico y Patrones Temporales

### Resumen
El tráfico está relativamente balanceado entre endpoints conocidos (6.7%-7.4% cada uno), aunque el **23.7% del tráfico total** cae en `unknown_endpoint`, lo que limita la confianza en cualquier ranking de "endpoint más usado". La hora pico real (excluyendo la hora 0, inflada por el default de casteo) es las **18:00** con 251 requests, seguida de las 6:00 con 247.

**Nota sobre la variación diaria:** el promedio de cambio porcentual día a día da **+31.79%**, pero este número es engañoso: los primeros días del período tienen volúmenes muy bajos (8, 3, 5, 7 requests), donde una diferencia de 2-3 requests genera swings de 40-60%. No lo reportaría como "tendencia de crecimiento" sin antes filtrar los días de bajo volumen o usar una ventana móvil de 7 días.

### Acciones recomendadas
- [ ] **Recalcular la variación diaria excluyendo o suavizando los días de volumen bajo** (ej. primeros 30 días o promedio móvil de 7 días) antes de usarla como KPI de crecimiento.
- [ ] **Escalar recursos antes de las 18:00 y las 6:00:** son los dos picos reales una vez descontado el artefacto de hora 0.
- [ ] **Resolver el 23.7% de `unknown_endpoint` antes de confiar en el ranking de tráfico por endpoint** — con ese volumen, el ranking real podría cambiar.

---

## 3. Análisis de Performance por Endpoint

### Resumen
El **p95 se mantiene consistentemente por debajo de 500ms en los 12 endpoints**, lo cual en aislamiento se vería saludable. Pero el promedio y el máximo cuentan otra historia: `/api/users` tiene un promedio de 2,620ms con un máximo de 319,802ms, y `/api/checkout` y `/metrics` tienen p99 de 32,844ms y outliers similares. Esto confirma la distribución bimodal descripta en la sección 0: la gran mayoría de las requests son rápidas, pero una fracción pequeña (outliers extremos, posiblemente timeouts colgados o corrupción de datos) infla dramáticamente el promedio y los percentiles altos.

`/api/products` es la excepción: su máximo es de solo 500ms, es decir, no tiene outliers — el endpoint más "limpio" del dataset en términos de latencia.

### Acciones recomendadas
- [ ] **Investigar los outliers de latencia como incidentes puntuales, no como comportamiento normal:** filtrar las requests con `response_time_ms` > percentil 99.9 y revisar si comparten `user_id`, `client_ip` o rango horario — podrían ser timeouts colgados o un bug de instrumentación (unidad incorrecta, reloj desincronizado).
- [ ] **Definir un SLO basado en p95, no en promedio:** con p95 < 500ms en todos los endpoints, el servicio cumple un SLO razonable en el caso típico; el problema real está en la cola larga, no en el caso general.
- [ ] **Usar `/api/products` como baseline de comparación:** al no tener outliers, sirve como referencia de "cómo se ve un endpoint sano" en este dataset.

---

## 4. Análisis de Errores por Método HTTP

### Resumen
**GET concentra la mayor cantidad absoluta de errores 5xx** (`/api/orders` con 36, `/api/products` con 35, `/api/users` con 33), lo cual es esperable dado que también es el método más usado. Pero el hallazgo más llamativo está en **PUT `/metrics`**, con un promedio de respuesta de **426,099ms** (más de 7 minutos) — un valor que no representa latencia real de un endpoint de métricas y es casi con certeza un outlier extremo o un error de instrumentación, no un problema de performance a resolver como tal.

### Acciones recomendadas
- [ ] **No incluir el promedio de `PUT /metrics` en ningún reporte sin aclarar que está dominado por outliers** — reportar la mediana en su lugar, o excluirlo con nota explícita.
- [ ] **Auditar si GET concentra errores en proporción a su volumen o desproporcionadamente:** normalizar por total de requests de cada método antes de concluir que "GET es el método más problemático".
- [ ] **Revisar DELETE en `/api/search`, `/api/products` y `/api/payments`** (11 errores cada uno): son operaciones destructivas, y un error 5xx ahí tiene más riesgo (inconsistencia de datos) que uno en una lectura.

---

## Métricas de Seguimiento (KPIs)

| Métrica | Valor Actual | Objetivo |
|---|---|---|
| Tasa de errores 5xx | 16.4% | <1% |
| % de `status_code` faltante | 23.65% | <2% |
| % de `endpoint` no identificable | 23.7% | <2% |
| Hora pico real (excl. hora 0) | 18:00, 251 req | Sin degradación |
| Endpoints con >20% error rate | 3 de 12 | 0 |
| p95 de latencia | <500ms (todos los endpoints) | Mantener <500ms |
| p99 de latencia (peores casos) | hasta ~32,800ms | <1,000ms |

---
