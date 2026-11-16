# Resultados de las cinco preguntas con SEARCH_TOP_K=5 (dominio `loans`)

Comparación de `KNOWLEDGE_BASE=local` (Qdrant en `data/index/loans`) contra `KNOWLEDGE_BASE=azure_search` (índice `idx-team-04` en `srch-workshop-jsu3`), con `SEARCH_TOP_K=5` en ambos.

## Cómo se corrió

- Corpus: 29 chunks (`INGEST_CHUNK_MAX_CHARS=1500`) con embeddings reales de Foundry, el mismo contenido en el índice local y en el de Azure.
- Modelo: `gpt-5-mini-2025-08-07` (Foundry, Entra ID).
- Cliente: `dev/chat.py` contra un agente propio por combinación (puerto 8101). `KNOWLEDGE_BASE` y `SEARCH_TOP_K` se pasaron como variables de entorno y el `.env` no se modificó (ya trae `SEARCH_TOP_K=5`).
- Una sola corrida por combinación. Las respuestas del modelo varían entre corridas.
- Debajo de cada pregunta se transcribe lo que devolvió el agente: intención, respuesta completa, citas (fuente, sección y `chunk_id`) y consumo. Las respuestas y las citas se generaron desde los eventos crudos que envió el agente (`/chat/stream`), sin editarlos.

## Resumen

| # | Tipo | Fuente esperada | local: fuentes citadas | azure_search: fuentes citadas | ¿Fuente correcta? |
|---|---|---|---|---|---|
| 1 | regla del corpus | `personal-loan-requirements.pdf` > 1. Quién puede solicitarlo | `personal-loan-requirements.pdf` (1. Quién puede solicitarlo) | `personal-loan-requirements.pdf` (1. Quién puede solicitarlo)<br>`loans-faq.md` (¿Qué necesito para un préstamo de S/ 20 000?) | Sí en ambos. Azure agrega `loans-faq.md` como apoyo |
| 2 | seguimiento | la misma fuente de la pregunta 1 | `personal-loan-requirements.pdf` (1. Quién puede solicitarlo) | `personal-loan-requirements.pdf` (1. Quién puede solicitarlo)<br>`personal-loan-requirements.pdf` (2. Documentos) | Sí en ambos. Azure agrega la sección de documentos |
| 3 | término exacto | `loans-glossary.md` > Desgravamen | `loans-glossary.md` (Desgravamen)<br>`mortgage.html` (Seguros obligatorios)<br>`personal-loan-terms.docx` (Seguro de desgravamen) | `loans-glossary.md` (Desgravamen) | Sí en ambos. Local agrega dos fuentes que completan la respuesta |
| 4 | documento vencido | no citar `personal-loan-terms-2023.md`; no existe un documento de 2017 | ninguna | ninguna | Sí en ambos: no citan nada ni el documento vencido |
| 5 | fuera del corpus | `out_of_scope`, sin fuentes | ninguna | ninguna | Sí en ambos: `out_of_scope` sin fuentes |

## Tiempos por turno (ms)

| # | local: primer texto / respuesta completa | azure_search: primer texto / respuesta completa |
|---|---|---|
| 1 | 8057 / 8192 | 5323 / 5454 |
| 2 | 10656 / 10783 | 12699 / 12956 |
| 3 | 6562 / 6700 | 4925 / 5047 |
| 4 | 5051 / 5069 | 4111 / 4123 |
| 5 | 3373 / 3508 | 3569 / 3705 |

## Transcripción de cada turno

### Pregunta 1 (regla del corpus): ¿Quién puede solicitar un préstamo personal?

Sesión: nueva.

#### `local`

- Intención: `product_question`
- Respuesta del agente:

  > Pueden solicitarlo personas de 21 a 70 años con DNI vigente, ingresos mensuales desde S/1 500 y sin deudas vencidas en el sistema financiero.

