# Ansiedad social · Curso de verano — Instrumento y Dashboard

Piloto de observación de ansiedad social (6–14 años) del Deportivo SNTSS, Sección VI Yucatán.
Uso académico y de acompañamiento. **No es un instrumento de diagnóstico.**

## Contenido

| Archivo | Qué es |
|---|---|
| `index.html` | El instrumento (juegos por edad) que se aplica desde el teléfono. Guarda cada sesión en la base de datos. |
| `dashboard.html` | El dashboard de reportes y métricas. Lee la base y muestra KPIs, gráficas por grupo de edad, evolución por corte/semana, tabla por participante y exportación a CSV. |
| `chart.umd.js` | Librería de gráficas (Chart.js) local, para que el dashboard funcione sin depender de internet externo. |

## Cómo publicarlo en GitHub Pages (5 minutos)

1. Crea un repositorio **público** en github.com (por ejemplo `curso-verano`).
2. Sube estos 4 archivos tal cual (Add file → Upload files → Commit).
3. Ve a **Settings → Pages** → en *Branch* elige `main` y carpeta `/ (root)` → **Save**.
4. Espera ~1 minuto. Tu sitio quedará en:
   - Instrumento: `https://TU-USUARIO.github.io/curso-verano/`
   - Dashboard: `https://TU-USUARIO.github.io/curso-verano/dashboard.html`

> Importante: abre siempre la URL de **Pages**, no el enlace "Raw" del archivo (ese muestra el código).

## Uso en campo

1. Abre el instrumento en el teléfono, elige participante, edad, **corte** (Inicio/Medio/Cierre), semana y la **actividad** del día.
2. El niño juega las actividades de su grupo de edad; al final tú llenas el **registro observacional** (7 indicadores, 0–3) y guardas.
3. Abre el dashboard en cualquier dispositivo y presiona **Actualizar** para ver los datos al momento. El botón **CSV** descarga el reporte para Excel/SPSS.

## Metodología (resumen)

- **Foco:** ansiedad social (subescala tipo SCAS de fobia social, ítems 0–3: Nunca / A veces / Muchas veces / Siempre).
- **6–7 años:** caras pictóricas tras la actividad + juego + registro del facilitador (el autorreporte aún no es confiable).
- **8–9:** escala con lectura en voz alta. **10–11:** escala + sociograma. **12–14:** autorreporte directo + sociograma + red de apoyo.
- **Cortes:** Inicio, Medio y Cierre, según el protocolo del estudio.
- Los índices (0–100) orientan el acompañamiento y el expediente; la interpretación se hace por grupo de edad y la valida un profesional.
- Para el estudio formal, sustituir los ítems orientativos por la versión oficial validada del SCAS (Hernández et al., 2010; scaswebsite.com).

## Ética y datos

- Aplicar con consentimiento informado de padres/tutores y autorización de la institución.
- El dashboard tiene activada por defecto la opción **"Ocultar nombres"**; en público usar solo códigos (N-01, N-02…).
- La base guarda datos mínimos (código, primer nombre, edad, índices y respuestas del juego).


## Seguridad (Fase 1 aplicada)

- La tabla del estudio **ya no es legible públicamente** (política de SELECT anónimo eliminada).
- El dashboard pide una **clave de acceso** y lee a través de una función protegida del servidor.
- El instrumento guarda con **UUID idempotente** (no se duplican sesiones al reintentar) y tiene **cola offline**: si no hay internet, la sesión queda en el teléfono y se sincroniza sola.
- El dashboard **escapa todo el contenido** proveniente de la base (prevención XSS).
- Existe un esquema clínico normalizado (`clinico`, 27 tablas con RLS cerrado) listo para las fases de autenticación y expediente. Ver `docs/AUDITORIA.md` y `docs/ARQUITECTURA.md`.

## Seguridad (Fase 2 aplicada)

- **Escritura anónima eliminada**: el instrumento ahora envía por la función `captura`, que exige una *clave de aplicación* y valida/sanea todo el payload en el servidor.
- **Roles y RLS por política** en el esquema clínico (admin, clínico, facilitador, capturista, supervisor) con pruebas por rol; el facilitador no puede ver pacientes ni historias.
- **Notas firmadas inmutables** (corrección solo por adenda) y **auditoría automática** de cambios.
- Usuario administrador real en Supabase Auth. Detalle completo y claves del equipo: `docs/FASE2.md`.
