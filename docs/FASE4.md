# Fase 4 · Módulo de riesgos y reportes PDF
*Ejecutada: 2026-07-27*

Dos capacidades nuevas en el expediente: **valoración de riesgos** (operativa, sin conclusiones automáticas) y **reportes en PDF** con vista previa y selección de secciones. Ambas respetan la regla del proyecto: un puntaje o una marca **nunca** es un diagnóstico; la valoración y la decisión final corresponden a un profesional.

## Backend (funciones `security definer`, probadas en producción)

| Función | Qué hace | Rol requerido | Prueba |
|---|---|---|---|
| `riesgo_guardar(p)` | Registra una valoración de riesgo por dominios (JSONB con nivel + nota), factores protectores, acciones, derivación, contacto de emergencia y seguimiento. | admin, clínico | 200 · devolvió UUID |
| `riesgos_de(pid)` | Lista las valoraciones de un paciente (orden reciente). | admin, clínico | 200 · lectura correcta |
| `reporte_registrar(p)` | Deja constancia en auditoría de cada reporte generado (paciente, tipo, secciones, fecha). | admin, clínico, supervisor | 200 · UUID + evento `REPORT` en `audit_logs` |

Probadas con el JWT real del administrador (login simulado → RPC → verificación); los datos de prueba se eliminaron (risk_assessments = 0, reports = 0). La valoración de riesgo queda **auditada** automáticamente.

## Panel (`admin.html`) — dos pestañas nuevas en la ficha

**Riesgos**
- Dominios: ideación/conducta suicida, autolesión no suicida, agresión a terceros, indicadores de abuso/maltrato, negligencia/desprotección y otros. Cada uno con nivel (*sin dato / bajo / moderado / alto*) y una observación.
- Campos de factores protectores, acciones tomadas, derivación, plan de seguimiento y casilla de "contacto de emergencia".
- **Alerta operativa** cuando algún dominio queda en nivel **ALTO**: recuerda activar el protocolo institucional y notificar al profesional responsable. Es un **aviso operativo, no un diagnóstico ni una instrucción clínica** — no se generan conclusiones automáticas.
- Historial de valoraciones con chips de color por nivel; las de nivel alto se resaltan.

**Reporte / PDF**
- Selección de secciones a incluir: resumen del expediente, historia clínica, consultas y notas, valoraciones de riesgo.
- **Vista previa** en pantalla con encabezado institucional, folio, fecha y la leyenda fija de que el documento es **orientativo y no constituye diagnóstico** (más aviso de confidencialidad por tratarse de datos de un menor).
- **Guardar como PDF** usa la impresión del navegador (destino "Guardar como PDF") con CSS de impresión dedicado; cada generación queda registrada vía `reporte_registrar` (auditoría).
- Las secciones clínicas solo están disponibles para personal clínico; el resumen básico, para roles con acceso al expediente.

## Pruebas de UI (Playwright, todo verde)
Login admin → ficha → **Riesgos**: carga previa con dominio alto (alerta y chip rojo visibles), alta de valoración (payload de dominios correcto, aviso de nivel alto, emergencia marcada). → **Reporte**: selección de las cuatro secciones, vista previa con disclaimer, historia (incluye "No evaluado"), consultas SOAP y riesgos; botón PDF dispara `window.print()` y registra `reporte_registrar` con las secciones. Sin errores de consola. No se rompió nada de las fases anteriores (suites de Fase 2 y 3 siguen verdes; escape anti-XSS intacto).

## Pendiente para las siguientes fases
- Consentimientos y documentos con **Storage privado + URLs firmadas**; **agenda/calendario**.
- Migrar el instrumento para escribir directo a `clinico.assessments` y jubilar `public.obs_sesiones`.
- **MFA** y cierre por inactividad.
- **Validación profesional pendiente:** umbrales de índices, licencia del SCAS oficial y política de datos de menores (LFPDPPP) antes de escalar el uso real.
