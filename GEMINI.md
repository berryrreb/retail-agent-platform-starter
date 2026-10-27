# Reglas de Gobernanza de IA — Plataforma de Interconexión IA & Google Cloud Well-Architected (`GEMINI.md`)

Este archivo establece las reglas obligatorias para **Google Antigravity** (tanto en el IDE como en `agy` CLI) dentro de este repositorio. Todas las generaciones de código, arquitecturas, agentes y componentes visuales deben materializar el **Framework de la Plataforma de Interconexión IA (9 Capas, 7 Reglas, Escala de Riesgo L0–L4 y 5 Patrones de Orquestación)** alineado a los **5 Pilares del Google Cloud Well-Architected Framework**.

---

## 1. Portafolio Gobernado de Modelos y Ruteo por Tarea (Capa 6 y 7 — Pilar FinOps & Performance)

- **ÚNICOS MODELOS PERMITIDOS**:
  - `gemini-3.8-flash` — **Modelo rápido y especializado**: Para extracción estructurada, clasificación de capacidades, triage, invocación de herramientas deterministas y coordinación de baja latencia (`MODEL_FAST`).
  - `gemini-4-argon` — **Modelo de frontera para razonamiento profundo**: Reservado para validación arquitectónica del ARB, auditoría contra las 7 Reglas del Blueprint y el Well-Architected Framework, y diseño de diagramas C4 (`MODEL_REASONING`).
- **PROHIBICIÓN ABSOLUTA**: Nunca utilices ni referencies ningún otro modelo en archivos `.py`, `.env`, o configuraciones. Los identificadores de modelo deben leerse siempre desde variables de entorno (`MODEL_FAST` y `MODEL_REASONING`) con fallback exclusivo a `gemini-3.8-flash` y `gemini-4-argon`.

---

## 2. Identidad, Autenticación y Seguridad (Regla 2 del Blueprint — Pilar de Seguridad Well-Architected)

