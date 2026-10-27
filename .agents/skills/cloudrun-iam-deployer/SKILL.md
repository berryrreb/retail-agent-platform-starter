---
name: cloudrun-iam-deployer
description: >-
  Configura el modelo IAM de 3 niveles en GCP (Administrador del Proyecto vs.
  Participante/Desarrollador sin permisos excesivos vs. Service Account dedicada del Agente
  conforme a la Regla 2 del Blueprint de Interconexión IA), aísla recursos por PARTICIPANT_ID
  para trabajo concurrente, genera el Dockerfile y despliega en Cloud Run con escala a cero (--min-instances=0).
---

# Skill: IAM de 3 Niveles, Aislamiento Multi-Usuario y Despliegue Seguro en Cloud Run (Regla 2)

Este skill implementa la **Regla 2 del Blueprint de Interconexión IA** y garantiza que múltiples personas puedan trabajar simultáneamente en el **mismo proyecto de GCP** sin colisionar ni dejar costos residuales:

> *"El agente actúa con identidad propia y permisos mínimos. Nunca hereda las credenciales del usuario ni una cuenta de servicio genérica de amplio alcance."*

---

## 1. Modelo IAM de 3 Niveles (Principio de Mínimo Privilegio)

### Nivel 1 — Administrador del Proyecto GCP (Quien prepara el proyecto en la Consola de GCP)
- **Quién es**: El administrador de infraestructura / Cloud Admin que habilita las APIs, crea las Service Accounts de los agentes y otorga los permisos acotados a los participantes desde la **Consola Web de GCP (`console.cloud.google.com`)**.
- **Roles requeridos en el proyecto**:
  - `Project IAM Admin` (`roles/resourcemanager.projectIamAdmin`)
  - `Service Usage Admin` (`roles/serviceusage.serviceUsageAdmin`)
  - `Service Account Admin` (`roles/iam.serviceAccountAdmin`)
  - `Artifact Registry Administrator` (`roles/artifactregistry.admin`)

### Nivel 2 — Participante del Taller / Desarrollador (Quien construye el agente en Antigravity)
- **Quién es**: Cada arquitecto o desarrollador que participa en la sesión práctica (autenticado en local vía `gcloud auth application-default login` — **ADC**).
- **Roles asignados en el proyecto (SIN permisos administrativos ni de IAM)**:
  - `Vertex AI User` (`roles/aiplatform.user`) — Invocar `gemini-3.8-flash` y `gemini-4-argon`, y usar `VertexAiSessionService` / `VertexAiMemoryBankService`.
  - `BigQuery Data Editor` (`roles/bigquery.dataEditor`) y `BigQuery Job User` (`roles/bigquery.jobUser`) — Crear su dataset aislado `psa_governance_<participant_id>` en `us-west1`, poblar las tablas de catálogo y consultar dictámenes en BigQuery.
  - `Cloud Run Developer` (`roles/run.developer`) — Desplegar su servicio en Cloud Run (**no** es `Cloud Run Admin`; no puede modificar políticas IAM globales).
  - `Cloud Build Editor` (`roles/cloudbuild.builds.editor`) — Construir la imagen del contenedor al desplegar.
  - `Artifact Registry Writer` (`roles/artifactregistry.writer`) — Subir imágenes de contenedor (**no** puede borrar repositorios).
  - `Service Usage Consumer` (`roles/serviceusage.serviceUsageConsumer`) — Usar cuotas del proyecto con ADC (`set-quota-project`).
  - `Logs Viewer` (`roles/logging.viewer`) — Consultar logs y trazas en la consola.
- **Permiso a nivel recurso (Resource-Level IAM Binding)**:
  - `Service Account User` (`roles/iam.serviceAccountUser`) otorgado **únicamente sobre su propia Service Account** (`sa-psa-<participant_id>@<PROJECT_ID>.iam.gserviceaccount.com`), **nunca** a nivel proyecto. Así ningún participante puede suplantar la cuenta de servicio de otro participante ni cuentas productivas.
