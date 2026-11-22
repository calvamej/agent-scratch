# Resultados AIE 36WC

## Sesión 4: retrieval híbrido y respuestas con sources

Dominio `loans` (30 chunks), Foundry real. Las cinco preguntas se hicieron en una sola conversación y en este orden, en cada corrida.

### Paso 11: cinco preguntas contra Foundry (base local)

`KNOWLEDGE_BASE=local`, `SEARCH_TOP_K=5`.

| # | Tipo | Pregunta | Fuentes que citó | ¿Fuente correcta? | Primer texto en | Respuesta completa en |
|---|---|---|---|---|---|---|
| 1 | Regla del corpus | ¿Cuál es el monto máximo de un préstamo personal? | personal-loan-requirements.pdf › 3. Montos y plazos | Sí (S/ 150 000) | 4859 ms | 4993 ms |
| 2 | Seguimiento | ¿Y en cuántos meses lo puedo pagar? | personal-loan-requirements.pdf › 3. Montos y plazos | Sí (6 a 60 meses): el rewrite entendió «lo» | 11103 ms | 11113 ms |
| 3 | Término exacto | ¿Qué es la TCEA? | loans-glossary.md › TCEA; personal-loan-terms.docx › Tasas | Sí | 8697 ms | 8835 ms |
| 4 | Documento vencido | ¿Es obligatorio el seguro de desgravamen en el préstamo personal? | loans-faq.md › ¿Es obligatorio el seguro de desgravamen?; personal-loan-terms.docx › Seguro de desgravamen | Sí: respondió «opcional» y no citó personal-loan-terms-2023.md (vencido) | 10892 ms | 11021 ms |
| 5 | Fuera del corpus | ¿Cuánto cuesta el mantenimiento de una tarjeta de crédito? | Ninguna (`out_of_scope`) | Sí: dijo que no tiene esa información | 10395 ms | 10568 ms |

### Paso 12: SEARCH_TOP_K

Tokens de entrada de cada turno (incluyen el historial y el rewrite, por eso crecen turno a turno):

| SEARCH_TOP_K | P1 | P2 | P3 | P4 | P5 | Promedio |
|---|---|---|---|---|---|---|
| 2 | 617 | 833 | 936 | 1026 | 1162 | 915 |
| 5 | 772 | 1050 | 1124 | 1319 | 1452 | 1143 |
| 10 | 1088 | 1320 | 1356 | 1583 | 1670 | 1403 |

**Con 2: ¿falta información?** Sí, en la pregunta 3: no llegó el glosario, así que no definió la TCEA (solo dijo qué incluye, según personal-loan-terms.docx). En la pregunta 4 la respuesta fue correcta, pero citó personal-loan-requirements.pdf › 3. Montos y plazos, que no viene al caso, en vez de personal-loan-terms.docx.

**Con 10: ¿cuánto más cuesta?** Unos 260 tokens de entrada más por turno frente a 5 (+23 %), y unos 490 más frente a 2. **¿Fuentes que no vienen al caso?** No: citó exactamente las mismas fuentes que con 5, y las respuestas fueron iguales.

**Valor elegido:** 5 (propuesta, por confirmar con el equipo).

**Por qué:** con 2 se pierde información (pregunta 3); con 10 se paga más por turno sin ninguna mejora en fuentes ni en respuestas.

### Paso 13: local frente a Azure AI Search

`SEARCH_TOP_K=5` en ambas. AI Search: índice `idx-team-04` (30 chunks).

| Pregunta | Fuentes en local | Fuentes en AI Search | ¿Mejoró con el reranker? | Respuesta completa en (local / AI Search) |
|---|---|---|---|---|
| 1. ¿Cuál es el monto máximo de un préstamo personal? | requirements.pdf › 3. Montos y plazos | requirements.pdf › 3. Montos y plazos | No: igual | 4993 ms / 7617 ms |
| 2. ¿Y en cuántos meses lo puedo pagar? | requirements.pdf › 3. Montos y plazos | requirements.pdf › 3. Montos y plazos | No: igual | 11113 ms / 9370 ms |
| 3. ¿Qué es la TCEA? | glossary › TCEA; terms.docx › Tasas | terms.docx › Tasas; glossary › TCEA | No: mismas fuentes, en otro orden | 8835 ms / 9832 ms |
| 4. ¿Es obligatorio el seguro de desgravamen…? | faq › ¿Es obligatorio…?; terms.docx › Seguro de desgravamen | terms.docx › Seguro de desgravamen; faq › ¿Es obligatorio…? | No: mismas fuentes, en otro orden. Ninguna citó el documento vencido | 11021 ms / 10713 ms |
| 5. ¿Cuánto cuesta el mantenimiento de una tarjeta de crédito? | Ninguna | Ninguna | No: igual | 10568 ms / 10130 ms |