- Citas:
  1. `personal-loan-requirements.pdf` · Requisitos del préstamo personal > 1. Quién puede solicitarlo (chunk `0205d90f-53e5-5191-8a00-ff8831fb7bdd`)
- Tiempos del cliente: primer texto en 8057 ms, respuesta completa en 8192 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 5089 ms, 769 tokens de entrada, 317 de salida
- Sesión: `08e211ce-b54e-4926-9c39-c0fff6153718`

#### `azure_search`

- Intención: `product_question`
- Respuesta del agente:

  > Pueden solicitarlo personas de 21 a 70 años con DNI vigente, ingresos mensuales desde S/ 1 500 y sin deudas vencidas en el sistema financiero. Además se requieren los documentos según tu tipo de trabajador.

- Citas:
  1. `personal-loan-requirements.pdf` · Requisitos del préstamo personal > 1. Quién puede solicitarlo (chunk `0205d90f-53e5-5191-8a00-ff8831fb7bdd`)
  2. `loans-faq.md` · Preguntas frecuentes sobre préstamos > ¿Qué necesito para un préstamo de S/ 20 000? (chunk `1ec4b3a0-d2b5-5f86-ab81-8dd5f0d54cc0`)
- Tiempos del cliente: primer texto en 5323 ms, respuesta completa en 5454 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 4285 ms, 766 tokens de entrada, 331 de salida
- Sesión: `c417568e-3cf9-446c-bcc0-97d7f8988c3a`

### Pregunta 2 (seguimiento): Es decir, si yo tengo 27 años y tengo ingresos de 4,000 soles, soy apta?

Sesión: misma que la pregunta 1.

#### `local`

- Intención: `product_question`
- Respuesta del agente:

  > Sí. Cumples los requisitos de edad (21–70 años) e ingresos (desde S/1,500). Además debes tener DNI vigente y no tener deudas vencidas en el sistema financiero.

- Citas:
  1. `personal-loan-requirements.pdf` · Requisitos del préstamo personal > 1. Quién puede solicitarlo (chunk `0205d90f-53e5-5191-8a00-ff8831fb7bdd`)
- Tiempos del cliente: primer texto en 10656 ms, respuesta completa en 10783 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 2 llamada, latencia del modelo 9980 ms, 1101 tokens de entrada, 1041 de salida
- Sesión: `08e211ce-b54e-4926-9c39-c0fff6153718`

#### `azure_search`

- Intención: `product_question`
- Respuesta del agente:

  > Sí. Cumples los requisitos básicos: tener entre 21 y 70 años, DNI vigente, ingresos mensuales desde S/ 1 500 y no tener deudas vencidas en el sistema financiero. Además deberás presentar los documentos según tu tipo de trabajador: dependientes (últimas 3 boletas y 6 meses de antigüedad), independientes (declaración anual del último año y 12 meses de actividad) o pensionistas (última boleta de pensión).

- Citas:
  1. `personal-loan-requirements.pdf` · Requisitos del préstamo personal > 1. Quién puede solicitarlo (chunk `0205d90f-53e5-5191-8a00-ff8831fb7bdd`)
  2. `personal-loan-requirements.pdf` · Requisitos del préstamo personal > 2. Documentos (chunk `44b4028b-1694-5a06-8017-2391f811fc38`)
- Tiempos del cliente: primer texto en 12699 ms, respuesta completa en 12956 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 2 llamada, latencia del modelo 11990 ms, 1115 tokens de entrada, 1171 de salida
- Sesión: `c417568e-3cf9-446c-bcc0-97d7f8988c3a`

### Pregunta 3 (término exacto): ¿Qué es el desgravamen?

Sesión: nueva.

#### `local`

- Intención: `product_question`
- Respuesta del agente:

  > Es un seguro que paga la deuda si el titular fallece. En el crédito hipotecario es obligatorio (el cliente puede elegir la aseguradora o endosar su seguro de vida). En el préstamo personal es opcional: puede contratarlo con el banco, endosar uno propio o no contratarlo.

