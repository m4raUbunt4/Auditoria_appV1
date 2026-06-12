# Auditoría-appV2 🛡️
### Auditoría_appV2 (Compliance Dashboard)

> Herramienta de gestión de controles de auditoría para Arquitectos de Ciberseguridad en entornos financieros regulados. Corre completamente en el navegador — sin servidor, sin instalación.

---

## ¿Qué es Auditoría_app?

Auditoría_app es un dashboard de cumplimiento normativo construido como **single-page application (SPA)** en HTML, CSS y JavaScript puro. Permite a un Arquitecto de Ciberseguridad o CISO administrar controles de auditoría bajo múltiples marcos normativos desde una sola interfaz, con visualización ejecutiva del estado de cumplimiento y análisis de brechas asistido por IA.

El proyecto nació como herramienta de portafolio profesional para demostrar la aplicación práctica de IA generativa en el contexto de la arquitectura de seguridad en el sector financiero mexicano.

---

## Marcos normativos soportados

| Marco | Descripción | Ámbito |
|-------|-------------|--------|
| **ISO 27001** | Sistema de Gestión de Seguridad de la Información | Internacional |
| **NIST CSF** | Cybersecurity Framework | Internacional |
| **PCI DSS** | Payment Card Industry Data Security Standard | Internacional |
| **CNBV** | Circular Única de Bancos — Arts. 164, 168, 172, 175 | 🇲🇽 México |
| **Banxico** | Circular 3/2012 — Disposiciones SPEI | 🇲🇽 México |

---

## Funcionalidades

- 📊 **Dashboard KPI** — métricas de cumplimiento en tiempo real por framework
- 🔵 **Rings de compliance** — visualización gráfica del avance por marco (Canvas 2D API nativa)
- ✅ **CRUD de controles** — gestión completa con campos de riesgo, owner, fecha de revisión, evidencia y notas
- ⚠️ **Alertas automáticas** — controles con revisión vencida (+90 días) y controles críticos (riesgo Alto + Pendiente)
- 🔍 **Filtros combinados** — por framework, estatus y búsqueda libre
- 🤖 **Análisis IA** — diagnóstico ejecutivo de brechas usando Claude (Anthropic API)
- ⬇️ **Exportación CSV** — con protección contra formula injection
- 💾 **Persistencia local** — datos guardados en `localStorage`, sin backend

---

## Demo

Al abrir el archivo se cargan automáticamente **25 controles de ejemplo** distribuidos en los 5 marcos normativos, incluyendo controles específicos de CNBV y Banxico para el sector financiero mexicano.

---

## Instalación

No requiere instalación ni dependencias. Es un único archivo HTML autocontenido.

```bash
# Opción 1: Abrir directamente en el navegador
open Auditoría_appv2_patched.html

# Opción 2: Servidor local (recomendado)
python3 -m http.server 8080
# Navegar a: http://localhost:8080/Auditoría_appv2_patched.html
```

**Compatibilidad:** Chrome 90+ · Firefox 88+ · Safari 14+ · Edge 90+

---

## Análisis IA

La función de análisis IA genera un diagnóstico ejecutivo del estado de cumplimiento, incluyendo brechas críticas por framework, top 5 acciones recomendadas y evaluación del riesgo regulatorio CNBV/Banxico.

Requiere una API Key de Anthropic:(Totalmente opcional para quien tenga servicio contratado con Anthropic)
1. Obtén tu key en [console.anthropic.com](https://console.anthropic.com)
2. Ingresala en el modal 🤖 de la aplicación
3. Se guarda localmente — no se comparte con nadie

> ⚠️ El análisis envía estadísticas agregadas de tus controles a la API de Anthropic. No incluyas información clasificada en campos de notas antes de ejecutar el análisis.

---

## Seguridad del código

El proyecto incluye análisis SAST post-construcción con 10 hallazgos identificados y corregidos:

| Severidad | Hallazgos | Estado |
|-----------|-----------|--------|
| 🔴 Crítica | 1 — XSS Stored via `innerHTML` | ✅ Corregido |
| 🟠 Alta | 3 — Attribute injection, SRI, API Key storage | ✅ Corregido / Mitigado |
| 🟡 Media | 4 — CSP, CSV injection, data disclosure, error handling | ✅ Corregido |
| 🔵 Baja | 1 — Input validation | ✅ Corregido |
| ⚪ Informativa | 1 — localStorage sin cifrado | 📋 Roadmap |

Ver reporte completo: [`SAST_Report_Auditoría_appv2.md`](./SAST_Report_Auditoría_appv2.md)

### Controles de seguridad implementados

- **XSS Prevention** — función `esc()` con HTML entity encoding en todos los puntos de salida
- **Event Delegation** — eliminación de `onclick` con datos de usuario interpolados
- **Content Security Policy** — meta CSP con orígenes explícitamente permitidos
- **CSV Formula Injection** — función `csvSafe()` con prefijo en caracteres de fórmula
- **Sin dependencias CDN para gráficas** — Canvas 2D API nativa (zero external scripts para rendering)
- **Input Validation** — `maxlength` en todos los campos del formulario

---

## Estructura del repositorio

```
secarch/
├── Auditoría_appv2.html              # Versión funcional original
├── Auditoría_apauditoría_appv2_patched.html      # Versión con parches de seguridad aplicados ← usar esta
├── SAST_Report__v2.md   # Reporte de análisis estático de seguridad
├── spec.md                      # Especificación técnica del proyecto
└── README.md                    # Este documento
```

---

## Stack tecnológico

| Componente | Tecnología | Notas |
|------------|------------|-------|
| Estilos | Tailwind CSS (CDN) | Utilidades CSS |
| Tipografía | Inter + JetBrains Mono | Google Fonts |
| Gráficas | Canvas 2D API | Nativa — sin librerías externas |
| IA | Anthropic Claude API | Opcional — requiere API Key |
| Persistencia | Web Storage API (`localStorage`) | Nativa del navegador |
| Runtime | Vanilla JavaScript ES2020+ | Sin frameworks, sin build step |

---

## Roadmap

- [ ] Eliminar `unsafe-inline` del CSP (refactoring a archivos externos)
- [ ] Cifrado de `localStorage` con Web Crypto API
- [ ] Importación de controles desde CSV
- [ ] Animación de entrada en rings (Canvas `requestAnimationFrame`)
- [ ] Audit trail — historial de cambios por control
- [ ] Backend ligero + autenticación (modo multi-usuario)

---

## Contexto profesional

Este proyecto forma parte de un portafolio de herramientas construidas con IA aplicadas a la arquitectura de ciberseguridad en el sector financiero mexicano. Desarrollado por un Arquitecto de Ciberseguridad con experiencia como CISO, como demostración de la integración práctica de IA generativa en procesos de seguridad reales.

---

*Auditoría_app v2.0 · Junio 2025 · Licencia MIT*
