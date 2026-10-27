---
name: psa-solution-architect
description: >-
  Construye desde cero las herramientas deterministas y la arquitectura de 2 niveles de agentes
  (Orquestador en gemini-3.8-flash + Auditor ARB/Well-Architected en gemini-4-argon) para resolver
  el Agente PSA (/psa), Cotizador (/quote) y Apertura de Tiendas (PFERNANDEZ) alineados al Blueprint
  de Interconexión IA y al Google Cloud Well-Architected Framework.
---

# Skill: Agente PSA, Cotizador (`/quote`) y Apertura de Tiendas con BigQuery (`us-west1`) bajo el Blueprint de Interconexión IA

Este skill implementa las reglas de negocio de **Arquitectura Empresarial**, las **7 Reglas del Blueprint de Interconexión IA** y los **5 Pilares del Google Cloud Well-Architected Framework** en `psa_agent/agent.py`, conectándose en tiempo real a **BigQuery (`us-west1` — Oregon)** mediante `google-cloud-bigquery` con **ADC / Service Account (Cero API Keys)**.

## 0. Esquema del Dataset Aislado en BigQuery (`psa_governance_<participant_id>` en `us-west1`)

Cada participante crea y puebla su propio dataset aislado `BQ_DATASET = f"psa_governance_{PARTICIPANT_ID.replace('-', '_')}"` en la región **`us-west1` (Oregon)** a partir de las semillas de `data/`:
1. **Tabla `catalogo_tarifas`**:
   - Columnas: `role_key STRING`, `nombre STRING`, `area STRING`, `negocio STRING`, `tarifa_base_mensual INT64`, `moneda STRING`, `version STRING` (poblada desde `data/catalogo_tarifas.json`).
2. **Tabla `mapa_capacidades_retail`**:
   - Columnas: `domain STRING`, `exposicion_inicial STRING`, `capacidad_n2 STRING`, `capacidad_n3 STRING` (poblada desde `data/mapa_capacidades_retail.json`).
3. **Tabla Transaccional y de Auditoría `registro_dictamenes_psa` (Idempotencia Sección 8.2 + Cola `Propose & Approve` Regla 4)**:
   - Columnas: `idempotency_key STRING`, `correlation_id STRING`, `participant_id STRING`, `project_name STRING`, `duration_months INT64`, `risk_tier STRING`, `orchestration_pattern STRING`, `approval_status STRING`, `monthly_recurrent_mxn FLOAT64`, `annual_run_rate_mxn FLOAT64`, `project_total_mxn FLOAT64`, `tbd_roles_count INT64`, `created_at TIMESTAMP`.

## 1. Las 4 Herramientas Deterministas con Contrato MCP (Sección 7.1 y 8.2) Conectadas a BigQuery

1. **`clean_and_anonymize_transcript(raw_transcript: str) -> dict`**:
   - **Contrato MCP**: `risk_level="L0"`, `execution_pattern="Consulta gobernada"`, `owner_domain="Seguridad & Privacidad IA"`.
   - **Pilar Well-Architected (Seguridad)**: Limpia muletillas y redacta con regex correos (`[EMAIL_REDACTED]`), tarjetas de 16 dígitos (`[PAN_REDACTED]`) y teléfonos (`[PHONE_REDACTED]`) antes de que el insumo toque el LLM. Devuelve `correlation_id` y conteo de redacciones.

2. **`get_retail_capability_map(domain_filter: str = "") -> dict`**:
   - **Contrato MCP**: `risk_level="L0"`, `execution_pattern="Consulta gobernada"`, `owner_domain="Arquitectura Empresarial (BigQuery Data Product)"`.
   - Consulta con `bigquery.Client(project=PROJECT_ID, location="us-west1")` la tabla real **`{BQ_DATASET}.mapa_capacidades_retail`** en BigQuery (filtrando opcionalmente por `domain_filter`) y devuelve los 9 dominios de negocio retail más las reglas de separación entre **Sección 5.2 (Infraestructura)** y **Sección 5.4 (Redes)**.

