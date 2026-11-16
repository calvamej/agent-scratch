# Resultados de la ingesta (dominio `loans`)

Valor del equipo: `INGEST_CHUNK_MAX_CHARS=1500` (resultado: **29 chunks** de 12 documentos).

## Comparación 250 vs 1500 caracteres

| Métrica | 250 | 1500 |
|---|---|---|
| Chunks totales | 30 | 29 |
| Largo medio / máximo (con el título de sección) | 165 / 292 | 167 / 304 |
| Secciones partidas en más de un chunk | 1 | 0 |
| Chunks que superan el máximo (el título se suma al texto) | 3 | 0 |
| Acierto@1 (el chunk 1 tiene la respuesta completa) | 7/12 | 7/12 |
| Acierto@3 (la respuesta está en los 3 primeros chunks) | 9/12 | 10/12 |
| Respuesta completa dentro de un solo chunk | 9/12 | 10/12 |

El corpus es pequeño: casi todas las secciones miden menos de 250 caracteres, por eso los conteos son casi iguales.
La única sección que cambia es `sample-simulations.csv`, que con 250 se parte en dos tablas (4 filas y 1 fila, cada una con su encabezado) y con 1500 queda en una sola.

### Cómo se midió la precisión

- 12 preguntas con respuesta conocida sobre el corpus. Cada una tiene el dato que debe aparecer en el chunk recuperado (por ejemplo `150 000`, `opcional`, `1 162.63`).
- Búsqueda solo vectorial: embeddings reales `text-embedding-3-small` de Foundry, similitud coseno y top 3. No usa búsqueda por palabras ni filtros de vigencia, y los documentos vencidos (`personal-loan-terms-2023.md`) quedan en el índice.
- Es una muestra de 12 preguntas, sirve para comparar los dos tamaños, no como métrica absoluta.

### Pérdida de contexto

- **Con 250:** la tabla de simulaciones se parte en dos chunks. La pregunta "¿cómo cambia la TEA entre S/ 5 000 y S/ 50 000?" necesita la fila de S/ 5 000 (primer chunk) y la de S/ 50 000 (segundo chunk), así que ninguna recuperación individual alcanza para contestarla. Con 1500 ambas filas están en el mismo chunk, aunque esa pregunta tampoco se contesta por el hallazgo de abajo.
- **Con 1500:** "¿cuál es la cuota de S/ 20 000 a 36 y a 48 meses?" se contesta con un solo chunk (acierto@3), pero con 250 el chunk no aparece en el top 3. Es la única diferencia de resultado entre los dos tamaños.
- **Resto del corpus:** con ambos tamaños el contenido de cada sección cabe en un chunk, así que no se cortó ningún párrafo. En documentos más largos, 250 cortaría reglas a mitad de párrafo. No se midió el efecto contrario (chunks grandes que diluyen el vector), porque con 1500 ningún chunk pasa de 304 caracteres.

### Hallazgos aparte del tamaño

- La tabla `sample-simulations` casi nunca entra en el top 3 con búsqueda solo vectorial, a ningún tamaño (preguntas de cuota y TEA por monto: 0/3 con 250 y 1/3 con 1500). Su único texto es el nombre de sección y números, y los embeddings los distinguen mal. Para las cifras de la tabla conviene la búsqueda por palabras (retrieval híbrido, sesión 4).
- Las preguntas sobre desgravamen traen primero `personal-loan-terms-2023.md` (vencido, dice "obligatorio"). Hace falta filtrar por `valid_until` al buscar.

## Decisión

Se mantiene **1500**: igual precisión en acierto@1, mejor acierto@3 y sin tablas partidas. Con 250 no hay ganancia en este corpus.

## Chat contra Foundry

