# Auditoría_appV2 — Security Architecture Compliance Dashboard
## Especificación del Proyecto (`spec.md`)

---

| Campo              | Detalle                                            |
|--------------------|----------------------------------------------------|
| **Proyecto**       | Auditoría_appV2                                            |
| **Versión**        | 2.0 (parcheada)                                    |
| **Tipo**           | Single-Page Application (SPA) — offline / local    |
| **Tecnología**     | HTML5 · Vanilla JavaScript · Tailwind CSS · Chart.js |
| **Autor**          | Arquitecto de Ciberseguridad / ex-CISO             |
| **Contexto**       | Herramienta personal de portafolio profesional     |
| **Repositorio**    | GitHub (privado/público según preferencia)         |

---

## 1. Contexto y Motivación

SecArch nació como una herramienta de portafolio para demostrar capacidad práctica en arquitectura de seguridad y gestión de cumplimiento normativo. La mayoría de los dashboards de auditoría disponibles en el mercado son soluciones SaaS costosas o herramientas empresariales que requieren infraestructura. SecArch demuestra que es posible construir una herramienta funcional, profesional y segura que corra completamente en el navegador, sin dependencias de servidor.

El proyecto también sirve como caso de estudio de **Secure Development Lifecycle (SDLC)**, ya que incluye análisis SAST post-construcción y versión parcheada con los hallazgos corregidos.

---

## 2. Objetivo

Proporcionar a un Arquitecto de Ciberseguridad o CISO una herramienta local para:

- **Administrar controles de auditoría** de múltiples marcos normativos desde una sola interfaz.
- **Visualizar el estado de cumplimiento** por framework de forma ejecutiva y técnica.
- **Generar análisis inteligente** del estado de los controles usando IA generativa.
- **Exportar evidencia** en formato CSV para reportes de auditoría.
- **Documentar** la postura de seguridad con campos de riesgo, owner, fechas de revisión y notas.

---

## 3. Alcance

### 3.1 Marcos normativos soportados

| Marco        | Descripción                                             | Contexto      |
|--------------|---------------------------------------------------------|---------------|
| ISO 27001    | Sistema de Gestión de Seguridad de la Información       | Internacional |
| NIST CSF     | Cybersecurity Framework del NIST                        | Internacional |
| PCI DSS      | Payment Card Industry Data Security Standard            | Internacional |
| CNBV         | Circular Única de Bancos (Comisión Nacional Bancaria)   | 🇲🇽 México     |
| Banxico      | Circular 3/2012 y disposiciones de pagos SPEI           | 🇲🇽 México     |

### 3.2 Funcionalidades incluidas

- ✅ CRUD completo de controles de auditoría
- ✅ Dashboard KPI con métricas de cumplimiento en tiempo real
- ✅ Rings de compliance animados por framework (Chart.js)
- ✅ Filtros por marco, estatus y búsqueda de texto libre
- ✅ Gestión de campos: ID, nombre, owner, riesgo, estatus, fecha revisión, evidencia, notas
- ✅ Indicador visual de controles con revisión vencida (+90 días)
- ✅ Highlight de controles críticos (riesgo Alto + Pendiente)
- ✅ Exportación CSV con protección contra formula injection
- ✅ Análisis de brechas por IA (integración con Anthropic Claude API)
- ✅ Persistencia local en `localStorage`
- ✅ Datos de ejemplo (demo data) para todos los marcos

### 3.3 Funcionalidades fuera de alcance (v2)

- ❌ Autenticación de usuarios
- ❌ Backend / base de datos
- ❌ Multi-usuario o compartición de datos
- ❌ Importación desde CSV/Excel
- ❌ Gestión de evidencias (adjuntos de archivos)
- ❌ Historial de cambios / audit trail
- ❌ Notificaciones / alertas programadas

---

## 4. Arquitectura

### 4.1 Diagrama de componentes

