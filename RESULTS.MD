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
