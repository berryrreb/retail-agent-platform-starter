---
name: retail-material-design
description: >-
  Diseña y construye desde cero la interfaz web en Material Design 3 (M3) con la identidad
  visual corporativa (temas claro y oscuro, tokens cromáticos Rojo #C8102E/#D51B24 y Amarillo Oro #F6A800,
  elevación tonal M3 y renderizado en vivo de tarjetas PSA, diagramas C4 Mermaid, tablas financieras /quote
  e inspector de Sesiones y Memoria). Úsalo siempre que crees o modifiques static/index.html, static/styles.css y static/app.js.
---

# Skill: Retail Material Design 3 (M3) UI System

Cuando construyas o edites cualquier interfaz web en este repositorio, **NUNCA utilices el estilo genérico de aplicaciones de IA** (gradientes morados/azules, fondos espaciales `#0f172a`, tarjetas de cristal con sombras de neón). Debes implementar estrictamente **Material Design 3 (Material You)** con la paleta institucional definida en [DESIGN.md](../../../DESIGN.md).

## 1. Reglas Visuales No Negociables

1. **Soporte Dual de Tema (Light & Dark Mode)**:
   - Define todos los colores mediante variables CSS (`--md-sys-color-*`) en `:root, [data-theme="light"]` y `[data-theme="dark"]` siguiendo exactamente la tabla de tokens de `DESIGN.md`.
   - Incluye siempre un botón accesible en el Top App Bar que permita alternar entre tema claro (`light_mode`) y oscuro (`dark_mode`) en vivo, persistiendo en `localStorage.getItem('psa-portal-theme')`.
2. **Identidad Institucional en el Top App Bar**:
   - Presenta la franja de identidad superior (`4px` con gradiente institucional Rojo `#C8102E` y Amarillo Oro `#F6A800`), el emblema `.retail-brand-pill`, chips con los modelos activos (`gemini-3.8-flash` / `gemini-4-argon`), el estado de autenticación (`ADC / Service Account`) y el selector de tema.
3. **Layout de Doble Panel (Split-Screen Workspace)**:
   - **Panel Izquierdo (Copiloto Conversacional + Casos Rápidos)**:
     - Selector de Sesión activa + botón `+ Nueva Sesión` + botón `Guardar en Memoria LP` (`POST /api/memory/consolidate`).
     - 3 chips de **Escenarios Rápidos en 1 Clic** (`/psa Minuta Conciliación POS`, `/quote Cotización Determinista`, `Apertura Tiendas PFERNANDEZ`).
     - Historial de conversación con chips de trazabilidad de herramientas invocadas (`clean_and_anonymize_transcript`, `get_retail_capability_map`, `calculate_deterministic_quote`).
     - Caja de texto con comando rápido y envío.
   - **Panel Derecho (Lienzo Interactivo de Entregables con 4 Pestañas M3)**:
     - **Pestaña 1: Documento PSA & Capacidades**: Renderiza Folio, los 4 roles de la Sección 1, capacidades mapeadas por dominio retail y la separación estricta entre **Sección 5.2 Infraestructura** y **Sección 5.4 Redes**.
     - **Pestaña 2: Diagramas C4 (Mermaid)**: Renderiza en vivo con `mermaid.js` los diagramas C4 Nivel 1 (Contexto), Nivel 2 (Contenedores) e Infraestructura/Redes.
     - **Pestaña 3: Cotización Determinista (`/quote`)**: Renderiza las 3 tarjetas KPI financieras (`Costo Mensual Recurrente`, `Anual Run-Rate = Mensual × 12`, `Total Proyecto`) y la tabla desglosada con `font-variant-numeric: tabular-nums`.
     - **Pestaña 4: Sesiones & Memoria Vertex AI**: Permite inspeccionar las sesiones del usuario y buscar recuerdos almacenados en `VertexAiMemoryBankService` para demostrar cómo un recuerdo guardado en la Sesión 1 persiste en la Sesión 2.
4. **Transformación Visual de `⟨propuesto: …⟩`**:
   - Todo texto que contenga `⟨propuesto: ...⟩` debe transformarse automáticamente en un **M3 Assist Chip** color ámbar (`--md-sys-color-secondary-container`) con ícono `auto_awesome` para que el arquitecto identifique al instante qué campos son inferencias propuestas vs. hechos confirmados.