```
┌─────────────────────────────────────────────────────────┐
│                    Navegador (local)                    │
│                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌──────────────┐  │
│  │   UI Layer  │   │  Logic Layer│   │ Storage Layer│  │
│  │             │   │             │   │              │  │
│  │ - Header    │◄─►│ - renderTable│◄─►│ localStorage │  │
│  │ - KPI Cards │   │ - renderKPIs │   │ (JSON)       │  │
│  │ - Rings     │   │ - renderRings│   │              │  │
│  │ - Table     │   │ - CRUD ops  │   └──────────────┘  │
│  │ - Modals    │   │ - Filters   │                      │
│  │ - AI Modal  │   │ - Export    │   ┌──────────────┐  │
│  └─────────────┘   └─────────────┘   │External APIs │  │
│                                      │              │  │
│  ┌──────────────────────────────┐    │ Anthropic    │  │
│  │     Dependencias CDN         │    │ api.anthr..  │  │
│  │ - Tailwind CSS (estilos)     │    │ (solo IA)    │  │
│  │ - Chart.js 4.4.0 (gráficas) │    └──────────────┘  │
│  │ - Google Fonts (tipografía)  │                      │
│  └──────────────────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Modelo de datos

Cada control de auditoría se almacena como un objeto JSON con el siguiente esquema:

```json
{
  "estandar":      "string  — ISO 27001 | NIST CSF | PCI DSS | CNBV | Banxico",
  "id":            "string  — Identificador del control (ej. A.5.1, ID.AM-1)",
  "nombre":        "string  — Nombre descriptivo del control",
  "owner":         "string  — Responsable del control",
  "riesgo":        "enum    — Alto | Medio | Bajo",
  "estatus":       "enum    — Activo | Parcial | Pendiente",
  "fechaRevision": "string  — Fecha ISO 8601 (YYYY-MM-DD) o vacío",
  "evidencia":     "string  — Nombre de archivo de evidencia o N/A",
  "notas":         "string  — Observaciones operacionales (max 500 chars)"
}
```

**Almacenamiento:** `localStorage` bajo la clave `secarch_v2` como array JSON serializado.

### 4.3 Flujo de la función de análisis IA

```
Usuario abre modal IA
        │
        ▼
¿Tiene API Key guardada?
    │           │
   SÍ          NO
    │           │
    │     Solicitar key → guardar en localStorage
    │
    ▼
Agregar aviso de transmisión de datos ⚠️
        │
        ▼
Construir prompt con estadísticas agregadas:
- Total controles por marco
- % cumplimiento por framework
- Lista de controles críticos pendientes
        │
        ▼
POST https://api.anthropic.com/v1/messages
  model: claude-sonnet-4-20250514
  max_tokens: 1000
        │
        ▼
Mostrar respuesta via textContent (no innerHTML)
```

---

## 5. Seguridad

### 5.1 Controles de seguridad implementados (v2 parcheada)

| Control                     | Implementación                                      |
|-----------------------------|-----------------------------------------------------|
| XSS Prevention              | Función `esc()` — HTML entity encoding en todos los puntos de salida dinámicos |
| Attribute Injection         | Event delegation con `data-*` attributes numéricos. Sin interpolación de datos en atributos HTML |
| Content Security Policy     | Meta CSP que restringe orígenes de scripts, estilos, fuentes y conexiones |
| Subresource Integrity       | Hash SHA-384 verificado en Chart.js 4.4.0           |
| CSV Formula Injection       | Función `csvSafe()` — prefijo `'` en valores con inicio de fórmula |
| Input Validation            | Atributos `maxlength` en todos los campos del formulario |
| Secure DOM Manipulation     | `textContent` en lugar de `innerHTML` para contenido dinámico de usuario |
| Data Disclosure Notice      | Aviso explícito antes de transmitir datos a API externa |

### 5.2 Análisis SAST

Se realizó análisis SAST sobre la versión v2 original. Se identificaron y corrigieron 10 hallazgos (1 crítico, 3 altos, 4 medios, 1 bajo, 1 informativo).

Ver: [`SAST_Report_Auditoría_appv2.md`](./SAST_Report_Auditoría_appv2.md)