- Citas:
  1. `loans-glossary.md` · Glosario de préstamos > TEA > Desgravamen (chunk `b5ebd5cf-2b8e-5329-9a1c-0ca6bbb3a715`)
  2. `mortgage.html` · Crédito hipotecario > Seguros obligatorios (chunk `65269fa7-ef7a-5803-a5fe-c8c8a328ff87`)
  3. `personal-loan-terms.docx` · Condiciones del préstamo personal > Seguro de desgravamen (chunk `d9620bc5-21e8-58ab-bec7-e4ae6dc88cd2`)
- Tiempos del cliente: primer texto en 6562 ms, respuesta completa en 6700 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 6416 ms, 748 tokens de entrada, 539 de salida
- Sesión: `404d2112-c531-4e58-bf09-4bcd87248f6d`

#### `azure_search`

- Intención: `product_question`
- Respuesta del agente:

  > El desgravamen es un seguro que paga la deuda si el titular fallece.

- Citas:
  1. `loans-glossary.md` · Glosario de préstamos > TEA > Desgravamen (chunk `b5ebd5cf-2b8e-5329-9a1c-0ca6bbb3a715`)
- Tiempos del cliente: primer texto en 4925 ms, respuesta completa en 5047 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 4140 ms, 748 tokens de entrada, 298 de salida
- Sesión: `1d862de1-b96d-4f4b-9a34-c0bae934aeb6`

### Pregunta 4 (documento vencido): ¿Qué decía la política de préstamos personales de 2017?

Sesión: nueva.

#### `local`

- Intención: `product_question`
- Respuesta del agente:

  > No encuentro en los documentos provistos información sobre la política de préstamos personales de 2017. ¿Quieres que busque la política vigente o prefieres información sobre requisitos, tasas o condiciones actuales?

- Citas: ninguna
- Tiempos del cliente: primer texto en 5051 ms, respuesta completa en 5069 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 4406 ms, 754 tokens de entrada, 448 de salida
- Sesión: `72f14a7c-1d3f-49ff-8ac8-a5eea87d87ed`

#### `azure_search`

- Intención: `product_question`
- Respuesta del agente:

  > Lo siento, no dispongo de la política de préstamos personales de 2017 en los documentos proporcionados.

- Citas: ninguna
- Tiempos del cliente: primer texto en 4111 ms, respuesta completa en 4123 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 3650 ms, 768 tokens de entrada, 302 de salida
- Sesión: `46759cbd-8767-424f-84d0-0e2a184f00dc`

### Pregunta 5 (fuera del corpus): ¿Cómo está el clima hoy?

Sesión: nueva.

#### `local`

- Intención: `out_of_scope`
- Respuesta del agente:

  > Lo siento, no puedo proporcionar información del clima. Soy el agente de préstamos del Banco Demo y puedo ayudar con requisitos, condiciones, tasas, seguros y prepagos. ¿En qué tema de préstamos puedo asistirte?

- Citas: ninguna
- Tiempos del cliente: primer texto en 3373 ms, respuesta completa en 3508 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 3093 ms, 793 tokens de entrada, 198 de salida
- Sesión: `d6ed37b3-7a79-4850-9e7b-84ac30c81cda`

#### `azure_search`

- Intención: `out_of_scope`
- Respuesta del agente:

  > Lo siento, ese tema está fuera del alcance del Banco Demo. Soy agente de préstamos y puedo ayudar con requisitos, condiciones, tasas, simulaciones y prepagos. Para consultar el clima, revisa una app meteorológica o busca en internet.

- Citas: ninguna
- Tiempos del cliente: primer texto en 3569 ms, respuesta completa en 3705 ms
- Consumo: modelo `gpt-5-mini-2025-08-07`, 1 llamada, latencia del modelo 3191 ms, 836 tokens de entrada, 203 de salida
- Sesión: `160a55ef-bacf-4835-b1cd-9f27de0bb894`