**Conclusión:** con este corpus (30 chunks cortos), el reranker no cambió las fuentes citadas; solo el orden en las preguntas 3 y 4. Los tiempos son parecidos: los domina la respuesta del modelo, no la búsqueda.

## Sesión 5: calidad del RAG

### Línea base (S5)

Dominio `loans`, Foundry real, AI Search (índice `idx-team-04`), `SEARCH_TOP_K=5`, juez local. 31 casos: los 30 del golden set más el mío.

| Métrica | Valor |
|---|---|
| recall@5 | 1.00 |
| intención | 1.00 |
| fuentes | 0.97 |
| hechos | 0.75 |
| apoyo | 5.0/5 |
| relevancia | 5.0/5 |
| cita correcta | 1.0 |
| fugas de datos personales | 0 |
| citas de fuentes prohibidas | 1 |
| tokens | 25 275 de entrada, 13 566 de salida |

Detalle de cada caso: `data/eval/loans-20261121-171432.json`.

### Casos con NO

| Casos | Qué falló | Causa | Qué se mejoraría |
|---|---|---|---|
| loans-01, 02, 04, 15, 22, 24, 30 | hechos (recall ok) | La respuesta es correcta (apoyo 5/5), pero no coincide letra por letra con el `key_fact`: el modelo escribe «S/1,500», «14.9%», «desde 6 hasta 60», «paga (cancela) la deuda», «una boleta»; el golden set espera «S/ 1 500», «14.9 %», «6 a 60», «paga la deuda», «1 boleta». La comparación solo quita tildes y mayúsculas. *(Deducción a partir del código y de las respuestas.)* | Según el paso 9 (recall ok y hechos NO): prompt o contexto, por ejemplo, pedir que copie cifras y frases tal cual las dice el documento. |
| loans-25 (noise) | fuentes, cita prohibida | Citó `internal-notice-training.html` (aviso interno) y respondió «El curso debe completarse antes del fin de mes». Debía tratarla como `out_of_scope`. | Según el paso 9: revisar el filtro de vigencia y los fragmentos. |

### Mi caso del golden set

Archivo `evals/golden/loans.AIE_36WC.jsonl`, caso `loans-AIE36WC`:

- **Pregunta:** ¿Qué es la TCEA de un préstamo?
- **Documento esperado:** `loans-glossary.md`. **Dato clave:** «comisiones y seguros».
- **Por qué la elegí:** el golden set pregunta por la TEA, pero no por la TCEA, que es una duda común de un cliente real. Además, en la sesión 4 (paso 12) esta pregunta falló con `SEARCH_TOP_K=2`, porque no llegó el glosario.
- **Cómo le fue:** todo ok (recall, intención, fuentes, hechos; apoyo 5.0, relevancia 5.0, cita 1.0). Citó `loans-glossary.md` y además `personal-loan-terms.docx`.

### Paso 10: antes y después

Un cambio por corrida, 31 casos cada una, juez local. La base es la línea base (S5): chunks de 250 (`INGEST_CHUNK_MAX_CHARS=250`), `SEARCH_TOP_K=5` y AI Search con reranker.

| Corrida | recall@5 | intención | fuentes | hechos | apoyo | relevancia | cita | prohibidas | fugas | tokens (entrada / salida) | Detalle |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Base | 1.00 | 1.00 | 0.97 | 0.75 | 5.0 | 5.0 | 1.00 | 1 | 0 | 25 275 / 13 566 | `loans-20261121-171432.json` |
| 1. Chunks de 800 | 1.00 | 1.00 | 0.97 | 0.68 | 4.96 | 5.0 | 1.00 | 1 | 0 | 25 348 / 13 346 | `loans-20261121-193204.json` |
| 2. `SEARCH_TOP_K=3` | 1.00 | 1.00 | 0.97 | 0.68 | 5.0 | 5.0 | 0.99 | 1 | 0 | 21 432 / 12 598 | `loans-20261121-174548.json` |
| 3. `SEARCH_TOP_K=8` | 1.00 | 1.00 | 0.97 | 0.75 | 5.0 | 5.0 | 1.00 | 1 | 0 | 30 711 / 13 522 | `loans-20261121-174604.json` |
| 4. Base local, sin reranker | 0.93 | 0.94 | 0.90 | 0.61 | 4.82 | 4.89 | 0.96 | 1 | 0 | 24 091 / 13 746 | `loans-20261121-174556.json` |