3. **`calculate_deterministic_quote(project_name: str, duration_months: int, line_items: list[dict], idempotency_key: str, risk_tier: str = "L2") -> dict`**:
   - **Contrato MCP**: `risk_level="L2"` (o `"L4"` si toca sistemas críticos/financieros), `execution_pattern="Ejecución acotada / Propose & Approve"`, `owner_domain="Finanzas & Estimación TI (BigQuery)"`. Exige `idempotency_key` (Sección 8.2) y devuelve `correlation_id`.
   - **Paso A — Control de Idempotencia Real en BigQuery (Sección 8.2 del Blueprint)**:
     - Ejecuta `SELECT * FROM {BQ_DATASET}.registro_dictamenes_psa WHERE idempotency_key = @idempotency_key LIMIT 1`. Si la llave ya existe en BigQuery, devuelve el dictamen previamente registrado con `"idempotent_replay": True` sin duplicar el registro.
   - **Paso B — Consulta del Tabulador Oficial en BigQuery (`{BQ_DATASET}.catalogo_tarifas`)**:
     - Consulta `SELECT role_key, nombre, area, negocio, tarifa_base_mensual, moneda FROM {BQ_DATASET}.catalogo_tarifas` en `us-west1`.
     - Ejecuta la aritmética en Python determinista (`costo_mensual = tarifa_base_mensual * allocation * units`). Cualquier rol que no exista en la tabla `catalogo_tarifas` de BigQuery queda como `"TBD"` y `⟨propuesto: Rol fuera de catálogo en BigQuery⟩`.
     - Calcula las **3 cifras obligatorias**: `monthly_recurrent_total_mxn`, `annual_run_rate_mxn` (`mensual * 12`) y `project_total_mxn`.
   - **Paso C — Registro de Auditoría y Cola `Propose & Approve` en BigQuery (`{BQ_DATASET}.registro_dictamenes_psa`)**:
     - Inserta la fila en `{BQ_DATASET}.registro_dictamenes_psa` con `approval_status = "PENDING_HUMAN_APPROVAL_L4"` cuando `risk_tier == "L4"` (cumpliendo la **Regla 4 del Blueprint**) o `"REGISTERED_DRAFT_L2"` en caso estándar.

4. **`load_sample_case(case_id: str) -> dict`**:
   - **Contrato MCP**: `risk_level="L0"`, `execution_pattern="Consulta gobernada"`, `owner_domain="Portal IA"`.
   - Carga los casos reales desde `data/ejemplos_minutas.json`.

## 2. Arquitectura de Orquestación en 2 Niveles (Sección 8.1 — Regla 5 y Regla 7)

1. **Sub-Agente Especialista Auditor (`psa_architecture_validator_agent` con `MODEL_REASONING = "gemini-4-argon"`)**:
   - Actúa como el revisor automático del **AI Architecture Board (ARB)** y del **Google Cloud Well-Architected Framework**.
   - Verifica:
     - **Reglas de Extracción `/psa`**: Nada inventado en silencio (`⟨propuesto: …⟩`); **Folio** y los **4 roles de la Sección 1** (`Arquitecto`, `TPO`, `Coordinador`, `PO`) permanecen como `"[VACÍO - Pendiente de declarar]"` si el insumo no los declara.
     - **Redes (5.4) vs. Infraestructura (5.2)**: En **Sección 5.2 (Infraestructura)** van el cómputo y estado (Cloud Run, Vertex AI Agent Engine, Artifact Registry). En **Sección 5.4 (Redes)** van Balanceadores F5, Cloud Load Balancing, Cloud Armor, Cloud API Gateway (explicando que en fase inicial se usa Cloud Run IAM OIDC + Cloud API Gateway sin el costo/complejidad de Apigee, dejando el Enterprise API/MCP Gateway como evolución a escala corporativa), VPN y Cloud Interconnect.
     - **Gobernanza L0–L4 y Patrón `Propose & Approve` (Regla 4)**: Si la iniciativa involucra sistemas críticos o financieros (`MPE`, `ERP`, conciliación contable en `Servicios` o `Administración y Finanzas`), dictamina obligatoriamente **Risk Tier `L4`** y patrón **`Propose & Approve`** (prohibida la ejecución autónoma; requiere doble aprobación humana).
     - **5 Pilares de Google Cloud Well-Architected**: Evalúa cómo la solución cumple *Seguridad (IAM 3 niveles / ADC)*, *Confiabilidad (Sesiones/Memoria en Agent Engine + Idempotencia)*, *Optimización de Costos (Escala a cero + Ruteo Flash/Argon + Gateway pragmático)*, *Excelencia Operativa (Contrato MCP 7.1 + Trazabilidad)* y *Rendimiento*.

