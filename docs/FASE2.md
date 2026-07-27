# Fase 2 · Identidad, roles y cierre de escritura anónima
*Ejecutada: 2026-07-27*

## Qué se implementó (todo verificado)

### 1. Usuario administrador real (Supabase Auth)
- Usuario administrador creado en `auth.users` (credenciales entregadas por canal privado; la contraseña inicial fue rotada tras el incidente del 2026-07-27).
- Perfil en `clinico.profiles` y roles **admin + clinico** en `clinico.user_roles`, vinculados a la organización de demostración (Deportivo SNTSS Secc. VI).
- Nota: el usuario fue creado por SQL; si el login fallara en la primera prueba de la app (Fase 3), se recrea en 1 minuto desde Supabase Studio → Authentication.

### 2. Políticas RLS por rol (menor privilegio)
Aplicadas sobre el esquema `clinico` para `authenticated` (anon: **cero acceso**):

| Recurso | admin | clinico | facilitador | capturista | supervisor |
|---|---|---|---|---|---|
| Organización/sedes/programas | RW | R | R | R | R |
| Pacientes, tutores, contactos, consentimientos | RW | RW | — (solo RPC mínima folio+nombre) | RW | — |
| Historia clínica, consultas, notas, riesgos, planes | RW | RW | — | — | — |
| Agenda | RW | RW | — | RW | — |
| Catálogo de instrumentos | RW | R | R | R | R |
| Aplicaciones (assessments) | RW | RW | inserta y ve **solo las suyas** | — | — |
| Estadística anonimizada (RPC, con supresión n<3) | ✔ | — | — | — | ✔ |
| Auditoría | R | — | — | — | — |

**Pruebas ejecutadas** simulando el JWT de cada rol (`request.jwt.claims` + `set role authenticated`):
admin ve pacientes (1); facilitador: tabla pacientes **0**, RPC mínima 1, historias **0**; capturista: pacientes 1, notas **0**, riesgos **0**; supervisor: pacientes **0**, estadística ejecuta; anon: sin permiso de ejecutar RPCs. ✅

### 3. Nota clínica firmada = inmutable
Probado en base: `UPDATE` sobre nota firmada → **rechazado por trigger**; corrección registrada como **adenda** con autor, fecha y motivo. ✅

### 4. Auditoría automática
Triggers `AFTER INSERT/UPDATE/DELETE` en pacientes, tutores, consentimientos, historias, consultas, notas, adendas, riesgos, citas y aplicaciones → `clinico.audit_logs` (actor = `auth.uid()`). Solo el admin puede leerla. ✅ (4 eventos registrados en la prueba.)

### 5. Cierre de la escritura anónima
- Política `obs_insert_publico` **eliminada**. Probado: `INSERT` como anon → `42501 row-level security`. ✅
- Nueva función edge **`captura`**: exige *clave de aplicación* (hash SHA-256 en servidor), **valida y sanea** todo el payload (UUID obligatorio, rangos 0–100, longitudes máximas, respuestas ≤60 KB) y escribe con `service_role`. Verificada en producción: clave correcta → 201; clave incorrecta → 401; reenvío del mismo id → 409 (idempotente). ✅
- `index.html` actualizado: campo "Clave de aplicación" (se guarda en el teléfono), envío vía `captura`, cola offline intacta, y distinción de errores: clave mala → aviso sin encolar; sin internet → cola y sincronización automática. Probado con Playwright (4 escenarios). ✅

## Claves del equipo
Las claves de aplicación, de dashboard y las credenciales de administrador se entregan **por canal privado** y NUNCA se publican en este repositorio. En el servidor solo viven sus hashes SHA-256.

> Incidente 2026-07-27: una versión anterior de este documento incluía las claves y fue publicada; **todas fueron rotadas de inmediato** (funciones redesplegadas con nuevos hashes y contraseña de admin regenerada), quedando inservibles las expuestas.

## Pendiente para Fase 3 (no simulado)
Aplicación Next.js con login real (las políticas ya están listas), expediente + historia clínica + notas con firma, instrumento leyendo el catálogo `clinico.instruments`, y jubilación de `public.obs_sesiones` como tabla de captura.