## Comparación `local` contra `azure_search`

- **Fuentes:** en las cinco preguntas, la fuente principal fue la misma en ambas bases. Cambia cuántas fuentes adicionales cita el modelo:
  - Pregunta 1: Azure agrega `loans-faq.md`. Local cita solo el PDF.
  - Pregunta 2: Azure agrega la sección "2. Documentos" del PDF y su respuesta incluye los documentos por tipo de trabajador. Local responde solo con la sección 1.
  - Pregunta 3: Local cita el glosario, `mortgage.html` y `personal-loan-terms.docx`, y explica que el desgravamen es obligatorio en el hipotecario y opcional en el personal. Azure cita solo el glosario y da la definición en una frase.
  - Preguntas 4 y 5: iguales en ambas.
- **Documento vencido:** `personal-loan-terms-2023.md` no se citó en ninguna base. Ambas respondieron que no tenían la política de 2017.
- **Fuera del corpus:** en ambas, intención `out_of_scope` y sin citas.
- **Qué falta confirmar:** el modelo decide cuántas fuentes cita, y con una corrida por combinación no se puede atribuir la diferencia a la base de búsqueda y no al modelo.
- **Tiempos:** no hay diferencia sostenida entre bases. Local fue más lenta en las preguntas 1, 3 y 4, y Azure en la 2 y la 5. La pregunta 2 es la más lenta en las dos (de 10 a 13 s) porque la respuesta es más larga.
- **Primer texto contra respuesta completa:** casi coinciden (la diferencia es de 12 a 257 ms). El tiempo lo domina la llamada al modelo y el texto llega casi de una vez, en 17 a 100 eventos.
- **Tiempos del cliente contra latencia del modelo:** el cliente mide más que la latencia que reporta el agente (por ejemplo, 5454 ms contra 4285 ms en la pregunta 1 de Azure). La diferencia corresponde al tiempo fuera de la llamada al modelo. No se midió qué parte es la búsqueda.


# Línea base (S5)

Resumen de 30 casos (juez local):

```
recall@5: 1.00   intención: 1.00   fuentes: 0.97   hechos: 0.81
apoyo: 5.0/5   relevancia: 5.0/5   cita correcta: 1.0
fugas de datos personales: 0   citas de fuentes prohibidas: 1
tokens: 24637 de entrada, 12976 de salida
```

## Mi caso del golden set (S5)

```json
{"id": "loans-AIE04MA", "type": "answerable", "history": [], "question": "En el crédito hipotecario, ¿puedo elegir mi propia aseguradora para el desgravamen?", "expected_intent": "product_question", "expected_sources": ["mortgage.html"], "forbidden_sources": [], "key_facts": ["endosar su seguro de vida"]}
```

### Por qué lo elegí

- Los 30 casos base no cubren esta pregunta. Los casos de `mortgage.html` (`loans-12`, `13`, `14`, `18`) preguntan por el porcentaje de financiamiento, el plazo, qué seguros pide y la cuota máxima. Ninguno pregunta quién escoge la aseguradora.
- Es una duda real de un cliente: tras saber que el desgravamen es obligatorio en el hipotecario, quiere saber si puede contratarlo donde quiera.
- Mezcla dos temas cercanos: el desgravamen es obligatorio en el hipotecario y opcional en el préstamo personal. Sirve para ver si el agente confunde ambos productos. En la corrida de S4 (pregunta 3) el agente ya citó `personal-loan-terms.docx` y `mortgage.html` juntos al hablar de desgravamen.
- La respuesta está en una sola frase de `mortgage.html` (sección "Seguros obligatorios"), así que se puede verificar con el dato `endosar su seguro de vida`, copiado tal cual del documento.

### Cómo le fue al agente

Resumen de 1 caso (juez local):