2. **Agente Coordinador Raíz (`retail_psa_architect_agent` / `root_agent` con `MODEL_FAST = "gemini-3.8-flash"`)**:
   - Recibe la solicitud del usuario, precarga memoria de largo plazo con `PreloadMemoryTool()`, invoca las herramientas deterministas y se apoya en `psa_architecture_validator_agent` para entregar el dictamen final.

## 3. Formato de Salida Dual (Narrativa Markdown + Bloque JSON de Artefacto para la UI)

Instruye al agente a responder siempre con su análisis ejecutivo en Markdown seguido de un bloque ` ```json ... ``` ` con esta estructura enriquecida con el Blueprint y el Well-Architected Framework:

```json
{
  "artifact_type": "psa | quote | expansion",
  "title": "Título de la Iniciativa",
  "folio": "[VACÍO - Pendiente de declarar]",
  "blueprint_governance": {
    "risk_tier": "L0 | L1 | L2 | L3 | L4",
    "orchestration_pattern": "Consulta gobernada | Ejecución acotada | Propose & approve | Orquestación multi-sistema",
    "human_approval_required": true,
    "gateway_strategy": "Fase 1: Cloud Run IAM OIDC + Cloud API Gateway / Cloud Armor (Sin costo base de Apigee) -> Evolución Escala: Enterprise API/MCP Gateway"
  },
  "section_1_roles": {
    "arquitecto": "[VACÍO - Pendiente de declarar]",
    "tpo": "[VACÍO - Pendiente de declarar]",
    "coordinador": "[VACÍO - Pendiente de declarar]",
    "po": "[VACÍO - Pendiente de declarar]"
  },
  "executive_summary": "Resumen ejecutivo...",
  "business_capabilities": [
    {"domain": "Operaciones", "capability": "2. Gestión de pagos", "subcapability": "2.2 Procesamiento de Transacciones"}
  ],
  "architecture_sections": {
    "infraestructura_5_2": ["Cloud Run (Serverless min-instances=0)", "Vertex AI Agent Engine (Sessions & Memory Bank)", "BigQuery"],
    "integraciones_5_3": ["Servidor MCP de Dominio en Cloud Run (Regla 3)", "Contrato Mínimo MCP 7.1 (Idempotencia + X-Correlation-ID)"],
    "redes_5_4": ["Cloud Load Balancing + Cloud Armor", "Cloud API Gateway + Cloud Run IAM OIDC (Evolución futura a Enterprise Gateway / Apigee a escala)", "Balanceador F5 / Cloud Interconnect"],
    "seguridad_iam": ["Service Account dedicada por agente sa-psa-<id> (Regla 2)", "Clasificación de riesgo L0-L4 y Propose & Approve en L4 (Regla 4)"],
    "monitoreo": "Pendiente de complementar por el dueño del PSA"
  },
  "well_architected_alignment": {
    "security": "ADC local + Service Account dedicada por agente (Cero API Keys) + Redacción DLP previa al LLM",
    "reliability": "Estado externalizado en Vertex AI Agent Engine + Clave de idempotencia obligatoria en L2-L4",
    "cost_optimization": "Cloud Run escala a cero + Ruteo gemini-3.8-flash / gemini-4-argon + Gateway serverless sin licencia base Apigee",
    "operational_excellence": "Ficha en Catálogo de Agentes + Contrato MCP 7.1 + Correlación de trazas",
    "performance": "PreloadMemoryTool selectivo + Cálculos financieros deterministas en Python"
  },
  "mermaid_diagrams": {
    "c4_context": "flowchart TD\n  ...",
    "c4_containers": "flowchart TD\n  ...",
    "infra_network": "flowchart TD\n  ..."
  },
  "quote_summary": {
    "currency": "MXN",
    "duration_months": 6,
    "monthly_recurrent_total_mxn": 0,
    "annual_run_rate_mxn": 0,
    "one_time_bags_total_mxn": 0,
    "project_total_mxn": 0,
    "line_items": []
  }
}
```
