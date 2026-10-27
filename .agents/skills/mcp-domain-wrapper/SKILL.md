---
name: mcp-domain-wrapper
description: >-
  Diseña y genera servidores MCP de dominio y herramientas ADK sobre APIs/catálogos existentes
  cumpliendo las Reglas 1 a 5, los 4 tramos de exposición (Sección 7), el Contrato Mínimo de Herramienta (Sección 7.1),
  el Control Transaccional (Sección 8.2) y la estrategia pragmática de Gateway en GCP (sin costo/complejidad de Apigee
  en fase inicial, usando Cloud Run IAM OIDC + Cloud API Gateway, con ruta evolutiva a Gateway Empresarial).
---

# Skill: Servidor MCP de Dominio, Contrato Mínimo 7.1 y Estrategia Pragmática de Gateway

En la arquitectura del **Blueprint de Plataforma de Interconexión IA** (**Reglas 1, 3, 4 y 5** y **Sección 7**), el camino desde un sistema empresarial hasta el agente tiene **cuatro tramos obligatorios**:
1. **API REST / Catálogo existente**: El contrato técnico que ya opera el sistema core (no se modifica).
2. **Servidor MCP de Dominio**: Fachada delgada mantenida por el equipo dueño del sistema (**Regla 3**) que traduce endpoints técnicos a herramientas con significado de negocio.
3. **Capa de Gateway (Estrategia en 2 Horizontes — Pilar FinOps del Well-Architected Framework)**:
   - **Horizonte 1 (Adopción Ágil / Ejercicio — Sin Apigee por costo y complejidad)**: Autenticación **Cloud Run IAM (OIDC máquina-a-máquina)** donde la Service Account del agente (`sa-psa-<participante>`) se autentica contra el servicio de dominio, complementado con **Cloud API Gateway** (serverless pago por uso) o **Cloud Load Balancing + Cloud Armor** para cuotas y protección perimetral.
   - **Horizonte 2 (Escala Corporativa Multi-Dominio)**: El componente de **Enterprise API & MCP Gateway (como Apigee X o el API Gateway corporativo ya instalado)** se incorpora cuando el volumen de agentes y dominios exige gobierno centralizado cross-enterprise, sin necesidad de modificar el código de los agentes ni de los servidores MCP.
4. **Agente de Dominio**: Descubre e invoca únicamente las herramientas permitidas por su *Risk Tier*.

---

## 1. Contrato Mínimo de una Herramienta MCP (Sección 7.1 del Blueprint)

Ninguna herramienta se expone al agente sin declarar en su docstring y devolver en su respuesta estructurada estos **8 campos obligatorios**:

| Campo del Contrato (7.1) | Qué debe implementar la función en Python |
| :--- | :--- |
| **1. Semántica de Negocio** | Nombre y docstring en lenguaje de negocio (ej. `calculate_deterministic_quote` o `get_retail_capability_map`, nunca nombres crudos de tablas o endpoints). |
| **2. Esquema Tipado** | Argumentos y retorno con tipos explícitos (`str`, `int`, `list[dict]`, modelos `Pydantic`). |
| **3. Clasificación y Risk Tier (`L0`–`L4`)** | Declarar el nivel exacto:<br>• `L0`: Lectura no sensible (documentación, mapa de capacidades).<br>• `L1`: Lectura sensible (transacciones, datos con PII/PAN -> exige redacción DLP).<br>• `L2`: Escritura/generación reversible (borrador de PSA, cotización formal).<br>• `L3`: Escritura con impacto operativo (reorden, ajuste acotado).<br>• `L4`: Escritura financiera o en sistema crítico (MPE, ERP, ajustes contables) -> **Prohibido ejecutar de forma autónoma (Regla 4)**; solo patrón *Propose & Approve*. |
| **4. Patrón de Orquestación (Sección 8)** | Declarar a qué patrón pertenece: `"Consulta gobernada"`, `"Ejecución acotada"`, `"Propose & approve"`, `"Orquestación multi-sistema"` o `"Reactivo a eventos"`. |
| **5. Idempotencia (Sección 8.2)** | Toda operación `L2`–`L4` recibe obligatoriamente `idempotency_key: str` para evitar duplicación ante reintentos. |
| **6. Correlación y Trazabilidad (Sección 12.1)** | Generar y devolver un `correlation_id: str` (`X-Correlation-ID`) para vincular la traza del agente con el log del sistema. |
| **7. Redacción de Datos Sensibles** | Enmascarar `PAN` (`[PAN_REDACTED]`), correos (`[EMAIL_REDACTED]`) y teléfonos (`[PHONE_REDACTED]`) antes de devolver datos al contexto del LLM. |
| **8. Ownership de Dominio (Regla 3 y 5)** | Incluir `owner_domain` (ej. `"Arquitectura Empresarial"`, `"Finanzas & Estimación"`, `"Expansión"`) y `sla_target`. |
