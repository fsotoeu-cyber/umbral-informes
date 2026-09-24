
---

```markdown
# Fase 2 — Persistencia (OCI Object Storage)

Documento de trabajo para el equipo. Define el contrato de persistencia y el
criterio de aceptación de la Fase 2.

**Prerrequisito:** no construir esta fase hasta que la Fase 1 pase su checklist
(ver `docs/fase-01-nucleo.md`, sección 11).

---

## 1. Objetivo

Guardar cada activo generado en **OCI Object Storage** (capa Always Free)
y exponer la ruta en el JSON final.

---

## 2. Flujo

```text
... (núcleo Fase 1)
↓
Merge
↓
OCI Object Storage
↓
JSON final (con almacenamiento_oci)
```

---

## 3. Contrato — Resultado de escritura en OCI

```json
{
  "bucket": "communitylab-activos-marketing",
  "ruta_objeto": "activos/2026-semana-04/CL-2026-001/asset-001.json",
  "status": "guardado_con_exito"
}
```

---

## 4. Convención de rutas

```text
activos/{periodo_referencia}/{execution_id}/{asset_id}.json
```

**Ejemplo:**

```text
activos/Semana_04/CL-20260923-142/asset-001-li.json
```

---

## 5. JSON final actualizado (Fase 2)

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
  },
  "almacenamiento_oci": {
    "bucket": "communitylab-activos-marketing",
    "ruta_objeto": "activos/Semana_04/CL-20260923-142/asset-001-li.json",
    "status": "guardado_con_exito"
  }
}
```

**Nota:** en Fase 2, `almacenamiento_oci` refleja el guardado del pipeline
(cada activo generado se persiste). En Fase 3, la escritura definitiva ocurre
tras la aprobación humana; el pipeline solo persistirá activos aprobados.

---

## 6. Requisitos técnicos OCI

| Elemento              | Valor                                      |
| --------------------- | ------------------------------------------ |
| Bucket                | `communitylab-activos-marketing`           |
| Capa                  | Always Free                                |
| Content-Type          | `application/json`                         |
| Región                | La configurada en la cuenta Always Free    |
| Autenticación         | API Key / Config file de OCI CLI           |

---

## 7. Comportamiento esperado

1. Cada activo que sale del **Merge** se serializa como JSON.
2. Se sube a OCI con la convención de rutas definida.
3. Se recibe confirmación de escritura.
4. Se agrega el bloque `almacenamiento_oci` al JSON final.
5. Si la subida falla, el flujo no debe romperse por completo:
   - Registrar el error en el status.
   - Continuar con el resto de activos si es posible.

---

## 8. Criterio de aceptación Fase 2

- [ ] Cada activo del Merge se guarda en OCI con la convención de rutas.
- [ ] El JSON final incluye `almacenamiento_oci` con valores reales.
- [ ] Un reintento de ejecución con el mismo `execution_id` no sobrescribe
      sin control (usar `asset_id` único o sufijo de versión).
- [ ] El bucket existe y está en la capa Always Free.
- [ ] El Content-Type del objeto es `application/json`.

---

## 9. Notas de implementación en n8n

- Usar un nodo **HTTP Request** (o el nodo oficial de OCI si está disponible).
- Method: `PUT` (o el que indique la API de Object Storage).
- Body: el JSON del activo serializado.
- Headers: autenticación OCI + `Content-Type: application/json`.
- Después de la subida, construir el objeto `almacenamiento_oci` y
  unirlo al JSON final.

---

## 10. Relación con Fase 3

| Fase | Qué se guarda en OCI                          |
| ---- | --------------------------------------------- |
| 2    | Todos los activos generados por el pipeline   |
| 3    | Solo los activos **aprobados** por un humano  |

En Fase 3 el guardado se mueve **después** de la decisión humana
(`approve` → OCI).

---

## 11. Estado del documento

- **Versión:** v1.0
- **Estado:** Congelado para construcción (después de Fase 1).
- **Última actualización:** 2026-09-23
- **Documento relacionado:** `docs/fase-01-nucleo.md` (v1.0)
- **Documento completo de fases posteriores:** `docs/fases-02-04.md` (v1.0)
```

---

Listo para copiar y guardar como `docs/fase-02-persistencia.md` (o integrarlo dentro de `fases-02-04.md`).
