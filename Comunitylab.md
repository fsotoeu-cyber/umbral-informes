# 🛡️ CommunityLab
### Motor Inteligente de Transformación y Distribución para Comunidades Digitales

> **Hackathon ONE G10 · Oracle Next Education & Alura · Octubre 2026**  
> **Equipo 36**  
> *"La IA propone · El humano decide · OCI guarda solo lo aprobado."*

---

## 📌 ¿Qué es CommunityLab?

**CommunityLab** transforma la actividad orgánica de una comunidad digital (testimonios, dudas, logros dispersos en canales como Discord) en activos de marketing listos para distribución — de forma automatizada, con curación humana obligatoria y persistencia en la nube de Oracle.

> El sistema **genera** contenido, el curador **aprueba o rechaza**, y **OCI Object Storage** conserva solo lo aprobado.  
> **No publica automáticamente** en redes sociales externas.

| Recurso | URL | Nota |
|--------|-----|------|
| **Panel de curaduría (operativo)** | [g10-latam-equipo-36.streamlit.app](https://g10-latam-equipo-36.streamlit.app/) | Estación real de aprobación / rechazo |
| **Landing de presentación** | [proyecto-equipo-36.netlify.app](https://proyecto-equipo-36.netlify.app/) | **Vista demostrativa** (ver nota abajo) |
| **Repositorio** | [github.com/FerIqm1993/Proyecto-Equipo-36](https://github.com/FerIqm1993/Proyecto-Equipo-36) | Código y documentación |

### Nota sobre la landing (Netlify)

La landing pública es una **vista demostrativa de presentación**: puede incluir simulador, métricas ilustrativas o botones de ejemplo. **No es la interfaz operativa de curaduría** y no debe interpretarse como escritura en tiempo real sobre Google Sheets ni como publicación en OCI.

La curaduría operativa se realiza en el **panel Streamlit**. La persistencia en OCI la ejecuta **WF2 en n8n** tras una decisión humana registrada en Sheets.

> **Pendiente en Netlify:** mostrar el mismo aviso en la página y alinear textos técnicos (PAR de solo escritura; ruta `activos/aprobados/{execution_id}/{asset_id}.json`) para que no contradigan este README.

---

## 🎯 Justificación y Propuesta de Valor

### El Problema

Las comunidades de aprendizaje tecnológico (Discord, Slack, foros, etc.) generan a diario interacciones de alto valor: testimonios de empleo, dudas técnicas recurrentes y logros de proyectos. Una parte importante de ese contenido **queda dispersa en el historial de los canales**, porque identificarlo, revisarlo y adaptarlo a formatos de comunicación requiere trabajo manual de Community Management.

### La Solución

**CommunityLab** implementa una arquitectura orientada a eventos con IA generativa y orquestación que:

1. **Ingiere y normaliza** interacciones (Webhook HTTP validado E2E + Manual Trigger; integración nativa con Discord **diseñada, pendiente**).
2. **Evalúa semánticamente** los mensajes con modelos LLM mediante la API de **Groq** y aplica scoring determinístico.
3. **Bifurca y genera** 3 formatos especializados:
   - 📢 **Post de LinkedIn** — testimonios de éxito profesional  
   - 💡 **FAQ / guía técnica** — dudas recurrentes  
   - 📰 **Destaque de newsletter** — logros y proyectos  
4. **Incorpora curaduría humana (HITL)** con dashboard **Streamlit** + Google Sheets.
5. **Persiste los activos aprobados** en **OCI Object Storage (Always Free)**.
6. **Consolida métricas diarias** (WF3) para monitoreo y BI.

---

## 🏛️ Arquitectura del Sistema

```mermaid
flowchart TD
subgraph INGESTA["1. Ingesta y Normalización"]
A[Webhook POST / Manual Trigger] --> B[Normalización de Interacciones]
B --> C[Loop Over Items]
end

subgraph PROCESAMIENTO["2. Inteligencia Semántica WF1"]
C --> D[Prompt de Análisis]
D --> E[Groq LLM]
E --> F[Parse + Scoring]
end

subgraph BIFURCACION["3. Enrutamiento"]
F --> G{Elegible y Tipo}
G -->|Testimonio + score >= 6| H[LinkedIn]
G -->|Pregunta técnica + score >= 6| I[FAQ]
G -->|Logro + score >= 6| J[Newsletter]
G -->|No elegible| K[Data Table descartes]
end

subgraph GENERACION["4. Generación"]
H --> L[Parse LinkedIn]
I --> M[Parse FAQ]
J --> N[Parse Newsletter]
end

subgraph COLA["5. Entrega a curaduría"]
L --> O[Merge Activos]
M --> O
N --> O
O --> P[(Google Sheets Curaduria)]
end

subgraph HITL["6. Decisión humana Streamlit"]
P --> Q[Panel Streamlit]
Q -->|Aprobar| R[decision = aprobado]
Q -->|Rechazar| S[decision = rechazado]
end

subgraph WF2["7. Procesamiento WF2"]
R --> T[WF2 Schedule / Manual]
S --> T
T -->|aprobado| PUT[PUT a OCI]
PUT --> X[(OCI Object Storage)]
X --> CHECK{Resultado de subida verificado}
CHECK -->|Correcto| U[Actualizar Sheet: almacenado_oci]
CHECK -->|Error o no verificado| W[Registrar error o diagnóstico]
T -->|rechazado| V[Actualizar Sheet: cerrado_rechazado]
end

subgraph METRICAS["8. Observabilidad WF3"]
Y[Schedule diario] --> Z[Metricas_diarias / Metricas_descartes_diarias]
end

style A fill:#e1f5fe,stroke:#0288d1
style E fill:#fff3e0,stroke:#f57c00
style G fill:#f3e5f5,stroke:#7b1fa2
style P fill:#e8f5e9,stroke:#388e3c
style Q fill:#e0f2f1,stroke:#00897b
style X fill:#ffebee,stroke:#c62828
```

**Contrato de decisión**

- **Streamlit** registra la decisión humana en Google Sheets (`decision` y, cuando aplica, `user`, `decision_at`, `comentario`).
- **WF2** procesa esa decisión:
  - aprobado → PUT a OCI → si la subida se verifica → `estado_proceso = almacenado_oci`
  - rechazado → `estado_proceso = cerrado_rechazado`
  - fallo o respuesta no verificada → registro de error / diagnóstico (`error_oci` u otros estados de diagnóstico definidos en el flujo)

**Ingesta externa (solo WF1):** validada con cliente HTTP (ReqBin): `POST` → respuesta HTTP 200 con `{"message":"Workflow was started"}`.

Eso acredita que el webhook **aceptó iniciar** el workflow. El procesamiento posterior se acredita con la ejecución exitosa de WF1 y las filas en Curaduria. La persistencia se verifica tras la aprobación humana, mediante WF2 y el objeto correspondiente en OCI. Para una cadena E2E, relacionar el mismo `execution_id` y `asset_id` en esas evidencias.

Endpoint de referencia:

```text
POST https://equipo36.app.n8n.cloud/webhook/communitylab-ingesta
Content-Type: application/json
```

WF2 y WF3 **no** usan este webhook. Discord/Slack como fuente nativa: **diseñado, pendiente**.

---

## 📂 Estructura del Proyecto

```text
communitylab/
├── README.md
├── .gitignore
├── .env.example
├── workflows/
│   ├── WF1-ingesta-analisis-generacion.json
│   ├── WF2-curaduria-oci.json
│   ├── WF3-metricas-diarias.json
│   └── sample-payload-ingesta.json
├── dashboard/
│   ├── dashboard_curaduria.py
│   ├── requirements.txt
│   ├── README-dashboard.md
│   ├── plantilla_curaduria.csv
│   └── .streamlit/
│       └── secrets.toml.example
└── docs/
    ├── README.md
    └── capturas/
```

---

## ⚙️ Workflows de Automatización (n8n)

### 1. `WF1-ingesta-analisis-generacion.json`

- **Disparador:** Webhook HTTP (`POST /webhook/communitylab-ingesta`) o ejecución manual con dataset de prueba.
- **Normalización:** Homogeneiza autores, canales, tipos y timestamps; asigna `execution_id` con formato `CL-YYYYMMDD-XXX`.
- **Análisis semántico (Groq):** sentimiento, temas, `tipo_detectado`, `score_llm`, insight.
- **Scoring determinístico:**
  ```text
  score_final = 0.75 × score_llm + 0.25 × señal_tipo
  ```
  **Triple barrera de elegibilidad:** sentimiento `muy_positivo` o `positivo` · `score_final >= 6` · tipo en `testimonio` | `pregunta_tecnica` | `logro`.
- **Bifurcación:** tres generadores LLM (LinkedIn, FAQ, Newsletter).
- **Entrega a curaduría:** registra activos en Google Sheets (`pendiente_curaduria` / `pendiente_decision`).
- **Descartes:** Data Table con `motivo_descarte` clasificado.
- **Nota:** WF1 **no** escribe en OCI.

### 2. `WF2-curaduria-oci.json`

- **Disparador:** Schedule cada **15 minutos** + Manual Trigger.
- **Filtro:** filas con decisión humana explícita (p. ej. `decision = aprobado` o `rechazado`), excluyendo pendientes y estados ya cerrados (`almacenado_oci`, `cerrado_rechazado`); permite reintento controlado en estados de error/diagnóstico.
- **OCI:** **PUT** a Object Storage mediante **Pre-Authenticated Request (PAR)** de **solo escritura**.
- **Ruta de objetos:**
  ```text
  activos/aprobados/{execution_id}/{asset_id}.json
  ```
- **Cierre de ciclo (resultado de WF2):**
  1. Si `decision = aprobado` → PUT a OCI  
  2. Si la subida se verifica (HTTP 200/201) → actualizar Sheet: `estado_proceso = almacenado_oci`  
  3. Si la subida falla o no se verifica → registrar error / diagnóstico (`error_oci`, `respuesta_oci_no_verificada`, etc.)  
  4. Si `decision = rechazado` → actualizar Sheet: `cerrado_rechazado` (sin persistir en OCI)

**Seguridad OCI:** el PAR usado en desarrollo fue **rotado** tras exposición accidental en exports de workflow. En producción el valor del PAR **no** se versiona en el repositorio; se configura solo en n8n (credencial / variable de entorno / campo del nodo). Los JSON exportados en el repo deben ir con `PAR_BASE` vacío o placeholder.

### 3. `WF3-metricas-diarias.json`

- **Disparador:** Schedule **diario** (intervalo en días; p. ej. cada 1 día, a la hora/minuto configurados) + Manual Trigger para pruebas.
- **Salida:** pestañas `Metricas_diarias` y `Metricas_descartes_diarias`.
- **Indicadores:** totales, pendientes, aprobados, rechazados, `almacenado_oci`, distribución por tipo, score promedio, embudo de motivos de descarte.
- Listo para conectar **Power BI / Looker Studio**.

---

## 🖥️ Módulo de Curaduría (Dashboard Streamlit)

El archivo [`dashboard/dashboard_curaduria.py`](dashboard/dashboard_curaduria.py) es la **estación operativa** de curaduría:

- Visualización de activos pendientes (LinkedIn / FAQ / Newsletter).
- Acciones **Aprobar** / **Rechazar** que escriben en Google Sheets.
- El panel permite revisar y editar el texto antes de registrar la decisión humana.
- **WF2** empaqueta los campos de contenido presentes en la fila al momento del procesamiento, junto con los metadatos de auditoría disponibles.
- Como mejora de integridad, se recomienda comprobar la correspondencia de `execution_id` y `asset_id` antes de actualizar una fila.

Guía: [`dashboard/README-dashboard.md`](dashboard/README-dashboard.md)  
**Panel en vivo:** [https://g10-latam-equipo-36.streamlit.app/](https://g10-latam-equipo-36.streamlit.app/)

---

## 📑 Acuerdos y Contratos de Datos (Data Contracts)

### Entrada al pipeline (Trigger)

```json
{
  "origen_comunidad": "Discord_Grupo_ONE_G10",
  "periodo_referencia": "Semana_04",
  "interacciones": [
    {
      "autor": "Mariana Souza",
      "canal": "#logros-y-empleos",
      "tipo": "testimonio",
      "texto": "Después de 8 meses aplicando lo que aprendí en el programa, firmé mi primera oferta como desarrolladora junior."
    }
  ]
}
```

### Salida del análisis semántico (LLM)

```json
{
  "sentimiento": "muy_positivo",
  "temas": ["empleo", "desarrollo", "carrera"],
  "tipo_detectado": "testimonio",
  "score_llm": 9,
  "insight": "Historia de inserción laboral con alta resonancia comunitaria"
}
```

### Ejemplo de metadatos de persistencia en OCI

```json
{
  "bucket": "communitylab-activos-marketing",
  "ruta_objeto": "activos/aprobados/CL-20261014-806/CL-20261014-806-int-001-li.json",
  "status": "almacenado_oci",
  "timestamp": "2026-10-14T18:00:00Z"
}
```

---

## 🚀 Guía de Despliegue Rápido

### 1. Workflows en n8n

1. **Workflows** → **Import from File**.
2. Importar:
   - `workflows/WF1-ingesta-analisis-generacion.json`
   - `workflows/WF2-curaduria-oci.json`
   - `workflows/WF3-metricas-diarias.json`
3. Credenciales:
   - **Groq API:** Header Auth (`Authorization: Bearer <TU_GROQ_API_KEY>`).
   - **Google Sheets:** OAuth2 o Service Account.
   - **OCI:** URL del **PAR** (solo escritura) configurada **únicamente en n8n**, nunca en el repositorio. Si un PAR se expuso, **rotarlo** en la consola OCI y actualizar solo la instancia de n8n.

### 2. Dashboard Streamlit

```bash
cd dashboard
python -m venv venv
# Activar el entorno virtual según el SO
pip install -r requirements.txt
# Configurar .streamlit/secrets.toml (GCP / Sheets)
streamlit run dashboard_curaduria.py
```

### 3. Prueba de ingesta externa

```text
POST https://equipo36.app.n8n.cloud/webhook/communitylab-ingesta
Content-Type: application/json
```

Body de ejemplo: `workflows/sample-payload-ingesta.json`.  
Respuesta esperada del webhook: HTTP **200** con `{"message":"Workflow was started"}`.

Para una cadena E2E completa, relacionar el mismo `execution_id` (y `asset_id`) en: ejecución Success de WF1 → fila en Curaduria → decisión humana → ejecución de WF2 → objeto en OCI.

---

## 🔒 Seguridad

- Secretos (Groq, Google, PAR de OCI) solo en n8n / Streamlit Cloud; **no** versionados en el repo.
- PAR de OCI con permiso mínimo (escritura de objetos). **El PAR utilizado en desarrollo fue rotado** tras una exposición en un export de workflow; el valor activo no se publica en GitHub.
- Texto de usuario tratado como dato; validación de salidas JSON; **HITL antes de persistir**.
- Datos de prueba / mock en validaciones del hackathon.

---

## 📸 Evidencias del sistema (`docs/capturas/`)

| # | Archivo | Contenido validado |
|---|---------|-------------------|
| 00 | `00-n8n-overview-workflows.png` | Panel n8n: **487+ ejecuciones**, ~**0.4%** tasa de fallo (valores visibles en la captura) |
| 01 | `01-wf1-arquitectura.png` | Canvas WF1 |
| 02 | `02-wf2-arquitectura.png` | Canvas WF2 |
| 03 | `03-wf3-arquitectura.png` | Canvas WF3 |
| 04 | `04-oci-bucket-prefijos.png` | Bucket OCI y prefijos |
| 05 | `05-oci-json-trazabilidad.png` | Objeto JSON en OCI |
| 06a | `06-sheet-curaduria-datos.png` | Sheet: contenidos y scores |
| 06b | `06-sheet-curaduria-estados.png` | Sheet: decisiones y `almacenado_oci` |
| 07a | `07-sheet-metricas-diarias.png` | Métricas diarias |
| 07b | `07-sheet-metricas-descartes.png` | Embudo de descartes |
| 08 | `08-datatable-descartes.png` | Data Table de descartes |
| 09 | `09-execuciones-automaticas-wf2.png` | Schedule WF2 en Success |
| 10 | `10-panel-curaduria.png` | Streamlit operativo |
| 11 | `11-landing-presentacion.png` | Landing **demostrativa** (Netlify) |
| 12 | `12-reqbin-post-ingesta.png` | POST → 200 + `Workflow was started` |
| 13 | `13-executions-wf1-success.png` | Historial Success de WF1 |

**Lectura de la evidencia E2E:**  
Ingesta externa validada con POST y HTTP 200 (captura 12). El procesamiento posterior se acredita con la ejecución exitosa de WF1 (13) y las filas en Curaduria (06). La persistencia se verifica tras la aprobación humana, mediante WF2 (02, 09) y el objeto en OCI (04, 05).

---

## 🏆 Matriz de Cumplimiento de Criterios de Evaluación

| Criterio del Hackathon ONE | Estado | Evidencia en el Repositorio |
| :---------------------------------------------------- | :--------: | :----------------------------------------------------------------------- |
| **1. Ingestión funcional de interacciones** | ✅ Cumplido | Webhook + Normalización en WF1; prueba externa ReqBin (`docs/capturas/12-...`). |
| **2. Análisis de sentimiento y extracción con LLM** | ✅ Cumplido | Prompt estructurado y parsing JSON con Groq en WF1. |
| **3. Generación de ≥ 2 formatos de marketing** | ✅ Cumplido | **3 formatos:** LinkedIn, FAQ y Newsletter. |
| **4. Orquestación del flujo en n8n** | ✅ Cumplido | WF1 + WF2 + WF3 en `workflows/`. |
| **5. Persistencia en OCI Object Storage Always Free** | ✅ Cumplido | WF2 con PUT vía PAR; bucket y objetos verificados. |
| **6. Demostración de ≥ 3 ejemplos de transformación** | ✅ Cumplido | Testimonio→LinkedIn · duda→FAQ · logro→Newsletter (+ descartes auditados). |
| **7. Documentación y diagrama del pipeline** | ✅ Cumplido | Este README, contratos de datos y `docs/capturas/`. |
| **Diferencial: Curaduría Humana (HITL)** | ✅ Cumplido | Streamlit operativo + Google Sheets + WF2. |
| **Diferencial: Observabilidad y métricas** | ✅ Cumplido | WF3 + pestañas `Metricas_diarias` / `Metricas_descartes_diarias`. |

---

## 🚀 Roadmap y Evolución del Producto

### 1. Notificaciones por correo electrónico
Integrar un **nodo de correo en n8n** para notificar al equipo de marketing sobre eventos relevantes del proceso, por ejemplo nuevos activos pendientes de curaduría y activos aprobados cuya persistencia en OCI haya sido confirmada.

**Beneficio esperado:** mejorar la coordinación del equipo y reducir la necesidad de revisar manualmente el panel para detectar novedades.

### 2. Integración con Discord
Desarrollar un bot o una integración que capture mensajes de canales autorizados de Discord y los adapte al **contrato de entrada existente** (`origen_comunidad`, `periodo_referencia`, `interacciones`). Los mensajes se enviarán al **webhook actual de WF1**, reutilizando la normalización, el análisis con LLM, el scoring y la generación de activos.

**Beneficio esperado:** convertir CommunityLab en un sistema capaz de recibir interacciones directamente desde la comunidad, reduciendo la ingesta manual.

### 3. Gobernanza y trazabilidad
**Ampliar** la gobernanza ya operativa (curaduría humana obligatoria, registro de decisiones en Google Sheets, seguimiento de `estado_proceso` y auditoría de descartes en Data Table) con controles de acceso más finos y mayor trazabilidad de extremo a extremo, sin eliminar el requisito de aprobación humana antes de persistir en OCI Object Storage.

**Nota:** las funcionalidades de este roadmap se consideran **mejoras futuras** hasta que su implementación y funcionamiento hayan sido probados.

---

## 👥 Equipo 36

- **Proyecto:** CommunityLab  
- **Landing (demo de presentación):** [https://proyecto-equipo-36.netlify.app/](https://proyecto-equipo-36.netlify.app/)  
- **Panel de curaduría (operativo):** [https://g10-latam-equipo-36.streamlit.app/](https://g10-latam-equipo-36.streamlit.app/)  
- **Repositorio:** [https://github.com/FerIqm1993/Proyecto-Equipo-36](https://github.com/FerIqm1993/Proyecto-Equipo-36)  
- **Hackathon:** Oracle Next Education & Alura — Edición G10  

| Integrante | Contribución principal |
|------------|------------------------|
| Fausto Soto | Arquitectura, n8n (WF1/WF2/WF3), OCI, pruebas, documentación |
| Óscar Velaztiqui | Curación, panel Streamlit (deploy y E2E) |
| Luis Fernando Martínez | Repositorio GitHub, landing de presentación |
| Lisa Gonzalez | Business Intelligence (Power BI / Looker Studio) |
| Daryin Carreon Gonzalez | Video de demostración |

---

**CommunityLab — La IA propone · El humano decide · OCI guarda solo lo aprobado.**
```
