

---

```markdown
# Fase 1 — Núcleo del pipeline (sin Slack, sin OCI, sin Trello)

Documento de trabajo para el equipo. Define el alcance, los contratos de datos
y el criterio de aceptación de la Fase 1.

**Regla general:** ningún nodo improvisa su formato. Cada nodo respeta el contrato
de su entrada y produce el contrato de su salida.

---

## 1. Alcance de la Fase 1

Construir y validar el núcleo semántico del pipeline sobre n8n:

```text
JSON de prueba
↓
Normalización
↓
Loop Over Items
↓
Groq — análisis semántico
↓
Parse + Scoring
↓
Switch
↓
Generación de activos
↓
Merge
↓
JSON final
```

**Fuera de alcance en esta fase:** OCI Object Storage, Slack, Webhook Callback,
Trello, DataTables, Streamlit.

**Criterio de avance:** no se pasa a la Fase 2 hasta que la Fase 1 produzca el
JSON final correctamente contra el contrato.

---

## 2. Contrato 1 — Trigger → Normalización

**Entrada (lo que llega al trigger):**

```json
{
  "origen_comunidad": "Discord_Grupo_ONE_G10",
  "periodo_referencia": "Semana_04",
  "interacciones": [
    {
      "autor": "string",
      "canal": "string",
      "tipo": "testimonio | pregunta_tecnica | logro | otro",
      "texto": "string"
    }
  ]
}
```

**Contrato:** array de interacciones con al menos `autor`, `canal`, `texto`.
`tipo` es opcional.

---

## 3. Contrato 2 — Normalización → Loop Over Items

**Salida:**

```json
{
  "execution_id": "CL-2026-001",
  "interaction_id": "int-001",
  "origen_comunidad": "Discord_Grupo_ONE_G10",
  "periodo_referencia": "Semana_04",
  "autor": "Mariana Souza",
  "canal": "#logros-y-empleos",
  "tipo_original": "testimonio",
  "texto": "string",
  "timestamp_ingesta": "2026-09-21T15:30:00Z"
}
```

**Contrato:** un item por interacción. Cada uno con `interaction_id` único.

**Nota técnica:** en versiones actuales de n8n el nodo se llama `Loop Over Items`.
Si tu instancia muestra `Split In Batches`, es funcionalmente equivalente.

**Nota de diseño:** el loop procesa una interacción por iteración. Alternativa
para volúmenes grandes: enviar el batch completo al LLM en una sola llamada
con un array en el prompt y pedir un array JSON de salida. Para el alcance del
hackathon se mantiene el loop individual.

---

## 4. Contrato 3 — Loop → Groq LLM (análisis semántico)

**Prompt de entrada:** se construye con los campos del item anterior.

**Salida del LLM (JSON estricto):**

```json
{
  "sentimiento": "muy_positivo | positivo | neutro | negativo",
  "temas": ["string"],
  "tipo_detectado": "testimonio | pregunta_tecnica | logro | otro",
  "score_llm": 0,
  "insight": "string"
}
```

**Contrato:** exactamente estos 5 campos. El system prompt fuerza el formato.
`score_llm` es la propuesta del LLM; no es la decisión final.

**Convención de valores:** todos los valores categóricos van en minúscula con
guion bajo (`snake_case`). La capitalización para presentación humana ocurre
únicamente en la capa de generación de activos, nunca en los datos del pipeline.

**System prompt recomendado (base):**

```text
Eres un analista semántico de comunidades tecnológicas. Analiza la
interacción y responde ÚNICAMENTE con un JSON válido con exactamente
estos campos: sentimiento, temas, tipo_detectado, score_llm, insight.

- sentimiento: uno de muy_positivo, positivo, neutro, negativo.
- temas: array de strings con los temas principales (máximo 3).
- tipo_detectado: uno de testimonio, pregunta_tecnica, logro, otro.
- score_llm: número entre 0 y 10 (valoración de valor para marketing).
- insight: una frase corta con el hallazgo clave.