```
recall@5: 1.00   intención: 1.00   fuentes: 1.00   hechos: 1.00
apoyo: 5.0/5   relevancia: 5.0/5   cita correcta: 1.0
fugas de datos personales: 0   citas de fuentes prohibidas: 0
tokens: 812 de entrada, 504 de salida
```

- **Respuesta del agente:** "Sí. En el crédito hipotecario el seguro de desgravamen es obligatorio, pero puede elegir la aseguradora o endosar su seguro de vida."
- **Cita:** solo `mortgage.html`, la fuente esperada.
- **Búsqueda:** `mortgage.html` quedó primero entre los 5 fragmentos recuperados. Los otros cuatro fueron `loans-faq.md`, `personal-loan-terms.docx` (dos fragmentos) y `personal-loan-requirements.pdf`. El agente no los citó y no mezcló la regla del préstamo personal.
- **Juez:** la respuesta reproduce el fragmento de `mortgage.html` y responde la pregunta. Una llamada al modelo (`gpt-5-mini-2025-08-07`), 5359 ms de latencia.
- **Límite:** es un caso fácil. La respuesta está en una sola frase y la reproduce casi literal, así que sirve para detectar regresiones y no para medir mejoras. Con una sola corrida no se sabe si el modelo siempre conserva la frase `endosar su seguro de vida`.

## Cambios de uno en uno (S5)

Cada fila cambia una sola cosa respecto de la corrida de referencia y se evalúa con `uv run eval` completo (31 casos: los 30 base y el mío, juez `local`, `azure_search`, `gpt-5-mini-2025-08-07`). Una corrida por fila.

| # | Cambio | Cómo | recall | intención | fuentes | hechos | apoyo | relevancia | cita | fugas | prohibidas | tokens (entrada / salida) | Detalle |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 0 | referencia | `INGEST_CHUNK_MAX_CHARS=1500`, `SEARCH_TOP_K=5`, `azure_search` | 1.00 | 1.00 | 0.97 | 0.71 | 5.0 | 5.0 | 1.0 | 0 | 1 | 25400 / 13419 | `loans-20261115-225028.json` |
| 1 | menos fragmentos | `SEARCH_TOP_K=3` | 1.00 | 1.00 | 0.97 | 0.71 | 5.0 | 5.0 | 1.0 | 0 | 1 | 21532 / 12857 | `loans-20261115-225650.json` |
| 2 | más fragmentos | `SEARCH_TOP_K=8` | 1.00 | 1.00 | 0.97 | 0.79 | 5.0 | 5.0 | 1.0 | 0 | 1 | 31080 / 13298 | `loans-20261115-230333.json` |
| 3 | base local, sin reranker | `uv run ingest publish --target local` y `KNOWLEDGE_BASE=local` | 0.93 | 0.97 | 0.87 | 0.57 | 4.9 | 4.9 | 0.98 | 0 | 1 | 24233 / 14549 | `loans-20261115-231316.json` |
| 4 | chunks más chicos | `INGEST_CHUNK_MAX_CHARS=800`, `uv run ingest build` y `uv run ingest publish` | 1.00 | 1.00 | 0.97 | 0.68 | 5.0 | 5.0 | 1.0 | 0 | 1 | 25479 / 12805 | `loans-20261115-232040.json` |

El recall se mide en @k, con el k de cada fila (@5, @3, @8).

La referencia da hechos 0.71 y la línea base de las 30 preguntas dio 0.81. La diferencia viene de mi caso (que ahora falla en hechos) y de la variación entre corridas, porque cambió el conjunto de casos y no la configuración.

### Qué se vio en cada cambio

