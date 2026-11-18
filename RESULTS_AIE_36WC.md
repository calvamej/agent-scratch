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
