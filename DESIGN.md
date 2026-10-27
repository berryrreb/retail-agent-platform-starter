# Retail Material Design 3 (M3) — Design System Specification (`DESIGN.md`)

Este documento define el sistema de diseño obligatorio para todas las interfaces web de la **Plataforma de Interconexión IA**. Está basado en **Material Design 3 (Material You)** con paleta institucional cálida (Rojo Carmesí `#C8102E` / `#D51B24`, Amarillo Oro `#F6A800` y superficies neutrales cálidas), con soporte nativo para **Modo Claro (Light Theme)** y **Modo Oscuro (Dark Theme)**.

---

## 1. Principios de Diseño (Anti-Generic AI UI)

1. **Identidad Corporativa Auténtica, No "Plantilla IA"**:
   - **PROHIBIDO**: Gradientes púrpura/violeta/cian (`linear-gradient(135deg, #6366f1, #a855f7)`), fondos espaciales `#0f172a`, bordes brillantes de neón o estéticas genéricas de SaaS.
   - **OBLIGATORIO**: Uso de la franja institucional superior (Rojo `#D51B24`, Amarillo Oro `#F6A800` y Blanco), superficies tonales cálidas de Material Design 3 y jerarquía tipográfica clara para arquitectura y finanzas.
2. **Elevación Tonal Material 3 (Tonal Surface Hierarchy)**:
   - En lugar de abusar de sombras pesadas (`drop-shadow`) o `backdrop-filter: blur()`, la jerarquía entre paneles se construye mediante los niveles de superficie de Material 3: `surface-dim`, `surface`, `surface-container-low`, `surface-container`, `surface-container-high` y `surface-container-highest`.
3. **Densidad Ejecutiva y Legibilidad Técnica**:
   - Los diagramas de arquitectura C4, tablas de cotización y matrices de riesgo (L0–L4) deben leerse de un vistazo, con números tabulares (`font-variant-numeric: tabular-nums`) para alinear cifras en MXN al centavo.

---

## 2. Tokens de Color Material Design 3 (Light & Dark Themes)

### 2.1 Tema Claro (`[data-theme="light"]` — Default)

| Token M3 | Valor Hex | Rol en la Interfaz |
| :--- | :--- | :--- |
| `--md-sys-color-primary` | `#C8102E` | Acciones primarias, botones principales (Filled Buttons), indicadores activos |
| `--md-sys-color-on-primary` | `#FFFFFF` | Texto e iconos sobre color primario |
| `--md-sys-color-primary-container` | `#FFDAD8` | Chips activos, badges de selección, fondos de énfasis primario |
| `--md-sys-color-on-primary-container` | `#410006` | Texto sobre `primary-container` |
| `--md-sys-color-secondary` | `#855300` | Énfasis secundario accesible derivado del Amarillo Oro |
| `--md-sys-color-secondary-container` | `#FFDEA8` | Contenedores de acento cálido, etiquetas `⟨propuesto: …⟩`, alertas de negocio |
| `--md-sys-color-on-secondary-container` | `#2A1700` | Texto de alto contraste sobre contenedores amarillos/ámbar |
| `--md-sys-color-brand-yellow` | `#F6A800` | Barra institucional superior, acentos gráficos de marca |
| `--md-sys-color-background` | `#FAF8F6` | Fondo base cálido de la aplicación |
| `--md-sys-color-surface` | `#FFFFFF` | Tarjetas principales, lienzo de documento PSA y diagramas C4 |
| `--md-sys-color-surface-container-low` | `#F5F2EF` | Panel lateral de chat, fondos secundarios |
| `--md-sys-color-surface-container` | `#EFEBE7` | Encabezados de tabla, bloques de código, contenedores de inputs |
| `--md-sys-color-surface-container-high` | `#E8E2DD` | Hover states, separadores activos |
| `--md-sys-color-on-surface` | `#1E1B1A` | Texto principal de alta legibilidad |
| `--md-sys-color-on-surface-variant` | `#534342` | Subtítulos, metadatos, etiquetas de campo |
| `--md-sys-color-outline` | `#857371` | Bordes de inputs y botones outlined |
| `--md-sys-color-outline-variant` | `#D8C2BF` | Líneas divisorias sutiles de tablas y tarjetas |

