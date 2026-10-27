---
name: adk-platform-scaffolder
description: >-
  Crea y configura desde cero agentes con Google ADK (google-adk), su Ficha de Catálogo (Agent Card Sección 6.2)
  y el backend FastAPI siguiendo el Blueprint de Interconexión IA y el Google Cloud Well-Architected Framework:
  autenticación vía ADC (sin API keys), sesiones con VertexAiSessionService, memoria con VertexAiMemoryBankService
  y uso exclusivo de gemini-3.8-flash y gemini-4-argon.
---

# Skill: Creación de Agentes Google ADK, Ficha de Catálogo (Sección 6.2) y Runner en Vertex AI

Utiliza este skill cuando el participante pida inicializar desde cero el paquete del agente (`psa_agent/`), configurar Sesiones y Memoria en **Vertex AI Agent Engine**, o crear el servidor web `main.py` con **FastAPI**.

## 1. Reglas de Configuración (Blueprint de Interconexión IA + GCP Well-Architected)

1. **Portafolio Gobernado de Modelos (Ruteo por Tarea — Sección 6.6 y 6.7)**:
   - Lee siempre los modelos desde variables de entorno:
     ```python
     MODEL_FAST = os.getenv("MODEL_FAST", "gemini-3.8-flash")
     MODEL_REASONING = os.getenv("MODEL_REASONING", "gemini-4-argon")
     ```
   - Nunca uses otro modelo fuera de estos dos.
2. **Autenticación con ADC (Cero API Keys — Regla 2)**:
   - El archivo `.env` debe configurar siempre:
     ```ini
     GOOGLE_GENAI_USE_VERTEXAI=TRUE
     GOOGLE_CLOUD_PROJECT=<PROJECT_ID>
     GOOGLE_CLOUD_LOCATION=us-west1
     PARTICIPANT_ID=<PARTICIPANT_ID>
     BQ_DATASET=psa_governance_<PARTICIPANT_ID_CON_GUION_BAJO>
     MODEL_FAST=gemini-3.8-flash
     MODEL_REASONING=gemini-4-argon
     AGENT_ENGINE_ID=
     ```
   - Nunca incluyas `GOOGLE_API_KEY`. En local el SDK de Vertex AI y el cliente de **BigQuery (`google-cloud-bigquery`)** toman las credenciales de `gcloud auth application-default login` (ADC), y en Cloud Run toman el token OIDC/IAM de la Service Account adjunta al servicio (`sa-psa-<PARTICIPANT_ID>`).

## 2. Ficha Mínima del Catálogo de Agentes (Sección 6.2 del Blueprint)

El Blueprint establece en su **Sección 6.2**: *"Si un agente no está registrado con su ficha mínima (owner, versión, risk tier, datos y evaluación), no llega a producción"*.
Por ello, incluye en `psa_agent/agent.py` el diccionario `AGENT_CATALOG_CARD` y expónlo en `GET /api/config`:

```python
AGENT_CATALOG_CARD = {
    "agent_id": f"retail-psa-architect-{os.getenv('PARTICIPANT_ID', 'dev')}",
    "version": "1.0.0",
    "owner_squad": "Arquitectura Empresarial & Plataforma IA",
    "max_risk_tier": "L2 (Escritura reversible de borradores PSA/Quote; L4 bloqueado a Propose & Approve)",
    "orchestration_patterns": ["Consulta gobernada", "Ejecución acotada", "Propose & approve"],
    "models": {
        "orchestrator_fast": MODEL_FAST,
        "validator_reasoning": MODEL_REASONING,
    },
    "gateway_posture": "Cloud Run IAM OIDC + Cloud API Gateway (Fase 1 Ágil sin Apigee) -> Enterprise API/MCP Gateway (Escala)",
    "sla_p95_seconds": 15,
}
```

## 3. Estructura del Paquete `psa_agent/` para `adk web` y `adk run`

Para que el CLI oficial `adk web` y `adk run psa_agent` descubra el agente automáticamente:
- `psa_agent/__init__.py` debe exportar `root_agent`:
  ```python
  from .agent import root_agent

  __all__ = ["root_agent"]
  ```
- `psa_agent/agent.py` debe definir:
  - El sub-agente auditor del ARB y Well-Architected Framework `psa_architecture_validator_agent` (`LlmAgent` usando `MODEL_REASONING = "gemini-4-argon"`).
  - El agente coordinador raíz `retail_psa_architect_agent` (`LlmAgent` usando `MODEL_FAST = "gemini-3.8-flash"`, `PreloadMemoryTool()` y las 4 herramientas deterministas conectadas a `data/*.json`).
  - El alias `root_agent = retail_psa_architect_agent`.

## 4. Sesiones y Memoria 100% Reales en Vertex AI Agent Engine (`us-west1` — Sin Mocks en Memoria)

En `main.py`, conecta directamente `VertexAiSessionService` y `VertexAiMemoryBankService` contra la instancia real de **Vertex AI Agent Engine (`AGENT_ENGINE_ID`)** en `us-west1` (Oregon), sin usar mocks ni servicios `InMemory`:

```python
from google.adk.sessions import VertexAiSessionService
from google.adk.memory import VertexAiMemoryBankService
from google.adk.runners import Runner

if not AGENT_ENGINE_ID:
    raise RuntimeError(
        "AGENT_ENGINE_ID es obligatorio en .env para conectar con Vertex AI Agent Engine real en us-west1 (sin mocks)."
    )

session_service = VertexAiSessionService(
    project=PROJECT_ID,
    location=LOCATION,
    agent_engine_id=AGENT_ENGINE_ID,
)
memory_service = VertexAiMemoryBankService(
    project=PROJECT_ID,
    location=LOCATION,
    agent_engine_id=AGENT_ENGINE_ID,
)
persistence_mode = f"Vertex AI Agent Engine ({AGENT_ENGINE_ID} en {LOCATION})"

runner = Runner(
    app_name=f"retail-psa-portal-{PARTICIPANT_ID}",
    agent=root_agent,
    session_service=session_service,
    memory_service=memory_service,
)
```

## 5. Contrato de Endpoints del Backend (`main.py`)

Cuando crees `main.py` con FastAPI, expón los siguientes endpoints para la interfaz web:
- `GET /api/config`: Devuelve `project_id`, `location`, `participant_id`, `app_name`, `model_fast`, `model_reasoning`, `persistence_mode`, `auth_mode` (`"ADC / IAM Service Account (Zero API Keys)"`) y `agent_card` (`AGENT_CATALOG_CARD`).
- `GET /api/samples`: Lee y devuelve los casos de prueba desde `data/ejemplos_minutas.json`.
- `GET /api/sessions?user_id=...`: Lista las sesiones activas del usuario desde `session_service.list_sessions(...)`.
- `POST /api/sessions`: Crea una nueva sesión en `session_service.create_session(...)`.
- `POST /api/memory/consolidate`: Obtiene la sesión actual con `session_service.get_session(...)` y la guarda en el banco de memoria de largo plazo con `await memory_service.add_session_to_memory(session)`.
- `GET /api/memories?user_id=...&query=...`: Consulta recuerdos almacenados usando `await memory_service.search_memory(...)`.
- `POST /api/chat`: Ejecuta `runner.run_async(user_id=..., session_id=..., new_message=...)`, recolecta los eventos de herramientas invocadas (`get_function_calls()` / `get_function_responses()`), extrae el bloque JSON estructurado (`artifact_json`) si la respuesta lo incluye, y devuelve `{ "reply": ..., "tool_traces": [...], "artifact": ..., "session_id": ... }`.
