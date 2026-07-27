# Auditoría técnica del sistema actual
*Fecha: 2026-07-27 · Alcance: index.html, dashboard.html, chart.umd.js, README.md, tabla `public.obs_sesiones` y funciones edge en Supabase (proyecto TVContigo).*

## 1. Arquitectura actual (cómo funciona hoy)

- **index.html** — SPA vanilla JS autocontenida. Configura bandas de edad (6–7, 8–9, 10–11, 12–13, 14) con módulos por banda (caras, situaciones, escala tipo SCAS, atención, globos, memorama, sociograma, red de apoyo) y un registro observacional del facilitador (7 ítems, 0–3). Calcula 5 índices 0–100 mediante promedios ponderados documentados en `score()` y hace `INSERT` directo a PostgREST (`/rest/v1/obs_sesiones`) con la llave *publishable*.
- **dashboard.html** — SPA vanilla + Chart.js local. Leía la misma tabla por PostgREST y calcula KPIs, comparación por banda, evolución por corte/semana, tabla por participante y exportación CSV.
- **Base de datos** — una sola tabla desnormalizada `public.obs_sesiones` (identidad + resultados + respuestas JSONB en la misma fila), dentro de un proyecto Supabase **compartido con otra aplicación** (TVContigo: quinielas, ads, push...).
- **Distribución** — GitHub Pages (estático) y una función edge (`juego`) que sirve el HTML.

## 2. Hallazgos (priorizados)

| # | Severidad | Hallazgo | Evidencia | Estado |
|---|---|---|---|---|
| 1 | **CRÍTICA** | Datos de menores legibles públicamente: política RLS `obs_select_publico` con `USING (true)` para `anon` — cualquiera con la URL del proyecto podía descargar nombres/edades/resultados. | `pg_policy` + linter Supabase (`rls_policy_always_true`) | **CORREGIDO** en Fase 1a: política eliminada; `set role anon; select count(*)` = 0. Lectura ahora solo vía función `datos` (clave + service_role). |
| 2 | **ALTA** | XSS almacenado: `dashboard.html` inyectaba `nino_nombre`, `actividad`, etc. con `insertAdjacentHTML` sin escape; combinado con INSERT anónimo, un tercero podía inyectar `<img onerror=...>` y ejecutar JS en el navegador del equipo. | Revisión de código; probado con payload malicioso | **CORREGIDO** en Fase 1b: `esc()` en todos los campos de origen externo; test automatizado confirma que el payload se renderiza como texto. |
| 3 | **ALTA** | Pérdida de datos sin conexión: si fallaba el `INSERT`, la sesión solo sobrevivía si el usuario descargaba el respaldo manual. | Revisión de código | **CORREGIDO** en Fase 1b: cola local (`localStorage`) con reintento automático al evento `online`, indicador de pendientes y botón de sincronización. |
| 4 | **ALTA** | Duplicidad de registros: reintentos generaban filas nuevas (id lo ponía el servidor). | Revisión de código | **CORREGIDO** en Fase 1b: UUID generado en cliente por sesión; el reintento produce `409` (conflicto de PK) tratado como éxito → idempotente. |
| 5 | **ALTA** | `INSERT` anónimo sin restricción (`WITH CHECK (true)`): cualquiera puede insertar filas (spam/veneno de datos), aunque ya no pueda leerlas. | Linter Supabase | **PENDIENTE (aceptado temporalmente)** para no romper la operación en campo. Se elimina en Fase 2 con login de facilitador (Supabase Auth) o proxy de escritura con clave. |
| 6 | MEDIA | Identidad y resultados en la misma tabla: impide anonimización y roles. | Diseño | **MITIGADO**: nuevo esquema `clinico` separa `patients` de `assessments/scores`; migración disponible (`clinico.migrar_obs_sesiones()`). |
| 7 | MEDIA | Proyecto Supabase compartido con otra app (decenas de funciones/tablas ajenas, varios lints de seguridad de esa app). | Linter | **MITIGADO** con esquema `clinico` aislado y RLS cerrado; recomendación: proyecto dedicado en producción. |
| 8 | MEDIA | Nombres de participantes escritos en el código (`NINOS=[...]`) y nombre completo del menor en la base. | Código | **PENDIENTE Fase 2**: catálogo de participantes servido por API con permisos; en campo usar solo folio + nombre preferido. |
| 9 | MEDIA | Sin autenticación ni roles ni auditoría. | Diseño | **PENDIENTE Fases 2–3** (base ya creada: `profiles`, `user_roles`, `audit_logs`). |
| 10 | BAJA | Ítems y fórmulas del instrumento incrustados en el código. | Código | **MITIGADO**: catálogo `instruments/instrument_versions/...` creado y sembrado con el instrumento actual v1.0; la app lo consumirá en Fase 3. |
| 11 | BAJA | Accesibilidad: sin `aria-labels` en opciones tipo botón; contraste correcto; TTS presente. | Revisión | **PENDIENTE Fase 3** (checklist WCAG AA). |
| 12 | BAJA | Rendimiento: adecuado (archivos <250 KB, Chart.js local, sin dependencias de CDN). | Medición | OK |

## 3. Cálculo de indicadores (verificado)
Promedios ponderados por fuente disponible (escala, registro observacional, situaciones, preocupaciones, sociograma), reescalados a 0–100, con `null` cuando no hay fuente (se muestra "—", nunca 0 fantasma). Niveles: <25 mínimo, <45 leve, <65 moderado, ≥65 marcado — **etiquetas operativas, no clínicas**. Los pesos están documentados en el código y deberán congelarse por versión de instrumento (`assessment_scores.rule_snapshot`) al migrar a catálogo.

## 4. Qué se conserva y qué se reconstruye
**Se conserva:** experiencia de aplicación móvil (flujo por bandas, TTS, SVG), motor de puntuación (como v1.0 del catálogo), dashboard (gráficas y CSV), Chart.js local, GitHub Pages como canal de distribución de la fase estática.
**Se reconstruye por módulos (sin migración destructiva):** capa de datos (tabla única → esquema `clinico`), acceso (clave temporal → Supabase Auth + roles + RLS por política), captura de identidad (código → expediente), reportes (CSV → PDF clínicos), y el shell de aplicación (vanilla → Next.js + TypeScript cuando entren los módulos clínicos).