### 5.3 Consideraciones de privacidad

- Los datos de auditoría se almacenan localmente en el navegador del usuario.
- La función de análisis IA envía **únicamente estadísticas agregadas** a la API de Anthropic (conteos por framework, nombres de controles pendientes). No envía datos de negocio, credenciales ni información personal.
- La API Key de Anthropic se almacena en `localStorage` (sin cifrado). Se recomienda no compartir el perfil del navegador ni el `localStorage` con terceros.

---

## 6. Stack Tecnológico

| Componente       | Tecnología                  | Versión  | CDN/Fuente                      |
|------------------|-----------------------------|----------|---------------------------------|
| Framework CSS    | Tailwind CSS                | Latest   | cdn.tailwindcss.com             |
| Gráficas         | Chart.js                    | 4.4.0    | cdn.jsdelivr.net (con SRI)      |
| Tipografía       | Inter + JetBrains Mono      | —        | fonts.googleapis.com            |
| IA               | Anthropic Claude API        | —        | api.anthropic.com               |
| Runtime          | Vanilla JavaScript (ES2020+)| —        | —                               |
| Persistencia     | Web Storage API             | —        | Nativo del navegador            |

**Dependencias de runtime:** Ninguna (sin `node_modules`, sin build step).  
**Compatibilidad:** Chrome 90+, Firefox 88+, Safari 14+, Edge 90+.

---

## 7. Instalación y Uso

No requiere instalación. Es un archivo HTML autocontenido.

```bash
# Opción 1: Abrir directamente
open SecArch_v2_patched.html

# Opción 2: Servidor local (recomendado para restricciones de CSP en algunos navegadores)
python3 -m http.server 8080
# Navegar a: http://localhost:8080/SecArch_v2_patched.html
```

### Primera ejecución
1. Al abrir, se cargan datos de ejemplo para los 5 marcos normativos.
2. Los cambios se guardan automáticamente en `localStorage`.
3. Para el análisis IA, ir al botón 🤖 e ingresar una API Key de Anthropic.

---

## 8. Estructura del Repositorio

```
secarch/
├── SecArch_v2.html            # Versión funcional original
├── SecArch_v2_patched.html    # Versión con parches de seguridad aplicados
├── SAST_Report_SecArch_v2.md  # Reporte de análisis de seguridad estático
├── spec.md                    # Este documento
└── README.md                  # Descripción general del proyecto
```

---

## 9. Roadmap

| Versión | Feature                                              | Prioridad |
|---------|------------------------------------------------------|-----------|
| v2.1    | Eliminar `unsafe-inline` del CSP (archivos externos) | Media     |
| v2.2    | Cifrado de `localStorage` con Web Crypto API         | Alta      |
| v2.3    | Importación de controles desde CSV                   | Media     |
| v3.0    | Backend ligero + autenticación (modo multi-usuario)  | Baja      |
| v3.1    | Audit trail / historial de cambios por control       | Media     |
| v3.2    | Integración con APIs de GRC (ServiceNow, Archer)     | Baja      |

---

## 10. Contexto Regulatorio Mexicano

SecArch incluye soporte nativo para los marcos regulatorios más relevantes del sistema financiero mexicano:

**CNBV — Comisión Nacional Bancaria y de Valores**
- Circular Única de Bancos (CUB) — Artículos 164, 168, 172, 175
- Disposiciones sobre gestión de riesgos tecnológicos, continuidad de negocio y auditoría de sistemas

**Banxico — Banco de México**
- Circular 3/2012 y sus modificaciones
- Disposiciones de seguridad para sistemas de pago SPEI
- Controles de cifrado, acceso y trazabilidad en mensajería financiera

Estos marcos se complementan con ISO 27001, NIST CSF y PCI DSS para proporcionar una vista integral del cumplimiento en instituciones financieras reguladas en México.

---

*Auditoría_appV2 — Construido como herramienta de portafolio profesional en ciberseguridad.*  
*Versión 2.0 | Junio 2025*
