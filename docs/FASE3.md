# Fase 3 · Núcleo clínico operativo (escritura real)
*Ejecutada: 2026-07-27*

El panel dejó de ser solo lectura: ahora **crea y edita** de verdad, con permisos por rol, validación en el servidor y auditoría automática.

## Backend (funciones `security definer` con control de rol)
Cada función valida el rol del llamante antes de escribir; expuestas en `public` para la API. Probadas **en producción** con el token real del admin (login → RPC → verificación), y luego se limpiaron los datos de prueba.

| Función | Qué hace | Rol requerido | Prueba |
|---|---|---|---|
| `pac_guardar(p)` | Alta/edición de paciente (+tutor opcional). Genera folio si va vacío. | admin, clínico, capturista | 200 · devolvió UUID de paciente |
| `consent_guardar(p)` | Registra consentimiento (tipo, versión, firmante, fecha). | admin, clínico, capturista | — |
| `historia_guardar(p)` | Upsert de historia clínica **modular** (secciones JSONB con `no_evaluado`). Guardado parcial. | admin, clínico | 200 · UUID de historia |
| `consulta_guardar(p)` | Crea consulta + nota (SOAP/DAP/libre); si `firmar=true`, la nota queda **firmada**. | admin, clínico | 200 · encounter+note, `firmada:true` |
| `nota_adenda(p)` | Adenda a una nota firmada (autor, fecha, motivo). | admin, clínico | 200 · UUID de adenda |
| `usuario_crear(p)` | Crea usuario en Auth + perfil + rol. | **solo admin** | 200 · el usuario nuevo **inició sesión** correctamente |
| `historia_de(pid)` | Lectura de la historia de un paciente. | admin, clínico | 200 |

Todo lo anterior quedó **registrado en auditoría** (`clinico.audit_logs`) automáticamente.

## Panel (`admin.html`)
- **Participantes:** botón “＋ Nuevo participante” con formulario (datos + tutor para menores) y **folio automático**. Cada fila abre la **ficha** del expediente.
- **Ficha del participante** con pestañas:
  - **Datos** (con tutores) y botón *Editar datos*.
  - **Historia clínica** modular: 15 apartados (motivo, problema actual, antecedentes, desarrollo, escolar, familiar, hábitos, trauma, factores protectores, estado mental, riesgos…) + impresión, objetivos, plan, pronóstico, observaciones y **diagnóstico profesional** (opcional, con la leyenda de que un puntaje nunca es diagnóstico). Cada apartado tiene **“No evaluado”** (deshabilita el campo) y **guardado parcial**.
  - **Consultas y notas:** nueva consulta con formato **SOAP / DAP / libre**, opción de **firmar** (queda bloqueada) y, para notas firmadas, **agregar adenda**. Lista el historial con estado *Firmada/Borrador* y sus adendas.
- **Usuarios y roles:** botón “＋ Nuevo usuario” (solo admin) → crea la cuenta y asigna rol.
- Validación en cliente + servidor, escape anti-XSS en todo el contenido, y el aviso **“no constituye diagnóstico”** fijo.

Prueba de UI (Playwright, todo verde): login admin → alta de participante con tutor → ficha → historia (carga previa, “no evaluado” deshabilita, guarda) → consulta **firmada** SOAP → adenda sobre nota firmada → alta de usuario facilitador. Sin errores de consola.

## Qué queda para las siguientes fases
- Consentimientos y documentos con **Storage privado + URLs firmadas**; módulo de **riesgos** con formulario dedicado; **agenda**; **reportes PDF** con vista previa.
- Migrar el instrumento para que escriba directo al esquema clínico (`assessments`) y jubilar `public.obs_sesiones`.
- MFA y cierre por inactividad.
- **Validación profesional pendiente:** umbrales de índices, licencia del SCAS oficial, y política de datos de menores (LFPDPPP) antes de escalar.