- **Menos fragmentos (`SEARCH_TOP_K=3`):** recall, fuentes y hechos quedan igual que con 5. Los tokens de entrada bajan de 25400 a 21532 (-15 %). No se perdió ningún documento esperado entre los 3 primeros.
- **Más fragmentos (`SEARCH_TOP_K=8`):** apoyo igual (5.0) y la cita prohibida de `loans-25` sigue (1, igual que con 5). Hechos sube a 0.79 y los tokens de entrada suben a 31080 (+22 %). El aumento en hechos no se puede atribuir a los fragmentos extra: los casos que fallan siguen siendo de formato (ver abajo) y la misma configuración varía entre 0.68 y 0.79 de una corrida a otra.
- **Base local, sin reranker:** es el único cambio que empeora de forma clara. Recall baja de 1.00 a 0.93, fuentes de 0.97 a 0.87 y hechos de 0.71 a 0.57. Fallan sobre todo las preguntas que dependen de números en `sample-simulations.csv`: en `loans-11` y `loans-19` el CSV no aparece entre los 5 resultados y el agente responde con una cuota aproximada o dice que no está en los fragmentos; en `loans-11` la intención sale como `data_query`. El reranker de AI Search sí aporta aquí. Con un corpus de 29 chunks, la diferencia viene de la calidad de la búsqueda local y no de que falte contexto.
- **Chunks de 800:** no cambia nada. `ingest build` generó los mismos 29 chunks con el mismo contenido que con 1500, porque el chunk más largo mide 304 caracteres. Las diferencias de la fila (hechos 0.68) son variación entre corridas, no efecto del tamaño. El tamaño de chunk solo importaría con documentos más largos que el corpus actual.

### Casos que fallan en todas las filas

- `loans-25` (noise): cita `internal-notice-training.html`. Es la cita prohibida (1) en todas las filas, también en la base local. No la mueve ni el top-k ni la base: es un problema del filtro de ruido o del prompt, no de la búsqueda.
- Hechos: casi todos los NO son de formato (`14.9 %` contra `14.9%`, `S/ 1 500` contra `S/1,500`, "21 a 70 años" contra "entre 21 y 70 años"). Los casos concretos cambian de una corrida a otra, y eso explica la variación de hechos.

### Qué elijo yo y por qué

Cada integrante del equipo (somos 4) elige según su propio análisis. Esta es mi elección y no una decisión del equipo.

**Elijo: `SEARCH_TOP_K=3`, `INGEST_CHUNK_MAX_CHARS=1500`, `KNOWLEDGE_BASE=azure_search` (con el reranker).**

- Me quedo con `azure_search`: sin el reranker bajan recall, fuentes y hechos.
- Elijo `SEARCH_TOP_K=3` porque da los mismos resultados que 5 con 15 % menos de tokens de entrada. Repetí esa configuración al final: recall@3 1.00, fuentes 0.97, hechos 0.71, apoyo 5.0, 21578 tokens de entrada. Con 8 solo suben los tokens.
- Me quedo con 1500 en chunks: 800 no cambia nada con este corpus.
- **Límite:** con 31 casos y una corrida por fila, diferencias de hechos de 0.1 caben dentro de la variación entre corridas. Que 3 y 5 den lo mismo en este golden set no garantiza que pase con un corpus más grande. Si se agregan documentos largos, hay que volver a evaluar. Esta configuración tampoco resuelve `loans-25` ni el formato de los hechos, que se arreglan con el prompt y el filtro, no con la configuración.

### Estado al terminar

- Variables de shell `INGEST_CHUNK_MAX_CHARS`, `SEARCH_TOP_K` y `KNOWLEDGE_BASE` borradas (`unset`).
- `.env` queda con `INGEST_CHUNK_MAX_CHARS=1500`, `KNOWLEDGE_BASE=azure_search` y `SEARCH_TOP_K=3`.
- Índice `idx-team-04` republicado con `INGEST_CHUNK_MAX_CHARS=1500`: 29 chunks, el mismo contenido que antes de las pruebas.
- El índice local de `data/index/loans` quedó con los chunks de 1500 (publicados antes de la prueba de 800).
