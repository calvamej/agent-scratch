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