No incluyas texto fuera del JSON. No uses markdown.
```

---

## 5. Contrato 4 — Groq → Parse + Scoring (decisión determinística)

**Entrada:** respuesta cruda del LLM.

**Salida:**

```json
{
  "execution_id": "CL-2026-001",
  "interaction_id": "int-001",
  "autor": "string",
  "canal": "string",
  "texto_original": "string",
  "sentimiento": "muy_positivo",
  "temas": ["string"],
  "tipo_detectado": "testimonio",
  "score_llm": 9.0,
  "score_final": 7.4,
  "insight": "string"
}
```

**Fórmula del `score_final`:**

```text
score_final = 0.8 × score_llm + 0.1 × señal_de_tipo + 0.1 × señal_de_recurrencia
```

Donde:

- `score_llm` → propuesta del modelo (0–10).
- `señal_de_tipo` → peso determinístico por tipo detectado:

| tipo_detectado   | señal_de_tipo |
| ---------------- | ------------- |
| testimonio       | 2.0           |
| logro            | 1.5           |
| pregunta_tecnica | 1.5           |
| otro             | 0.5           |

- `señal_de_recurrencia` → 1.0 si el tema ya apareció en la misma ejecución,
  0.0 si no.

**Verificación del ejemplo:**

```text
0.8 × 9.0 + 0.1 × 2.0 + 0.1 × 0.0 = 7.2 + 0.2 + 0.0 = 7.4
```

**Rango real de salida:**

```text
Máximo: 0.8 × 10.0 + 0.1 × 2.0 + 0.1 × 1.0 = 8.3
Mínimo: 0.8 × 0.0 + 0.1 × 0.5 + 0.1 × 0.0 = 0.05
```

El clamp al rango 0–10 es defensivo y solo tendría efecto si en el futuro se
modifican los pesos.

**Contrato:** el LLM propone; el sistema decide. `score_final` es el que se usa
para todo lo que sigue.

---

## 6. Contrato 5 — Parse + Scoring → Switch

**Regla de bifurcación:**

| Condición                                                                                                         | Rama                |
| ----------------------------------------------------------------------------------------------------------------- | ------------------- |
| `tipo_detectado == "testimonio"` AND `sentimiento` en `["muy_positivo", "positivo"]` AND `score_final >= 6`       | Rama A → LinkedIn   |
| `tipo_detectado == "pregunta_tecnica"` AND `sentimiento` en `["muy_positivo", "positivo"]` AND `score_final >= 6` | Rama B → FAQ        |
| `tipo_detectado == "logro"` AND `sentimiento` en `["muy_positivo", "positivo"]` AND `score_final >= 6`            | Rama C → Newsletter |
| Cualquier otro caso                                                                                               | Descartar           |

**Contrato:** cada tipo tiene una única ruta. No hay solapamiento. El filtro de
sentimiento aplica de forma simétrica en las tres ramas: contenido neutral o
negativo se descarta sin importar el tipo.

---

## 7. Contrato 6 — Switch → Generación de activos

### Rama A — LinkedIn

**Entrada al LLM:**

```json
{
  "autor": "string",
  "texto": "string",
  "insight": "string",
  "temas": ["string"]
}
```

**Salida:**

```json
{
  "titulo": "string",
  "copy": "string",
  "canal_recomendado": "LinkedIn Oficial",
  "potencial_engagement": "Alto | Medio | Bajo"
}
```

### Rama B — FAQ

**Entrada al LLM:**

```json
{
  "autor": "string",
  "texto": "string",
  "temas": ["string"]
}
```

**Salida:**

```json
{
  "tema": "string",
  "pregunta": "string",
  "respuesta": "string",
  "origen": "string",
  "status": "listo_para_revision"
}
```

### Rama C — Newsletter

**Entrada al LLM:**

```json
{
  "autor": "string",
  "texto": "string",
  "insight": "string"
}
```

**Salida:**

```json
{
  "seccion": "Logro de la Semana",
  "titular": "string",
  "resumen": "string"
}
```

**Contrato:** cada rama tiene su propio prompt y su propio formato. Todos
respetan el brief.

---

## 8. Contrato 7 — Generación → Merge

**Salida del Merge:**

```json
{
  "execution_id": "CL-2026-001",
  "interaction_id": "int-001",
  "asset_id": "asset-001",
  "autor": "string",
  "tipo_activo": "linkedin | faq | newsletter",
  "contenido_generado": {},
  "score_final": 7.4,
  "temas": ["string"],
  "insight": "string"
}
```

**Contrato clave:** el `asset_id` se genera en el Merge. Todo el recorrido
queda trazable desde `execution_id` + `interaction_id` + `asset_id`.

---

## 9. JSON de prueba de entrada (Fase 1)

```json
{
  "origen_comunidad": "Discord_Grupo_ONE_G10",
  "periodo_referencia": "Semana_04",
  "interacciones": [
    {
      "autor": "Mariana Souza",
      "canal": "#logros-y-empleos",
      "tipo": "testimonio",
      "texto": "Después de 8 meses aplicando lo que aprendí en el programa, firmé mi primera oferta como desarrolladora junior. Gracias a toda la comunidad por el acompañamiento."
    },
    {
      "autor": "Carlos Peña",
      "canal": "#ayuda-tecnica",
      "tipo": "pregunta_tecnica",
      "texto": "¿Alguien tiene un ejemplo de despliegue continuo en Oracle Cloud con n8n? Encontré una guía muy buena, pero quiero ajustar bien los permisos IAM. Cualquier tip es bienvenido."
    },
    {
      "autor": "Lucía Fernández",
      "canal": "#logros-y-empleos",
      "tipo": "logro",
      "texto": "Mi proyecto final fue seleccionado entre los 10 mejores del hackathon regional. Lo construimos en equipo con todo lo del módulo de APIs."
    },
    {
      "autor": "Andrés Ríos",
      "canal": "#general",
      "tipo": "otro",
      "texto": "Hola, ¿qué día es la próxima reunión? No encuentro el calendario."
    }
  ]
}
```

**Casos cubiertos:** testimonio positivo (Rama A), pregunta técnica (Rama B),
logro (Rama C) y caso descartable (neutral/otro, score bajo).

**Nota:** los textos del JSON de prueba están redactados para que el LLM los
clasifique con el sentimiento esperado (positivo o muy positivo) y así activar
las tres ramas. La clasificación real depende del modelo.

---

## 10. JSON final esperado (criterio de aceptación)

```json
{
  "status": "exito",
  "resumen_comunidad": {
    "total_interacciones_procesadas": 4,
    "sentimiento_predominante": "muy_positivo",
    "temas_principales": ["empleo", "despliegue", "hackathon"],
    "score_promedio": 5.54
  },
  "activos_distribucion_generados": {
    "post_linkedin": {},
    "destaque_newsletter_semanal": {},
    "sugerencia_contenido_faq": {}
  }
}
```

**Notas:**

- `total_interacciones_procesadas` cuenta **todas** las interacciones de entrada,
  incluidas las descartadas.
- `score_promedio` se calcula sobre `score_final` de todas las interacciones.
- `sentimiento_predominante` es el valor más frecuente del campo `sentimiento`
  (`snake_case`).
- Los objetos de activos quedan vacíos o poblados según las ramas activadas
  por el JSON de prueba.
- `almacenamiento_oci` NO aparece en la Fase 1: es parte del contrato completo
  pero se agrega en la Fase 2.

**Estimación del `score_promedio` con los 4 casos:**

| Interacción          | score_llm | señal_de_tipo | recurrencia | score_final |
| -------------------- | --------- | ------------- | ----------- | ----------- |
| Mariana (testimonio) | 9.0       | 2.0           | 0.0         | **7.4**     |
| Carlos (pregunta)    | 7.5       | 1.5           | 0.0         | **6.15**    |
| Lucía (logro)        | 8.5       | 1.5           | 0.0         | **6.95**    |
| Andrés (otro)        | 2.0       | 0.5           | 0.0         | **1.65**    |

```text
score_promedio = (7.4 + 6.15 + 6.95 + 1.65) / 4 = 22.15 / 4 = 5.54
```

Los valores de `score_llm` son estimaciones. El valor real depende del LLM.
En la práctica, el `score_promedio` puede variar.

**Sobre `sentimiento_predominante`:**

Con los 4 casos redactados (3 con sentimiento positivo o muy positivo, 1 con
neutro), el predominante esperado es `muy_positivo` por mayoría. Si el LLM
clasifica alguno de otra forma, el valor cambiará. La regla es: el valor más
frecuente del campo `sentimiento` entre las interacciones procesadas.

---

## 11. Checklist de validación de la Fase 1

- [ ] El trigger acepta el JSON de prueba sin errores de esquema.
- [ ] La Normalización emite un item por interacción con `interaction_id` único.
- [ ] Groq devuelve JSON válido con exactamente los 5 campos del contrato.
- [ ] Parse + Scoring calcula `score_final` con la fórmula documentada.
- [ ] El `score_final` de cada interacción está dentro del rango real 0.05–8.3.
- [ ] El Switch enruta cada interacción a una sola rama o a Descartar.
- [ ] Las tres ramas de generación respetan su formato de salida.
- [ ] El Merge asigna `asset_id` único por activo.
- [ ] El JSON final cumple el contrato de la sección 10.
- [ ] Una interacción negativa o neutral se descarta correctamente.

---

## 12. Estado del documento

- **Versión:** v1.0
- **Estado:** Congelado para construcción.
- **Última actualización:** 2026-09-23
- **Contrato maestro:** `docs/contracts.md` (v1.0)
```

---

Listo para copiar y subir a `docs/fase-01-nucleo.md`.