- **Cero API Keys**: Está estrictamente prohibido usar `GOOGLE_API_KEY`, `GEMINI_API_KEY` o llaves estáticas en código o archivos de configuración.
- **Cero Llaves JSON Descargadas**: Nunca generes instrucciones ni código que utilice `gcloud iam service-accounts keys create` o `GOOGLE_APPLICATION_CREDENTIALS` apuntando a un archivo JSON descargado.
- **Modelo IAM de 3 Niveles (Mínimo Privilegio)**:
  1. **Siempre usar Vertex AI en `us-west1` (Oregon)**: `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, `GOOGLE_CLOUD_PROJECT` y `GOOGLE_CLOUD_LOCATION=us-west1`.
  2. **Nivel 2 — En Desarrollo Local (Participante / Desarrollador)**: Autenticación mediante **Application Default Credentials (ADC)** (`gcloud auth application-default login`) con roles acotados sin privilegios administrativos de IAM.
  3. **Nivel 3 — En Cloud Run (Identidad Propia del Agente — Regla 2)**: El servicio se despliega adjuntando una **Service Account dedicada por agente y participante** (`--service-account=sa-psa-<PARTICIPANT_ID>@...`) con permisos mínimos (`roles/aiplatform.user`, `roles/bigquery.dataEditor`, `roles/bigquery.jobUser`, `roles/logging.logWriter`, `roles/cloudtrace.agent`).

---

## 3. Estrategia Pragmática de Gateway y Los 4 Tramos de Integración (Regla 1 y 3 — Pilar de Costos)

El camino desde un sistema hasta el agente respeta siempre los **4 tramos del Blueprint**:
1. **API REST / Catálogo Existente** -> 2. **Servidor MCP de Dominio (mantenido por el equipo dueño)** -> 3. **Capa de Gateway** -> 4. **Agente**.
- **Directriz de Costo y Complejidad sobre Apigee**:
  - **En este ejercicio y en la Fase Inicial de adopción en GCP, NO se despliega ni se exige Apigee** por razones de costo base y complejidad operativa. En su lugar, la capa de Gateway y seguridad entre servicios se resuelve de forma nativa, serverless y costo-eficiente con **Cloud Run IAM (autenticación OIDC máquina-a-máquina) + Cloud API Gateway / Cloud Load Balancing + Cloud Armor**.
  - **En la visión de Arquitectura Empresarial Objetivo**, el componente de **Enterprise API / MCP Gateway (como Apigee o el API Gateway corporativo existente)** se reconoce como pieza estratégica para etapas de madurez y escala multi-dominio, y debe mencionarse en los diagramas de arquitectura empresarial explicando esta evolución gradual (Fase 1 Serverless Ligera -> Fase 2 Gateway Empresarial Consolidado).

---

## 4. Sesiones, Memoria y Resiliencia en Google ADK (Capa 3 — Pilar de Confiabilidad)

- Todo agente construido con **Google ADK (`google-adk`)** externaliza su estado directamente en **Vertex AI Agent Engine (`us-west1` — Oregon)** sin usar mocks en memoria:
  - **Sesiones**: `VertexAiSessionService(project=..., location="us-west1", agent_engine_id=...)`
  - **Memoria de Largo Plazo**: `VertexAiMemoryBankService(project=..., location="us-west1", agent_engine_id=...)` junto con `PreloadMemoryTool()`.

---

## 5. Las 7 Reglas del Blueprint, Escala de Riesgo `L0–L4` y Reglas de `/psa` y `/quote`

1. **Privacidad e Inyección de Datos (Riesgo `L0`/`L1`)**: Ninguna transcripción cruda se procesa sin pasar primero por la herramienta determinista `clean_and_anonymize_transcript` que redacta `PAN`, correos y teléfonos.
2. **Contrato Mínimo de Herramienta MCP (Sección 7.1) y Control Transaccional (Sección 8.2)**: Toda herramienta devuelve `risk_level` (`L0`–`L4`), `execution_pattern` (*Consulta gobernada*, *Ejecución acotada*, *Propose & approve*), `owner_domain` y un `correlation_id` de trazabilidad; las herramientas `L2`–`L4` exigen `idempotency_key`.
3. **Frontera `L4` — `Propose & Approve` (Regla 4)**: Ninguna acción financiera o sobre sistemas críticos (ERP, MPE, pagos) se ejecuta de forma autónoma; el agente solo genera una propuesta con evidencia sujeta a aprobación humana.
4. **Nada Inventado en Silencio**: Todo dato inferido o sugerido por el agente debe marcarse explícitamente con `⟨propuesto: …⟩`.
5. **Campos que Nunca se Infieren**: El **Folio de la iniciativa** y los **4 roles de la Sección 1** (*Arquitecto*, *TPO*, *Coordinador*, *PO*) solo se incluyen si el insumo los declara explícitamente; de lo contrario, permanecen como `[VACÍO - Pendiente de declarar]`.
6. **Separación Redes vs. Infraestructura**: Balanceadores F5, Cloud Load Balancing, Cloud Armor, Cloud API Gateway (o Gateway corporativo), proxies, VPN e Interconnect pertenecen a **Redes (Sección 5.4)**; cómputo serverless (Cloud Run, Vertex AI Agent Engine) y almacenamiento pertenecen a **Infraestructura (Sección 5.2)**.
7. **Catálogos y Registro Transaccional en BigQuery (`us-west1`)**: Los cálculos financieros jamás los suma el LLM mentalmente; siempre se ejecutan mediante `calculate_deterministic_quote` consultando la tabla real `catalogo_tarifas` en **BigQuery (`psa_governance_<PARTICIPANT_ID>`)** y registrando el dictamen con `idempotency_key` en `registro_dictamenes_psa`.

---

## 6. Sistema de Diseño Material Design 3 — Identidad Retail (`DESIGN.md`)

- Toda interfaz web generada en este proyecto **DEBE** seguir las especificaciones de [DESIGN.md](./DESIGN.md) y el skill `.agents/skills/retail-material-design/SKILL.md` (Rojo Primario `#C8102E` / `#D51B24`, Amarillo/Oro `#F6A800`, superficies tonales M3 y soporte completo para **Tema Claro** y **Tema Oscuro**).