### 2.2 Tema Oscuro (`[data-theme="dark"]`)

| Token M3 | Valor Hex | Rol en la Interfaz |
| :--- | :--- | :--- |
| `--md-sys-color-primary` | `#FFB3AF` | Acciones primarias en modo oscuro (accesibilidad AA/AAA sobre fondo oscuro) |
| `--md-sys-color-on-primary` | `#68000F` | Texto sobre botón primario en modo oscuro |
| `--md-sys-color-primary-container` | `#930019` | Contenedor primario rojo profundo |
| `--md-sys-color-on-primary-container` | `#FFDAD8` | Texto claro sobre `primary-container` |
| `--md-sys-color-secondary` | `#FABC2A` | Amarillo/Oro luminoso para modo oscuro |
| `--md-sys-color-secondary-container` | `#643E00` | Contenedor ámbar oscuro para etiquetas `⟨propuesto: …⟩` |
| `--md-sys-color-on-secondary-container` | `#FFDEA8` | Texto sobre `secondary-container` |
| `--md-sys-color-brand-yellow` | `#F6A800` | Franja institucional |
| `--md-sys-color-background` | `#161212` | Fondo base carbón cálido |
| `--md-sys-color-surface` | `#1F1A1A` | Superficie principal de tarjetas y paneles |
| `--md-sys-color-surface-container-low` | `#262020` | Panel de conversación y barra superior |
| `--md-sys-color-surface-container` | `#2E2727` | Celdas de agrupación, cajas de herramientas e inputs |
| `--md-sys-color-surface-container-high` | `#393131` | Hover states y chips elevados |
| `--md-sys-color-on-surface` | `#ECE0DF` | Texto principal en modo oscuro |
| `--md-sys-color-on-surface-variant` | `#D8C2BF` | Texto secundario y etiquetas |
| `--md-sys-color-outline` | `#A08C8A` | Bordes de controles |
| `--md-sys-color-outline-variant` | `#534342` | Divisores de tablas y contenedores |

---

## 3. Semántica de Niveles de Riesgo (Risk Tiers L0–L4 del Blueprint)

Cada capacidad o herramienta expuesta debe mostrar su badge semántico de nivel de riesgo:

- **L0 (Lectura no sensible)**: Verde esmeralda M3 (`#146C2E` Light / `#81C995` Dark)
- **L1 (Lectura sensible con DLP)**: Azul petróleo M3 (`#00639B` Light / `#7FCFFF` Dark)
- **L2 (Escritura reversible / Idempotente)**: Ámbar (`#855300` Light / `#FABC2A` Dark)
- **L3 (Escritura con impacto operativo)**: Naranja intenso (`#B35000` Light / `#FFB683` Dark)
- **L4 (Escritura financiera / Crítica — Prohibido autónomo, Propose & Approve)**: Rojo de alerta (`#B3261E` Light / `#F2B8B5` Dark)

---

## 4. Tipografía, Geometría y Componentes M3

- **Fuentes**:
  - **UI & Títulos**: `'Plus Jakarta Sans'`, `'Roboto'`, system-ui.
  - **Código, IDs, Mermaid y Montos**: `'JetBrains Mono'`, `'Roboto Mono'`, monospace con `font-variant-numeric: tabular-nums`.
- **Radios de Borde (Material 3 Shape Scale)**:
  - `--md-sys-shape-corner-small`: `8px` (Chips, badges L0–L4)
  - `--md-sys-shape-corner-medium`: `12px` (Inputs, tarjetas internas, bloques de métricas)
  - `--md-sys-shape-corner-large`: `16px` (Contenedores principales, tarjetas de sección PSA)
  - `--md-sys-shape-corner-full`: `9999px` (Botones primarios M3 Pill, selector de tema, Segmented Tabs)
- **Top App Bar Institucional**:
  - Incluye el emblema tipográfico en bloque rojo `#D51B24` con franja inferior amarilla `#F6A800`, indicadores de estado de plataforma en tiempo real (`ADC / Service Account`, `VertexAiSessionService`, `Modelo Activo`) y botón de cambio de tema Claro/Oscuro con icono Material Symbols.