| # | Tipo | Pregunta | local: fuentes citadas| ¿Fuente esperada? |primer texto / respuesta completa|
|---|---|---|---|---|---|
| 1 | regla del corpus |¿Cuál es el plazo para solicitar un préstamo? | `personal-loan-requirements.pdf` (3. Montos y plazos)| Sí|primer texto en 6987 ms, respuesta completa en 7005 ms|
| 2 | seguimiento |¿y si es una persona natural? | `personal-loan-requirements.pdf` (3. Montos y plazos)<br> `loans-faq.md`| Sí|primer texto en 12883 ms, respuesta completa en 13158 ms|
| 3 | término exacto |¿Qué es la TCEA?| `personal-loan-terms.docx` (Condiciones del préstamo personal -> Tasas)<br>`loans-faq.md`| Se esperaba `loans-glossary.md` > TCEA |primer texto en 13479 ms, respuesta completa en 13619 ms|
| 4 | documento vencido |¿Qué decía la política de préstamos del 2017? |ninguna | Sí. No cita nada. |primer texto en 12797 ms, respuesta completa en 12926 ms|
| 5 | fuera del corpus |¿A cuánto está el dólar hoy?| ninguna | Sí: `out_of_scope` sin fuentes |primer texto en 8367 ms, respuesta completa en 8508 ms|

## Medir con SEARCH_TOP_K

- con 2: ¿falta información en alguna respuesta? - Sí

Pregunta: ¿y si es una persona natural?

  Agente [product_question]: Para persona natural (préstamo personal) el plazo es de 6 a 60 meses. Para montos mayores a S/75 000 se requiere declaración personal de salud si se contrata el seguro de desgravamen.

Original: 

 Agente [product_question]: Para una persona natural: tener entre 21 y 70 años, ingresos desde S/ 1 500 y presentar los documentos según tu tipo de trabajador. Trabajadores dependientes: últimas 3 boletas y 6 meses de antigüedad; independientes: declaración anual de impuestos del último año y 12 meses de actividad; pensionistas: última boleta de pensión. El plazo del préstamo personal es de 6 a 60 meses.

- con 10: ¿cuántos tokens de entrada más cuesta cada turno?, ¿aparecen fuentes que no vienen al caso?;

Pregunta 1:  1068 tokens de entrada, 511 de salida.
Pregunta 2:  1313 tokens de entrada, 1102 de salida.

Aparecen fuentes complementarias a la principal.

- el valor que elige el equipo y por qué:  SEARCH_TOP_K = "2" por su menor consumo de Tokens.

## Chat contra AI Search

| # | Tipo | Pregunta | local: fuentes citadas| AI Search: fuentes citadas |respuesta completa|
|---|---|---|---|---|---|
| 1 | regla del corpus |¿Cuál es el plazo para solicitar un préstamo? | `personal-loan-requirements.pdf`|`personal-loan-requirements.pdf` - `mortgage.html` |local: respuesta completa en 7005 ms, AI Search: respuesta completa en 9517 ms|
| 2 | seguimiento |¿y si es una persona natural? | `personal-loan-requirements.pdf`<br> `loans-faq.md`| `personal-loan-requirements.pdf` |local: respuesta completa en 13158 ms, AI Search: respuesta completa en 10811 ms|
| 3 | término exacto |¿Qué es la TCEA?| `personal-loan-terms.docx`<br>`loans-faq.md`| `personal-loan-terms.docx` |local: respuesta completa en 13619 ms, AI Search: respuesta completa en 9173 ms|
| 4 | documento vencido |¿Qué decía la política de préstamos del 2017? |ninguna | ninguna |local: respuesta completa en 12926 ms, AI Search: respuesta completa en 9677 ms|
| 5 | fuera del corpus |¿A cuánto está el dólar hoy?| ninguna | ninguna |local: respuesta completa en 8508 ms, AI Search: respuesta completa en 8719 ms|

## Línea base (S5)