Los archivos de detalle están en `data/eval/`.

- **Chunks de 800:** ojo, en este repo los chunks ya eran de 250, así que con 800 quedaron más grandes, no más chicos. No mejoró nada: mismo recall y hechos un poco más bajos.
- **Top 3:** un 15 % menos de tokens de entrada, pero los hechos bajan de 0.75 a 0.68.
- **Top 8:** mismo resultado que la base, con un 21 % más de tokens de entrada. Las citas prohibidas no subieron (siguen en 1: loans-25).
- **Local, sin reranker:** empeora en todo (recall 0.93, apoyo 4.82). El reranker de AI Search sí aporta.

La diferencia de hechos entre 0.75 y 0.68 son unos 2 casos. Como la comparación de hechos es casi literal, puede ser ruido. *(Deducción.)*

**Configuración elegida:** chunks de 250, `SEARCH_TOP_K=5`, AI Search con reranker; es decir, la actual (propuesta, por confirmar con el equipo).

**Por qué:** ningún cambio mejora la base. Top 3 pierde hechos, top 8 cuesta más sin ganar nada y la base local pierde recall y apoyo. Mi índice en AI Search quedó otra vez con los chunks de 250.

### Paso 11: comparar los dos jueces

Misma configuración que la base (chunks de 250, `SEARCH_TOP_K=5`, AI Search), 31 casos. Las dos corridas se lanzaron al mismo tiempo, en terminales distintas, y se midieron con `time`.

| | decision-1 (`--judge azure_evaluation`) | G-Eval (`--judge local`) |
|---|---|---|
| Detalle | `loans-20261121-194111.json` | `loans-20261121-194319.json` |
| Tiempo total | 3:48 | 5:48 |
| apoyo | 4.95 | 5.0 |
| cita correcta | 0.99 | 1.0 |
| Cómo da la nota | Decimales (probabilidades), sin explicación | Enteros, con `reasoning` |

**¿Las notas de apoyo y de cita correcta van en la misma dirección?** Sí. En 26 de los 28 casos con nota, decision-1 da casi 5 en apoyo y casi 1 en cita (por ejemplo, 4.99 y 0.999), igual que el 5 y el 1 de G-Eval.

**¿Cuánto tardó cada corrida?** decision-1, 3:48; G-Eval, 5:48. decision-1 tardó un 35 % menos.

**Casos donde no coinciden:**

| Caso | decision-1 | G-Eval | Por qué |
|---|---|---|---|
| loans-19 (follow_up) | cita 0.78 | cita 1.0 | En la corrida de decision-1, la respuesta dice que «la cuota exacta la calcula el simulador», pero solo citó `sample-simulations.csv`; esa frase viene de `installment-formula.md`, que no citó. En la corrida de G-Eval citó los dos, y su `reasoning` ubica cada dato en su documento. |
| loans-AIE36WC (mi caso) | apoyo 4.06 | apoyo 5.0 | En la corrida de decision-1, la respuesta dice «En nuestro banco la TCEA considera…», una frase que no está en los documentos. El `reasoning` de G-Eval dice que, en su corrida, todas las afirmaciones estaban en las fuentes. |

Ojo: cada corrida vuelve a llamar al agente, así que los jueces no calificaron exactamente las mismas respuestas. Las dos diferencias salen de respuestas distintas, no de que los jueces opinen distinto. *(Deducción a partir de las respuestas guardadas en cada archivo.)*

**Conclusión:** decision-1 es más rápido y da probabilidades, así que sirve para frenar un despliegue (sesión 11). G-Eval tarda más, pero su `reasoning` explica la nota, así que sirve para entender por qué falló un caso.