- **Permisos que NUNCA recibe el participante**:
  - ❌ `Owner` / `Editor`
  - ❌ `Service Account Admin` / `Project IAM Admin`
  - ❌ `Service Account Key Admin` (prohibido crear llaves JSON descargables)

### Nivel 3 — Identidad Propia del Agente en Cloud Run (`sa-psa-<participant_id>`)
- **Quién es**: La Service Account dedicada adjunta al servicio `psa-agent-<participant_id>` en Cloud Run.
- **Roles asignados en el proyecto**:
  - `Vertex AI User` (`roles/aiplatform.user`)
  - `BigQuery Data Editor` (`roles/bigquery.dataEditor`)
  - `BigQuery Job User` (`roles/bigquery.jobUser`)
  - `Logs Writer` (`roles/logging.logWriter`)
  - `Cloud Trace Agent` (`roles/cloudtrace.agent`)

---

## 2. Aislamiento Multi-Participante ("Que nadie se pise los talones")

Cada participante define su identificador único `PARTICIPANT_ID` (por ejemplo, `jperez` o `ana`):

| Recurso en el Proyecto Compartido | Nombre Aislado por Participante | Etiqueta de Costos / Limpieza |
| :--- | :--- | :--- |
| **Service Account del Agente** | `sa-psa-${PARTICIPANT_ID}@${PROJECT_ID}.iam.gserviceaccount.com` | Prefijo `sa-psa-` |
| **Dataset de BigQuery (`us-west1`)** | `psa_governance_${PARTICIPANT_ID}` | `workshop=psa-agent-lab,participant=${PARTICIPANT_ID}` |
| **Vertex AI Agent Engine (Sessions & Memory)** | `psa-engine-${PARTICIPANT_ID}` | Prefijo `psa-engine-` |
| **Namespace ADK (`APP_NAME`)** | `retail-psa-portal-${PARTICIPANT_ID}` | Aislado por `PARTICIPANT_ID` |
| **Servicio Cloud Run** | `psa-agent-${PARTICIPANT_ID}` | `workshop=psa-agent-lab,participant=${PARTICIPANT_ID}` |

---

## 3. Generación de `Dockerfile` y Despliegue a Cloud Run con Escala a Cero

Cuando el participante pida preparar y desplegar el agente a Cloud Run:
1. Genera un `Dockerfile` ligero basado en `python:3.11-slim` que instale `requirements.txt`, copie `psa_agent/`, `data/`, `static/` y `main.py`, y arranque con `uvicorn main:app --host 0.0.0.0 --port ${PORT:-8080}`.
2. Despliega explicando cada flag al participante:
   ```bash
   gcloud run deploy psa-agent-${PARTICIPANT_ID} \
     --source . \
     --project "${GOOGLE_CLOUD_PROJECT}" \
     --region "${GOOGLE_CLOUD_LOCATION}" \
     --service-account "sa-psa-${PARTICIPANT_ID}@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" \
     --min-instances 0 \
     --max-instances 2 \
     --cpu 1 \
     --memory 512Mi \
     --labels "workshop=psa-agent-lab,participant=${PARTICIPANT_ID}" \
     --set-env-vars "GOOGLE_GENAI_USE_VERTEXAI=TRUE,GOOGLE_CLOUD_PROJECT=${GOOGLE_CLOUD_PROJECT},GOOGLE_CLOUD_LOCATION=${GOOGLE_CLOUD_LOCATION},PARTICIPANT_ID=${PARTICIPANT_ID},BQ_DATASET=${BQ_DATASET},MODEL_FAST=gemini-3.8-flash,MODEL_REASONING=gemini-4-argon,AGENT_ENGINE_ID=${AGENT_ENGINE_ID}" \
     --allow-unauthenticated
   ```
   - `--min-instances 0` garantiza que cuando nadie esté usando el navegador, el costo de cómputo sea **$0.00**.
   - `--service-account` asigna la identidad dedicada sin usar API keys.