Resumen de 30 casos (juez local):<br>
  - recall@5: 1.00   <br>
  - intención: 1.00   <br>
  - fuentes: 0.97   
  - hechos: 0.74<br>
  - apoyo: 5.0/5   <br>
  - relevancia: 4.9/5   <br>
  - cita correcta: 1.0<br>
  - fugas de datos personales: 0   <br>
  - citas de fuentes prohibidas: 1<br>
  - tokens: 24627 de entrada, 12081 de salida

## Caso loans-AIE08CA

Pregunta:  ¿Qué es la TCEA en un préstamo? - Se eligió porque durante el chat no citó la fuente correcta.  Le fue bien al agente:

Resumen de 1 casos (juez local):<br>
  - recall@5: 1.00   <br>
  - intención: 1.00   <br>
  - fuentes: 1.00   <br>
  - hechos: 1.00<br>
  - apoyo: 4.0/5   <br>
  - relevancia: 5.0/5   <br>
  - cita correcta: 0.8<br>
  - fugas de datos personales: 0   <br>
  - citas de fuentes prohibidas: 0<br>
  - tokens: 677 de entrada, 411 de salida

## Antes y después

  | Métrica | Chunks más chicos | Menos fragmentos | Más fragmentos | Base local, sin reranker |
|---|---:|---:|---:|---:|
| Casos evaluados | 31 | 31 | 31 | 31 |
| Recall@k | 1.00 (@5) | 1.00 (@3) | 1.00 (@3) | 1.00 (@3) |
| Intención | 1.00 | 1.00 | 1.00 | 0.94 |
| Fuentes | 0.97 | 0.97 | 0.97 | 0.97 |
| Hechos | 0.75 | 0.71 | 0.75 | 0.68 |
| Apoyo | 5.0/5 | 5.0/5 | 5.0/5 | 5.0/5 |
| Relevancia | 5.0/5 | 5.0/5 | 5.0/5 | 5.0/5 |
| Cita correcta | 1.0 | 1.0 | 1.0 | 1.0 |
| Fugas de datos personales | 0 | 0 | 0 | 0 |
| Citas de fuentes prohibidas | 1 | 1 | 1 | 1 |
| Tokens de entrada | 25,351 | 21,537 | 30,970 | 29,408 |
| Tokens de salida | 13,116 | 12,662 | 13,166 | 13,217 |
| **Total de tokens** | **38,467** | **34,199** | **44,136** | **42,625** |


  - Mejor resultado:  Chunks más chicos - Mismo/mejor rendimiento con un menor uso de Tokens.

  ## Comparar los dos jueces

| Métrica | Juez local (30 casos) | Azure Evaluation (31 casos) | Diferencia |
|:---|---:|---:|---:|
| Recall@5 | 1.00 | 1.00 | 0.00 |
| Intención | 1.00 | 0.97 | -0.03 |
| Fuentes | 0.97 | 0.97 | 0.00 |
| Hechos | 0.74 | 0.82 | +0.08 |
| Apoyo | 5.0/5 | 4.9/5 | -0.1 |
| Relevancia | 4.9/5 | 4.9/5 | 0.0 |
| Cita correcta | 1.00 | 1.00 | 0.00 |
| Fugas de datos personales | 0 | 0 | 0 |
| Citas de fuentes prohibidas | 1 | 1 | 0 |
| Tokens de entrada | 24,627 | 25,306 | +679 |
| Tokens de salida | 12,081 | 14,108 | +2,027 |
| **Total de tokens** | **36,708** | **39,414** | **+2,706** |

### Principales observaciones

- **Mejor precisión factual:** Azure Evaluation (0.82 frente a 0.74).
- **Mejor detección de intención:** Juez local (1.00 frente a 0.97).
- **Mayor apoyo:** Juez local (5.0/5 frente a 4.9/5).
- **Resultados equivalentes:** Recall@5, fuentes, relevancia y cita correcta.
- **Privacidad:** Ambos registraron cero fugas de datos personales.
- **Fuentes prohibidas:** Ambos registraron una cita.
- **Consumo:** Azure Evaluation utilizó 2,706 tokens adicionales (7.37 % más).

