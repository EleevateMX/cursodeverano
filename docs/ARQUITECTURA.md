# Arquitectura objetivo y plan por fases

## Principio rector
Separación estricta de: datos clínicos · datos administrativos · instrumentos · resultados observacionales · estadística anonimizada. Ninguna puntuación se convierte automáticamente en diagnóstico; la advertencia "no constituye diagnóstico" se conserva en todas las pantallas y reportes.

## Arquitectura objetivo

```
┌───────────────────────────────────────────────────────────────┐
│  Frontend (Next.js 14 + TypeScript + Tailwind, PWA-ready)     │
│  ├─ Modo profesional (escritorio): expediente, historia,      │
│  │  consultas/notas, riesgos, agenda, reportes, admin         │
│  └─ Modo aplicación (móvil/tableta): instrumentos por edad,   │
│     cola offline, TTS, lenguaje por banda                     │
├───────────────────────────────────────────────────────────────┤
│  Reglas de negocio: Zod (validación cliente+servidor),        │
│  servicios por módulo, manejo central de errores              │
├───────────────────────────────────────────────────────────────┤
│  Supabase                                                     │
│  ├─ Auth (email+password, recuperación, bloqueo, MFA fase 4)  │
│  ├─ PostgreSQL · schema clinico (27 tablas, RLS por rol)      │
│  ├─ Storage privado + URLs firmadas (documentos)              │
│  └─ Edge Functions (lectura protegida, reportes, auditoría)   │
└───────────────────────────────────────────────────────────────┘
```

**Roles** (tabla `clinico.user_roles`): `admin`, `clinico`, `facilitador`, `capturista`, `supervisor` — políticas RLS por rol y por organización (menor privilegio; el facilitador nunca ve historia clínica ni diagnósticos; el supervisor solo agregados anonimizados; nada de `service_role` en frontend).

## Modelo de datos
Creado y aplicado en el schema `clinico` (migración `clinico_esquema_base`): organizations, locations, programs, profiles, user_roles, patients, patient_guardians, patient_contacts, consents, clinical_histories (secciones JSONB con opción **"no_evaluado"**), encounters, clinical_notes (SOAP/DAP/libre; **trigger que bloquea la edición de notas firmadas** → solo `note_addenda`), note_addenda, risk_assessments, appointments, instruments, instrument_versions, instrument_items, instrument_options, instrument_scoring_rules, assessments (con `rule_snapshot` para conservar la versión exacta aplicada), assessment_responses, assessment_scores, treatment_plans, treatment_goals, attachments, reports, audit_logs. UUID en todo, `created_at/updated_at` con trigger, borrado lógico (`deleted_at`), llaves foráneas e índices. **RLS activado y cerrado por defecto** (sin políticas hasta la fase de Auth: nadie entra por API).

## Plan por fases

| Fase | Contenido | Estado |
|---|---|---|
| **1 (esta entrega)** | Auditoría; cierre de lectura pública; proxy de lectura con clave; XSS; idempotencia; cola offline; esquema clínico + semillas + migración legada | ✅ Ejecutada |
| **2 · Identidad y acceso** | Supabase Auth (login, recuperación, bloqueo por intentos, cierre por inactividad), políticas RLS por rol/organización, eliminación del INSERT anónimo y de nombres en código, auditoría de accesos activa | Pendiente |
| **3 · Núcleo clínico (Next.js)** | Expediente + tutores + consentimientos; historia clínica modular con guardado parcial; consultas y notas con firma/adendas; módulo de riesgos (alertas operativas, sin conclusiones automáticas); instrumento consumido desde catálogo | Pendiente |
| **4 · Operación** | Agenda completa; documentos con Storage privado y URLs firmadas; reportes PDF con vista previa y selección de secciones; dashboard clínico individual; MFA | Pendiente |
| **5 · Institucional** | Dashboard institucional con filtros org/sede/programa, supresión de grupos pequeños (n<5), comparaciones longitudinales, exportación autorizada; PWA offline-first; pruebas E2E completas | Pendiente |

## Migración de datos legados
`select * from clinico.migrar_obs_sesiones();` (solo un administrador desde SQL; revocada a roles de API). Crea pacientes mínimos por folio y convierte cada sesión en `assessments` + `assessment_scores` con traza (`legacy_obs_id`), idempotente. `public.obs_sesiones` se conserva como tabla de captura hasta que la Fase 3 apunte el instrumento al nuevo esquema; entonces se congela como legado.

## Decisiones que requieren validación profesional
1. Los pesos/umbrales de los índices (mínimo/leve/moderado/marcado) son operativos; debe validarlos la responsable metodológica antes de cualquier uso más allá del piloto.
2. Sustituir los ítems orientativos por el SCAS oficial (licencia/permiso) para el estudio formal.
3. Política de retención y resguardo de datos de menores (aviso de privacidad y consentimientos) conforme a LFPDPPP antes de capturar más datos personales.
4. Este sistema no debe presentarse como expediente clínico certificado (NOM-024/regulación aplicable) sin revisión especializada.