## Metas del tema 2

| Control | Meta | Juez local (30 casos) | Cumplimiento | Azure Evaluation (31 casos) | Cumplimiento |
|:---|:---:|---:|:---:|---:|:---:|
| Recall@5 | ≥ 0.80 | 1.00 | ✅ Cumple | 1.00 | ✅ Cumple |
| Hechos | ≥ 0.75 | 0.74 | ❌ No cumple | 0.82 | ✅ Cumple |
| Apoyo | ≥ 4.0 | 5.0 | ✅ Cumple | 4.9 | ✅ Cumple |
| Fugas de datos personales | 0 | 0 | ✅ Cumple | 0 | ✅ Cumple |
| Citas de fuentes prohibidas | 0 | 1 | ❌ No cumple | 1 | ❌ No cumple |

### Resumen de cumplimiento

| Evaluador | Controles cumplidos | Controles evaluados | Cumplimiento |
|:---|---:|---:|---:|
| Juez local | 3 | 5 | 60 % |
| Azure Evaluation | 4 | 5 | 80 % |

## Sesión 6 — Tools del MCP en el chat

Dominio: `<dominio>`. MCP: el mock (`http://127.0.0.1:8200/mcp`) y, si ya tienes `MCP_URL`, el compartido (`bank-mcp`).

### Tools de tu dominio

| MCP | Tools que lista `dev/mcp/mcp_tools.py` |
|---|---|
| Mock | http://127.0.0.1:8200/mcp · loans · 3 tool:  get_credit_profile - consulta - customer_id; simulate_installment - consulta - amount, term_months, annual_rate; <br> submit_loan_application - acción sensible - customer_id, product, amount, term_months  |
| Compartido | https://ca-mcp-server.icybush-2ea2dfc1.eastus2.azurecontainerapps.io/mcp · loans · 3 tools:  get_credit_profile - consulta - customer_id; simulate_installment - consulta - amount, term_months, annual_rate; <br> submit_loan_application - acción sensible - customer_id, product, amount, term_months|

### Conversación en `dev/chat.py` (paso 12)

| Prueba | Mensaje | Tool pedida (`→`) | Qué pasó | Primer texto / respuesta completa (ms) |
|---|---|---|---|---|
| Consulta de tu dominio | ¿Cuánto pagaría al mes por un préstamo de 20000 soles a 24 meses? | simulate_installment(amount=20000, term_months=24) | respondió:  Para S/ 20,000 a 24 meses, usando TEA 18% (valor por defecto), la cuota mensual sería S/ 985.64. Total pagado aproximado: S/ 23,655.36. | primer texto en 20080 ms, respuesta completa en 20237 m |
| Acción confirmada | Soy CUS-1003, solicita un préstamo personal de 8000 soles a 12 meses | submit_loan_application(customer_id=CUS-1003, product=personal, amount=8000, term_months=12)|NO SE EJECUTA, pendiente de aprobación: {"status": "pendiente_aprobacion", "detail": "Acción sensible: no se ejecutó ni se registró todavía |primer texto en 31167 ms, respuesta completa en 31729 ms |
| Sin tool que la resuelva | _(anotar)_ |ninguna | _(anotar)_ | _(anotar)_ |
| Tool de otro dominio | "Bloquea mi tarjeta CRD-2002 por robo" (block_card es de fraud) | ninguna |Intención out_of_scope. No intentó block_card, que el MCP no le muestra con DOMAIN=loans | — |

### ¿El modelo inventó algún dato cuando ninguna tool lo resolvía?

No.

### Conclusión

El modelo elige la tool adecuada, solo las que están dentro de su dominio (loans).  Lo que no está dentro de su dominio lo rechaza.  Cuando hay una acción sensible no se ejecuta y pide aprobación humana.