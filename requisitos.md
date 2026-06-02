# Metodología SDD para Proyectos de IA con Vibe Coding

> **Propósito.** Esta metodología define cómo construir, evolucionar y mantener proyectos donde la implementación se hace con IA generativa (Claude Code, agentes, skills) sin perder control sobre los requisitos, la base de datos, el frontend ni la identidad del producto. Está diseñada para equipos pequeños que necesitan absorber cambios de última hora a alta velocidad, como si construyeran un MVP de forma continua, sin destruir lo ya construido.

---

## 0. Filosofía y principios fundamentales

1. **La especificación es la única fuente de verdad.** El código es un subproducto. Si el código y el spec discrepan, manda el spec.
2. **Actualizar el spec ANTES que el código.** Ningún cambio funcional toca código sin haber actualizado el requisito correspondiente.
3. **Cambios atómicos versionados, nunca destructivos.** Los requisitos no se reescriben en sitio: se versionan en archivos nuevos que apuntan al estado anterior. La historia es navegable.
4. **Los errores son aprendizaje institucional, no vergüenzas.** Cada error de la IA o del humano se documenta con causa raíz, prompt detonante y acción preventiva en el spec.
5. **La IA propone, el humano decide.** Los agentes detectan impacto y proponen planes; el humano aprueba en dos checkpoints (papel → implementación).
6. **Trazabilidad bidireccional.** Cada REQ sabe qué archivos de código implementa (manifest). Cada decisión técnica (ADR) sabe qué REQ la motivó.
7. **Diferenciar cambio de requisito vs corrección de error.** Un cambio de requisito modifica el contrato del producto y genera nueva versión del REQ. Una corrección de error restaura el comportamiento ya pactado y NO genera nueva versión del REQ — va al log de errores.

---

## 1. Estructura del proyecto

Todo proyecto que adopte esta metodología parte de la siguiente estructura. Las carpetas marcadas como **OBLIGATORIAS** deben existir antes de generar el primer requisito.

```
project-root/
├── docs/                              [OBLIGATORIO antes de requisitos]
│   ├── vision.md                      Qué problema resuelve + objetivos
│   ├── tech-stack.md                  Framework, BD, hosting, integraciones
│   ├── personas.md                    Usuarios objetivo + casos de uso
│   └── constraints.md                 Límites: compliance, presupuesto, deadlines
│
├── requirements/                      Requisitos versionados, organizados por dominio
│   ├── auth/
│   │   ├── REQ-AUTH-001-login-google/
│   │   │   ├── initial.md             Primer estado (creado con /new-req)
│   │   │   ├── change-001.md          Primer cambio (Delta Spec)
│   │   │   ├── change-002.md          Segundo cambio
│   │   │   ├── INDEX.md               Estado actual + historial cronológico
│   │   │   ├── ledger.md              Refactor Ledger detallado (este REQ)
│   │   │   ├── manifest.md            Archivos de código que implementan este REQ
│   │   │   └── errors/
│   │   │       ├── ERR-001-loop-redirect.md
│   │   │       └── ERR-002-token-no-expira.md
│   │   └── REQ-AUTH-002-mfa/
│   └── payments/
│       └── REQ-PAY-001-checkout/
│
├── database/
│   ├── schema.sql                     Fuente de verdad de la BD
│   ├── diagram.mmd                    Mermaid ER autogenerado (NO editar a mano)
│   └── migrations/                    Generadas por el framework (Prisma, Alembic, etc.)
│
├── brand/                             Identidad visual y de marca
│   ├── colors.md                      Paleta + uso semántico
│   ├── typography.md                  Familias, jerarquía, tamaños
│   ├── voice-tone.md                  Cómo escribe la marca
│   └── logos/                         SVG/PNG con guía de uso
│
├── decisions/                         ADRs (Architecture Decision Records)
│   ├── ADR-001-elegir-framework.md
│   └── ADR-002-estrategia-cache.md
│
├── ledger.md                          Refactor Ledger GLOBAL (resumen, una fila por cambio)
├── CLAUDE.md                          Instrucciones para Claude Code en este proyecto
└── .claude/
    ├── agents/                        Subagentes especializados (planner, reviewer, implementer)
    └── commands/                      Slash commands del proyecto
```

---

## 2. Convenciones

### 2.1 IDs de requisitos

- Formato: `REQ-<DOMINIO>-<NNN>`
- Ejemplos: `REQ-AUTH-001`, `REQ-PAY-014`, `REQ-ONBOARD-003`
- Dominio en MAYÚSCULAS, 3–8 letras, derivado de la carpeta del dominio.
- `NNN` es secuencial dentro del dominio (no global), 3 dígitos con padding.

### 2.2 IDs de errores

- Formato: `ERR-<NNN>` dentro de la carpeta `errors/` del REQ "principal" afectado.
- Numeración local al REQ principal (no global).
- Si afecta a varios REQs, vive en el principal y los demás referencian con `[[ERR-NNN del REQ-XXX]]`.

### 2.3 IDs de decisiones

- Formato: `ADR-<NNN>-<slug>` global del proyecto.

### 2.4 Estados del ciclo de vida

Cada REQ tiene un único estado en cada momento, declarado en el frontmatter de su `INDEX.md`:

| Estado | Significado |
|---|---|
| `draft` | En discusión o redacción. No se debe implementar. |
| `aprobado` | Aprobado por el owner. Listo para implementar. |
| `implementado` | Código en producción + tests pasando + manifest actualizado. |
| `deprecado` | Reemplazado o eliminado. NO borrar archivos: marcar y enlazar al sustituto. |

### 2.5 Naming dentro de la carpeta de un REQ

| Archivo | Rol |
|---|---|
| `initial.md` | Primer estado del requisito al ser creado |
| `change-NNN.md` | Cada cambio incremental usando formato Delta Spec (ver §3.2) |
| `INDEX.md` | Estado actual + tabla cronológica de cambios |
| `ledger.md` | Refactor Ledger detallado del REQ |
| `manifest.md` | Lista de archivos de código que implementan este REQ |
| `errors/ERR-NNN-<slug>.md` | Archivos de error con causa raíz, solución y prompt detonante |

### 2.6 Idioma

El idioma de la documentación generada (REQs, errores, ledger) se elige al ejecutar `/init-project`. Los **keywords técnicos** permanecen siempre en inglés: IDs, estados (`draft`/`aprobado`/`implementado`/`deprecado` permiten variantes ES/EN), nombres de slash commands, frontmatter.

---

## 3. Plantillas

### 3.1 Plantilla: `initial.md` (creación de un requisito)

```markdown
---
id: REQ-AUTH-001
title: Login con Google
status: draft
owner: <nombre o email>
created: 2026-06-01
tags: [autenticacion, oauth, google]
db_entities: [users, sessions, oauth_accounts]
ui_components: [LoginPage, OAuthButton, AuthCallback]
dependencies:
  needs: []                              # IDs de REQs que este requisito necesita
  needed_by: []                          # Se llena automáticamente cuando otro REQ depende de este
---

# REQ-AUTH-001 — Login con Google

## Contexto y motivación
<Por qué este requisito existe. Vínculo a docs/vision.md o docs/personas.md cuando aplique.>

## Comportamiento esperado
<Descripción funcional en lenguaje natural. Qué hace el sistema cuando el usuario interactúa.>

## Criterios de aceptación
- [ ] Given <precondición>, When <acción>, Then <resultado esperado>
- [ ] Given ..., When ..., Then ...

## Entidades de base de datos afectadas
- `users` — se crea o vincula un registro al iniciar sesión.
- `oauth_accounts` — almacena el `provider_id` de Google.
- `sessions` — se crea una sesión activa.

## Componentes de UI/Frontend afectados
- `LoginPage` — renderiza botón de Google.
- `OAuthButton` — componente compartido.
- `AuthCallback` — ruta `/auth/callback/google`.

## Dependencias con otros requisitos
- Necesita: ninguno.
- Necesitado por: (se completa cuando otros REQs lo referencien).

## Notas
<Anything relevante que no encaje arriba.>
```

### 3.2 Plantilla: `change-NNN.md` (Delta Spec — ADDED / MODIFIED / REMOVED)

Cada cambio se expresa como **delta** sobre el estado anterior. NO se reescribe el requisito completo: se enumera explícitamente qué se añade, qué se modifica y qué se elimina.

```markdown
---
id: REQ-AUTH-001
change: change-002
supersedes: change-001                   # Archivo al que este cambio reemplaza ("initial" si es el primero)
superseded_by: null                      # Se rellena cuando un change-NNN posterior lo supersede
status: aprobado                         # draft | aprobado | implementado | superseded
owner: <nombre>
created: 2026-06-15
reason: |
  Cliente pidió que la sesión expire en 30 minutos en lugar de 60
  por requerimiento de compliance.
---

# Change-002 — REQ-AUTH-001

## ADDED
- Nueva regla: tras 25 minutos de inactividad, el sistema muestra un aviso
  "Tu sesión expirará en 5 minutos. ¿Continuar?".
- Nueva entidad de UI: `SessionExpiryWarning`.

## MODIFIED
- **Duración de sesión**: pasa de 60 minutos a 30 minutos. (Antes: 60 min)
- **Criterio de aceptación 3**: ahora valida 30 min + aviso a los 25.

## REMOVED
- Se elimina el flag `extend_session_on_click` (ya no aplica).

## Impacto en BD
- Sin cambios de schema.
- `sessions.expires_at` ahora se calcula con TTL=30min.

## Impacto en UI
- `LoginPage`: sin cambios.
- `AuthCallback`: sin cambios.
- `SessionExpiryWarning`: nuevo componente.

## Impacto en otros REQs (detectado por agente planner)
- `REQ-AUTH-002-mfa`: la duración de sesión MFA debe alinearse → ver change-001 de REQ-AUTH-002.
```

**Lifecycle del archivo.** Cuando un `change-NNN.md` posterior lo supersede, ESTE archivo se actualiza **únicamente de dos formas** (única excepción a la regla de inmutabilidad de los changes):

1. **Frontmatter**: `status: superseded` y `superseded_by: change-NNN`.
2. **Banner** insertado al inicio del cuerpo (tras el frontmatter):
   ```
   > ⚠️ **ESTADO HISTÓRICO.** Superseded by [[change-NNN]]. El estado vigente del REQ vive en `INDEX.md`. NO usar como fuente de verdad actual.
   ```

El contenido ADDED/MODIFIED/REMOVED **nunca** se reescribe. El motivo: los changes son log de auditoría, no documentación viva.

### 3.3 Plantilla: `INDEX.md` (estado actual del REQ)

```markdown
---
id: REQ-AUTH-001
current_state: change-002                # Cuál de los .md refleja el estado vivo
status: implementado
last_updated: 2026-06-15
---

# REQ-AUTH-001 — Login con Google

## Estado actual
**Versión vigente:** `change-002.md`
**Estado:** `implementado`
**Última actualización:** 2026-06-15

## Estado consolidado actual
<Descripción COMPLETA del comportamiento vigente del requisito tras aplicar todos los cambios.
Esta sección es la ÚNICA fuente de verdad del estado actual del REQ. Los agentes la leen
para entender qué hace HOY el requisito SIN necesidad de leer `initial.md` ni los `change-NNN.md`
históricos.

Debe contener:
- Comportamiento esperado completo (lo que el sistema hace hoy).
- Criterios de aceptación vigentes (los que aplican tras todos los changes).
- Entidades de BD afectadas en su estado actual.
- Componentes UI afectados en su estado actual.
- Dependencias actuales (`needs` y `needed_by` vigentes).

Esta sección se REESCRIBE íntegra en cada `/change-req`. Es la única forma de evitar que los
agentes reconstruyan el estado leyendo el historial completo — lo que llenaría el contexto
y mezclaría estados ya superados con el vigente. Ver §11.>

## Historial de versiones

| Versión | Fecha | Tipo | Autor | Resumen |
|---|---|---|---|---|
| `initial.md` | 2026-06-01 | creación | sotero | Login con Google con sesión de 60 min |
| `change-001.md` | 2026-06-08 | MODIFIED | sotero | Añadido "recordar dispositivo" |
| `change-002.md` | 2026-06-15 | MODIFIED | sotero | Sesión a 30 min por compliance |

## Errores conocidos
- [[ERR-001]] — Loop infinito en redirect cuando cookies bloqueadas (resuelto).
- [[ERR-002]] — Token no expiraba en sesiones MFA (resuelto, reveló spec ambiguo → change-002).

## Errores relacionados en otros REQs
<Solo si este REQ es secundario en un error multi-REQ; ej.: "[[ERR-001 de REQ-PAY-001]]".>
```

### 3.4 Plantilla: `manifest.md` (trazabilidad a código)

```markdown
---
id: REQ-AUTH-001
last_synced: 2026-06-15
---

# Manifest — REQ-AUTH-001

## Archivos implementados

### Backend
- `src/auth/google_oauth.ts` — handler OAuth + intercambio de tokens
- `src/auth/session.ts` — creación y validación de sesión
- `src/db/users.ts` — upsert de usuario al iniciar sesión

### Frontend
- `src/pages/login.tsx` — pantalla de login
- `src/components/OAuthButton.tsx` — botón compartido
- `src/components/SessionExpiryWarning.tsx` — aviso de expiración (añadido en change-002)

### Tests
- `tests/auth/google_oauth.spec.ts`
- `tests/e2e/login-flow.spec.ts`

### Migraciones
- `database/migrations/0003_add_oauth_accounts.sql`
- `database/migrations/0007_session_ttl_30min.sql` (change-002)
```

### 3.5 Plantilla: `ERR-NNN-<slug>.md` (gestión de errores)

```markdown
---
id: ERR-001
req_principal: REQ-AUTH-001
reqs_afectados: [REQ-AUTH-001, REQ-AUTH-002]
categoria: alucinacion_ia               # alucinacion_ia | spec_ambiguo | regresion | bug_humano
estado: resuelto                        # detectado | en_investigacion | resuelto | aceptado
detectado: 2026-06-10
resuelto: 2026-06-11
detectado_por: sotero
resuelto_por: claude-code + sotero
---

# ERR-001 — Loop infinito en redirect cuando cookies bloqueadas

## Síntoma
Cuando el navegador del usuario tiene cookies de terceros bloqueadas, el callback
de Google entra en loop entre `/auth/callback/google` y `/login` indefinidamente.

## Causa raíz
La IA generó código que dependía de una cookie `oauth_state` para validar el
callback, pero no verificó si la cookie estaba disponible antes de intentar leerla.
Cuando la cookie no existía, redirigía a `/login` sin limpiar el state, y `/login`
detectaba el state en URL y reintentaba el callback.

## Solución aplicada
1. Validar existencia de cookie antes de leerla en `src/auth/google_oauth.ts:45`.
2. Si la cookie no está, mostrar mensaje claro al usuario y NO redirigir.
3. Limpiar parámetros de URL al llegar a `/login` por flujo de error.

Commit: `a1b2c3d`

## ¿Era un cambio de requisito o un error?
**Error.** El requisito (REQ-AUTH-001) implícitamente esperaba el camino feliz
con cookies habilitadas. La IA produjo código frágil ante un escenario válido
no contemplado. NO se versionó el REQ por esto, pero SÍ se actualizó el spec
(ver "Acción preventiva").

## Prompt original que disparó el error (si aplica)
```
"Implementa el flujo de OAuth con Google para REQ-AUTH-001. Usa cookies
para el state."
```
La instrucción no contemplaba el caso de cookies bloqueadas. La IA asumió
camino feliz.

## Acción preventiva en el spec
- En `REQ-AUTH-001/change-001.md` (ADDED): "El sistema DEBE manejar el caso de
  cookies bloqueadas mostrando un mensaje informativo, sin entrar en loop."
- En `docs/constraints.md`: añadida nota "Asumir siempre que cookies de terceros
  pueden estar bloqueadas; diseñar fallbacks explícitos."

## Lecciones para futuros prompts a la IA
- Al pedir flujos OAuth, especificar explícitamente el manejo de cookies
  bloqueadas y modo incógnito.
- Pedir tests de propiedades para flujos con dependencias del browser.
```

### 3.6 Plantilla: `ledger.md` por REQ (Refactor Ledger detallado)

```markdown
# Refactor Ledger — REQ-AUTH-001

| Fecha | Objetivo del cambio | Archivos planificados | Riesgos | Comando de verificación | Estado |
|---|---|---|---|---|---|
| 2026-06-08 | Añadir "recordar dispositivo" (change-001) | `src/auth/session.ts`, `src/components/RememberDevice.tsx`, migración 0005 | Tokens de larga vida sin rotación | `npm run test:auth && npm run e2e:login` | hecho |
| 2026-06-15 | Sesión a 30 min (change-002) | `src/auth/session.ts`, migración 0007, `SessionExpiryWarning.tsx` | Romper sesiones activas en producción | `npm run test:auth && pm2 logs en staging` | hecho |
| 2026-06-11 | Fix loop cookies bloqueadas (ERR-001) | `src/auth/google_oauth.ts`, `src/pages/login.tsx` | Regresión en flujo feliz | `npm run e2e:login -- --grep cookies` | hecho |
```

### 3.7 Plantilla: `ledger.md` GLOBAL del proyecto

```markdown
# Refactor Ledger Global

> Resumen de un renglón por cambio importante. Detalles completos en el ledger.md de cada REQ.

| Fecha | REQ | Tipo | Resumen | Link |
|---|---|---|---|---|
| 2026-06-01 | REQ-AUTH-001 | nuevo | Login con Google | [REQ-AUTH-001](requirements/auth/REQ-AUTH-001-login-google/ledger.md) |
| 2026-06-08 | REQ-AUTH-001 | change-001 | Añadido "recordar dispositivo" | [REQ-AUTH-001](requirements/auth/REQ-AUTH-001-login-google/ledger.md) |
| 2026-06-10 | REQ-AUTH-001 | error | ERR-001: loop cookies bloqueadas | [ERR-001](requirements/auth/REQ-AUTH-001-login-google/errors/ERR-001-loop-redirect.md) |
| 2026-06-15 | REQ-AUTH-001 | change-002 | Sesión a 30 min por compliance | [REQ-AUTH-001](requirements/auth/REQ-AUTH-001-login-google/ledger.md) |
```

### 3.8 Plantilla: ADR (formato MADR + link a requisitos)

```markdown
---
id: ADR-001
title: Usar Prisma como ORM
status: aceptado                         # propuesto | aceptado | superseded | deprecado
date: 2026-06-01
deciders: [sotero]
related_reqs: [REQ-AUTH-001, REQ-PAY-001]
---

# ADR-001 — Usar Prisma como ORM

## Contexto
Necesitamos un ORM que soporte migraciones, sea compatible con Next.js y permita
generar tipos automáticamente.

## Decisión
Adoptar Prisma como ORM principal.

## Consecuencias
**Positivas:**
- Migraciones declarativas.
- Tipos generados automáticamente.
- Buen soporte para PostgreSQL.

**Negativas:**
- Dependencia de un único proveedor.
- Curva de aprendizaje del schema.prisma.

## Alternativas consideradas
- **Drizzle**: más liviano pero ecosistema más joven.
- **TypeORM**: maduro pero ergonomía inferior.

## Requisitos afectados
- [[REQ-AUTH-001]] — define entidades users, sessions, oauth_accounts.
- [[REQ-PAY-001]] — define entidades orders, payments.
```

### 3.9 Plantillas para `docs/` (documentación obligatoria)

Antes de poder ejecutar `/new-req`, las siguientes plantillas deben estar rellenas. Esta es la base que alimenta a la IA cuando genera nuevos requisitos.

**`docs/vision.md`**
```markdown
# Visión del producto

## Problema que resuelve
<1–3 párrafos. Qué dolor del usuario atacamos.>

## Objetivos del producto
- Objetivo 1: <medible>
- Objetivo 2: <medible>

## Métricas de éxito
- <KPI 1>
- <KPI 2>

## Out of scope
- <Lo que explícitamente NO haremos en esta fase.>
```

**`docs/tech-stack.md`**
```markdown
# Stack técnico

| Capa | Tecnología | Justificación / ADR |
|---|---|---|
| Frontend | Next.js 14 | [[ADR-002]] |
| ORM | Prisma | [[ADR-001]] |
| BD | PostgreSQL 16 | [[ADR-003]] |
| Hosting | Vercel + Supabase | [[ADR-004]] |
| Sistema de migraciones | `prisma migrate dev` | — |

## Integraciones externas obligatorias
- <Stripe / Auth0 / SendGrid / ...>

## Restricciones técnicas no negociables
- <Ej: latencia p95 < 300ms en API>
```

**`docs/personas.md`**
```markdown
# Personas y casos de uso

## Persona 1: <Nombre arquetipo>
- **Rol:** <freelancer, admin, etc.>
- **Objetivo principal:** ...
- **Dolor actual:** ...
- **Casos de uso clave:**
  - <Caso 1>
  - <Caso 2>

## Persona 2: ...
```

**`docs/constraints.md`**
```markdown
# Restricciones

## Compliance
- <GDPR / HIPAA / SOC 2 / ninguno>

## Presupuesto
- <Límite de coste mensual de infraestructura>

## Deadlines
- <Fechas duras conocidas>

## Restricciones técnicas heredadas
- <Asumir cookies de terceros bloqueadas (lección de ERR-001)>
- <...>
```

---

## 4. Workflows

### 4.1 Bootstrap — `/init-project`

**Precondiciones:** carpeta vacía (greenfield) o repo existente sin esta metodología.

**Pasos:**
1. La IA verifica si existe `docs/`. Si no existe, la crea con plantillas vacías y **se detiene**: no se puede continuar sin que el usuario llene `vision.md`, `tech-stack.md`, `personas.md` y `constraints.md`.
2. Una vez `docs/` está poblada, ejecutar `/init-project` de nuevo. La IA:
   a. Pregunta idioma de la documentación (ES/EN/...).
   b. Lee `tech-stack.md` y detecta el sistema de migraciones según la tabla del Apéndice A. Si no encuentra match, pregunta al usuario ofreciendo opciones similares a Django.
   c. Crea la estructura: `requirements/`, `database/`, `brand/`, `decisions/`, `ledger.md`, `CLAUDE.md`, `.claude/agents/`, `.claude/commands/`.
   d. Genera el `CLAUDE.md` inicial con instrucciones específicas del proyecto.
   e. Configura el workflow de GitHub Actions para regenerar el diagrama ER (Apéndice B).
3. La IA entrevista al usuario para esbozar los primeros REQs candidatos a partir de `vision.md` y `personas.md`. Para cada uno propone `id`, `title`, `db_entities` y `ui_components` esperados. El usuario aprueba y se ejecuta `/new-req` por cada uno.

### 4.2 Crear un requisito — `/new-req`

**Setup de git (si el proyecto está versionado, antes de crear archivos):**
- Detectar git con `git rev-parse --git-dir 2>/dev/null`.
- Si hay git: verificar árbol limpio, estar en rama base (`main`), crear y cambiar a rama `req/<REQ-ID>-<slug>` (ver Apéndice C).
- Si NO hay git: saltar este setup completo.

**Pasos:**
1. La IA pregunta: dominio, título corto, descripción funcional, criterios de aceptación.
2. Genera `requirements/<dominio>/REQ-<DOMINIO>-<NNN>-<slug>/` con `initial.md` (estado `draft`), `INDEX.md`, `ledger.md`, `manifest.md` (vacío) y `errors/` (vacía).
3. Detecta entidades de BD candidatas (cruzando con `database/schema.sql`) y componentes UI candidatos (cruzando con código existente). Las propone al usuario para confirmación.
4. Si el REQ depende de otros, actualiza el `dependencies.needs` y, en los REQs apuntados, su `dependencies.needed_by`.
5. Añade fila en `ledger.md` global.
6. **(Si hay git)** Commit del REQ inicial: `feat(REQ-<ID>): crear requisito <slug>`. La rama queda en local para que el usuario haga `git push -u origin <branch>` y abra PR cuando esté listo. Tras mergear, ejecutar `/cleanup-branches` (§4.8).

### 4.3 Cambiar un requisito — `/change-req`

Este es el flujo crítico. Usa **tres agentes en serie con dos checkpoints humanos**.

**Setup de git (si el proyecto está versionado, antes de invocar al planner):**
- Detectar git con `git rev-parse --git-dir 2>/dev/null`.
- Si hay git: verificar árbol limpio + estar en rama base. **NO crear rama todavía** — el planner es read-only. La rama se crea en el paso 7 (cuando ya sabemos REQ principal + NNN del nuevo change).
- Si NO hay git: saltar todo el flujo de ramas (los pasos marcados "(Si hay git)" se omiten).

**Pasos:**

**Fase 1 — Planner**
1. El usuario describe el cambio en lenguaje natural ("ahora la sesión debe durar 30 minutos en vez de 60").
2. El agente **planner** lee `docs/`, escanea todos los `INDEX.md` y `manifest.md` de la carpeta `requirements/`, y construye un grafo de impacto. Identifica:
   - REQs directamente afectados.
   - REQs indirectos (vía `dependencies.needed_by`).
   - Entidades de BD que se tocan.
   - Migraciones de BD necesarias.
   - Archivos de código a modificar (de los manifests).
3. Emite el **plan**: lista de REQs a versionar (con borrador de Delta Spec ADDED/MODIFIED/REMOVED para cada uno), entidades BD afectadas, migración esperada, archivos UI/backend a tocar, riesgos.

**Fase 2 — Peer reviewer (sesión limpia, otro agente)**
4. El agente **reviewer** recibe el plan SIN contexto previo de la sesión del planner. Su rol es adversario: busca contradicciones, dependencias circulares, REQs implícitamente afectados que el planner pasó por alto, riesgos de regresión.
5. Emite un **veredicto**: `aprobar`, `aprobar con observaciones` o `rechazar con motivos`.

**CHECKPOINT 1 — humano (papel):**
6. El usuario revisa plan + observaciones del reviewer. Aprueba, ajusta o aborta.
7. Si aprueba: la IA ejecuta:
   - **(a, solo si hay git)** Crea la rama definitiva ahora que conoce REQ principal + NNN: `git checkout -b req/<REQ-ID>-change-<NNN>-<slug>` (multi-REQ: `req/<PRIMARY-REQ-ID>-change-<NNN>-multi-<slug>`).
   - (b) Crea los `change-NNN.md` en cada REQ afectado siguiendo el formato Delta Spec.
   - (c) **Reescribe la sección "Estado consolidado actual"** del `INDEX.md` con el nuevo comportamiento completo tras aplicar el delta (no solo actualiza el puntero `current_state`).
   - (d) Marca el `change-NNN.md` anterior como `status: superseded`, añade `superseded_by` en su frontmatter e inserta el banner ⚠️ al inicio de su cuerpo (ver §3.2 lifecycle).
   - (e) Añade fila en `ledger.md` global y en cada `ledger.md` de REQ.
   - **(f, solo si hay git)** Commit del cambio de papel: `git commit -m "docs(REQ-<ID> change-<NNN>): <título>"`.

**Fase 3 — Implementer (sesión limpia, tercer agente)**
8. El agente **implementer** recibe únicamente los `change-NNN.md` aprobados y los `manifest.md` de los REQs afectados. NO ve el plan original ni la discusión.
9. Implementa el cambio: modifica código, genera/aplica migración con el sistema correspondiente (ver Apéndice A), actualiza `manifest.md` con archivos nuevos/cambiados.
10. Corre los tests del Apéndice C (tests automáticos obligatorios + smoke test en dev server). Genera checklist humano por REQ tocado.

**CHECKPOINT 2 — humano (implementación):**
11. El usuario revisa: tests verdes, smoke test ok, checklist humano. Aprueba el commit o pide ajustes (el implementer itera; el `change-NNN.md` NO cambia salvo que se descubra una ambigüedad → en ese caso vuelve a fase 1).
12. Una vez aprobado:
    - El status del REQ pasa a `implementado` en `INDEX.md`.
    - El status del `change-NNN.md` pasa a `implementado`.
    - **(Si hay git)** Commit del código sobre la misma rama: `git commit -m "feat(REQ-<ID> change-<NNN>): <título>"`. La rama queda en local con **dos commits** (papel + código).
    - **(Si NO hay git)** Solo se actualizan archivos en filesystem; no hay commit.
    - Próximo paso del usuario (si hay git): `git push -u origin <branch>`, abrir PR, mergear, y tras mergear ejecutar `/cleanup-branches` (§4.8).

### 4.4 Auditoría — `/audit`

Detecta divergencia entre código y requisitos (spec drift). **Crítico:** distingue entre cambio de requisito (no documentado) y corrección de error (no requiere versionar).

**Pasos:**
1. Escanea todos los archivos en `src/` (o carpetas de código según `tech-stack.md`).
2. Cruza con `manifest.md` de cada REQ. Archivos no referenciados son candidatos a drift.
3. Lee `git log` desde la última `/audit`. Para cada archivo modificado no en manifest:
   - Si el commit message contiene `fix:`, `bug:`, `hotfix:` → clasifica como **corrección de error candidato**. Pregunta al usuario si crear un `ERR-NNN.md` en el REQ asociado.
   - Si el commit message contiene `feat:`, `refactor:`, o no es clasificable → clasifica como **cambio de requisito candidato**. Pregunta al usuario si crear un `change-NNN.md` retroactivo o un nuevo REQ.
   - Si es ambiguo, pregunta al usuario.
4. Genera reporte: archivos huérfanos, REQs sin código (status `implementado` pero manifest vacío), inconsistencias entre `dependencies` de REQs.

### 4.5 Rollback — `/rollback REQ-XXX`

**Pasos:**
1. La IA muestra el `INDEX.md` del REQ: a qué versión anterior se puede revertir.
2. El usuario elige `version_objetivo` (ej. `change-001` para revertir `change-002`).
3. La IA:
   a. Identifica los commits asociados al cambio (cruzando `ledger.md` del REQ con `git log`).
   b. Genera el comando de migración `down` correspondiente al sistema de migraciones del proyecto (ver Apéndice A).
   c. Muestra el plan: `git revert <commit>` + comando de migración inversa.
4. **CHECKPOINT humano:** el usuario aprueba el rollback.
5. Tras ejecutar:
   - **(Si hay git)** Crear rama de rollback: `git checkout -b rollback/<REQ-ID>-to-<version-objetivo>` (ver Apéndice C).
   - `git revert <commits>` en orden inverso a su creación.
   - Ejecutar migración inversa.
   - El `INDEX.md` del REQ se actualiza apuntando a la versión objetivo como `current_state`. Se reescribe el "Estado consolidado actual" reflejando el estado de la versión objetivo. Se añade nota explicando el rollback.
   - El `change-NNN.md` revertido NO se borra (queda como historia). Se le añade un banner ⚠️ REVERTIDO al inicio.
   - Se añade fila tipo `rollback` en `ledger.md` del REQ y global.
   - **(Si hay git)** Commit: `git commit -m "revert(REQ-<ID>): rollback de change-<NNN> a <objetivo>"`. Próximo paso del usuario: push + PR de emergencia. Tras mergear: `/cleanup-branches`.
   - Si el rollback se hizo por bug en producción, se documenta como `ERR-NNN.md` (categoría suele ser `regresion`).

### 4.6 Regenerar diagrama BD — `/sync-schema`

Equivalente local al GitHub Action: parsea `database/schema.sql` y regenera `database/diagram.mmd`. Útil cuando se quiere ver el diagrama sin esperar al CI.

### 4.7 Estado del proyecto — `/status`

Genera un reporte:
- Total de REQs por estado (`draft`, `aprobado`, `implementado`, `deprecado`).
- REQs por dominio.
- Errores abiertos (`detectado` o `en_investigacion`) y sus REQs.
- Última actividad del ledger global.
- Lista de REQs sin manifest poblado (candidatos a drift).
- Lista de REQs con dependencias rotas (apuntan a REQs deprecados o inexistentes).

### 4.8 Limpiar ramas locales — `/cleanup-branches`

**Solo aplica en proyectos versionados con git.** Borra ramas locales cuyo PR ya está mergeado (o cuyos commits ya están en la rama base).

**Cuándo usarlo:**
- Tras mergear PRs generadas por `/new-req`, `/change-req` o `/rollback`, las ramas locales siguen en el repo. Este comando las limpia.
- Periódicamente (semanal, tras sprints) para mantener `git branch` aseado.

**Precondiciones:**
- Proyecto versionado con git.
- Idealmente, `gh` CLI instalado y autenticado (para detectar PRs mergeados con precisión). Si no, fallback a `git branch --merged` con limitaciones (no detecta squash-merge).

**Pasos:**
1. **Detectar `gh`** con `command -v gh`.
2. **Listar candidatas:**
   - Con `gh`: cruzar `gh pr list --state merged --json headRefName,number,mergedAt --limit 100` con ramas locales que sigan la convención SDD (`req/`, `err/`, `rollback/`).
   - Sin `gh`: fallback con `git branch --merged origin/main | grep -E '^\s*(req|err|rollback)/'`. **Advertir** que no detecta squash-merge.
3. **Presentar al usuario** agrupado por confianza:
   - 🟢 Seguras (PR mergeado + rama merged en git local) → borrar con `git branch -d`.
   - 🟡 PR mergeado pero rama no merged (squash-merge probable) → requiere `git branch -D` tras confirmación explícita.
4. **Si el usuario aprueba:**
   - Cambiar a `main` y `git pull`.
   - Borrar cada rama aprobada.
   - Omitir ramas con cambios sin commitear o commits no en el PR; reportar.
5. **Reportar resultado.**

**Casos especiales:**
- **Squash-merge:** GitHub/GitLab a menudo lo usan. `git branch -d` falla porque la rama no está "merged" según git local. La detección por `gh` lo resuelve. Requiere `-D` para borrado tras confirmación.
- **Rama con commits no en el PR:** advertir antes de borrar (alguien pushó cambios después del merge).
- **Rama con cambios sin commitear:** nunca borrar.
- **PR cerrado sin mergear:** no incluir como candidata.

**Restricciones:**
- Nunca `git branch -D` automático sin confirmación.
- Nunca tocar `main`, `master`, ramas de release.
- Nunca tocar el remoto; solo borrado local.
- Filtrar siempre por convención (`req/`, `err/`, `rollback/`).

---

## 5. Gestión de errores

Sección dedicada porque en vibe coding los errores son inevitables y, bien gestionados, son la fuente más rica de mejora del spec.

### 5.1 Distinción crítica: cambio de requisito vs corrección de error

| | Cambio de requisito | Corrección de error |
|---|---|---|
| **Causa** | El contrato del producto cambia. El comportamiento esperado es distinto. | El comportamiento real diverge del comportamiento ya pactado. |
| **Genera** | Nuevo `change-NNN.md` en el REQ + actualización de `INDEX.md` | Nuevo `ERR-NNN.md` en `errors/` del REQ |
| **Actualiza spec?** | Sí, es la razón del cambio | A veces (si reveló ambigüedad). Si sí, también se crea un `change-NNN.md`. |
| **Quién lo decide** | Owner del REQ (en checkpoint 1 de `/change-req`) | Cualquier dev al detectar el bug |

**Regla práctica:** si la pregunta "¿esto rompe la promesa que le hicimos al usuario?" se responde **sí, pero queremos cambiar la promesa** → cambio de requisito. Si se responde **no, la promesa sigue válida, esto es código incorrecto** → error.

### 5.2 Categorías de error

Cada `ERR-NNN.md` declara su `categoria` en el frontmatter:

- **`alucinacion_ia`** — La IA inventó una API, librería, comportamiento o asumió algo no especificado. **Signal:** revisar prompts y mejorar contexto en `CLAUDE.md`.
- **`spec_ambiguo`** — El requisito permitía múltiples interpretaciones razonables y la IA escogió una incorrecta. **Signal:** actualizar el `change-NNN.md` con un ADDED que cierre la ambigüedad.
- **`regresion`** — Un cambio rompió un comportamiento de otro REQ. **Signal:** el agente planner falló en detectar el impacto; revisar grafo de dependencias.
- **`bug_humano`** — Código modificado a mano fuera del flujo `/change-req` introdujo el bug. **Signal:** ejecutar `/audit` más seguido.

### 5.3 Errores que afectan a múltiples REQs

Cuando un error toca varios REQs, sigue esta regla:

1. **Identificar REQ principal**: el más afectado, o el primero donde se detectó.
2. El archivo `ERR-NNN.md` vive **en la carpeta del REQ principal**: `requirements/<dominio>/REQ-<PRINCIPAL>/errors/ERR-NNN-<slug>.md`.
3. En el frontmatter del error, `reqs_afectados` lista todos los REQs tocados.
4. En el `INDEX.md` de cada REQ secundario afectado, en la sección "Errores relacionados en otros REQs", se añade una línea referenciando: `[[ERR-NNN del REQ-PRINCIPAL]]`.

Esto evita duplicación de contenido manteniendo trazabilidad bidireccional.

### 5.4 El ciclo de vida de un error

1. **Detección** — Alguien (humano o agente) ve el bug. Estado: `detectado`.
2. **Triage** — Se clasifica categoría y se identifican REQs afectados. Si requiere investigación → `en_investigacion`.
3. **Causa raíz** — Se documenta en el `ERR-NNN.md`: síntoma, causa, prompt original (si fue IA).
4. **Solución** — Se implementa. Pasa a `resuelto`.
5. **Acción preventiva** — Si la categoría es `spec_ambiguo` o `alucinacion_ia`, el error debe generar un cambio en el spec (`change-NNN.md` que cierre la ambigüedad o instrucción en `CLAUDE.md` que evite la alucinación). Esto se documenta en la sección "Acción preventiva en el spec" del `ERR-NNN.md`.
6. **Aceptación** — En casos raros, un error se documenta y se acepta sin resolver (ej. limitación conocida de una librería externa). Estado: `aceptado`.

### 5.5 Por qué guardar el prompt original

En errores categoría `alucinacion_ia`, el campo "Prompt original" es crítico:

- Sirve como **caso de prueba** para evaluar futuras versiones del modelo o sesiones nuevas.
- Permite identificar **patrones de prompts frágiles** (ej. "siempre que pido OAuth sin especificar fallbacks, la IA falla").
- Alimenta `CLAUDE.md` con instrucciones que prevengan la recurrencia.

---

## 6. Validación de cambios

Todo cambio implementado debe pasar las tres capas siguientes antes del checkpoint 2 humano.

### 6.1 Tests automáticos obligatorios

Generados por el agente **implementer** durante la fase 3 de `/change-req`. Requisitos:

- Cada `change-NNN.md` debe traducirse en al menos un test que valide los criterios de aceptación ADDED o MODIFIED.
- Los tests existentes asociados al REQ (vistos en el `manifest.md`) deben seguir pasando.
- Tests de regresión para REQs `needed_by` deben seguir pasando.
- **No usar el mismo agente para implementar y testear.** Los tests los genera el implementer pero a partir de los criterios de aceptación del Delta Spec, no de su propio código (evita auto-validación circular — ver Investigacion.md §2).

### 6.2 Checklist humano

El implementer genera un checklist con los casos a probar a ojo, basado en los criterios de aceptación del Delta Spec. Ejemplo:

```
- [ ] Iniciar sesión con Google: el botón abre el popup de Google.
- [ ] Sesión expira a los 30 min de inactividad (verificable en logs).
- [ ] Aviso de expiración aparece a los 25 min.
- [ ] En navegador con cookies bloqueadas: muestra mensaje claro, no loop.
```

### 6.3 Smoke test en dev server

El implementer arranca el dev server localmente (o equivalente del stack) y verifica que:
- La aplicación inicia sin errores.
- Los endpoints/páginas tocadas por el cambio responden 200 / renderizan sin errores en consola.
- La migración aplica sin errores en una BD local de prueba.

Si alguna de estas tres capas falla, el cambio NO va a checkpoint 2; vuelve al implementer (o a fase 1 si la falla revela un problema en el plan).

---

## 7. Base de datos

### 7.1 `schema.sql` como fuente de verdad

Todo cambio de esquema empieza por `database/schema.sql`. El framework de migraciones del proyecto (ver Apéndice A) consume este archivo o el suyo equivalente (`schema.prisma`, modelos de Django, etc.) y genera la migración correspondiente.

### 7.2 Diagrama ER autogenerado

`database/diagram.mmd` se regenera automáticamente vía GitHub Action cada vez que `schema.sql` (o el archivo de schema del framework) cambia en `main`. **No editar a mano.**

Localmente, el comando `/sync-schema` hace lo mismo.

### 7.3 Migraciones

Según el framework detectado en `docs/tech-stack.md`. Ver Apéndice A.

---

## 8. Marca

Carpeta `brand/` con archivos:

- **`colors.md`** — Paleta con nombres semánticos, hex, casos de uso. Ejemplo:
  ```markdown
  ## Paleta primaria
  | Token | Hex | Uso |
  |---|---|---|
  | `primary` | #2563EB | Botones principales, links |
  | `primary-hover` | #1D4ED8 | Estado hover de `primary` |
  | `danger` | #DC2626 | Errores, acciones destructivas |
  ```

- **`typography.md`** — Familias (con fallbacks), jerarquía (h1–h6, body, caption), tamaños, line-heights.

- **`voice-tone.md`** — Cómo escribe la marca. Ejemplos de "decimos X / no decimos Y".

- **`logos/`** — SVG/PNG con guía de uso (cuándo usar versión clara/oscura, márgenes mínimos).

Cualquier cambio de marca sigue el flujo de cambio de requisito vía `/change-req` si afecta a componentes UI (entonces hay REQs cuyo `ui_components` se tocan).

---

## 9. Decisiones técnicas (ADRs)

Carpeta `decisions/` con formato MADR ligero (plantilla §3.8). Cada ADR enlaza los REQs que motivan o afecta. Cuando un ADR se supersede, no se borra: se marca `status: superseded` y se añade `superseded_by: ADR-NNN` en frontmatter.

Crear un ADR cuando:
- Se elige entre dos o más opciones técnicas no triviales (framework, ORM, estrategia de cache, modelo de auth).
- Se introduce una restricción que afectará a futuros REQs.
- Se descarta una tecnología candidata (documentar el "por qué no").

---

## 10. Configuración de Claude Code en el proyecto

### 10.1 `CLAUDE.md` mínimo

```markdown
# Instrucciones para Claude Code en <nombre del proyecto>

## Metodología
Este proyecto usa SDD con la metodología definida en `requisitos.md` (raíz).
Lee siempre esa metodología antes de proponer cambios estructurales.

## Reglas duras
1. Nunca modificar `requirements/**/initial.md`. Los `change-*.md` existentes solo pueden modificarse para marcar `status: superseded`, añadir `superseded_by` y banner ⚠️ al inicio (ver §3.2 lifecycle). El contenido ADDED/MODIFIED/REMOVED es inmutable. El `INDEX.md` SÍ se actualiza con cada cambio (sección "Estado consolidado actual").
2. Nunca modificar `database/diagram.mmd` (autogenerado).
3. Antes de tocar código, leer el `manifest.md` del REQ afectado.
4. Toda migración pasa por el sistema definido en `docs/tech-stack.md`.
5. Distinguir cambio de requisito (→ /change-req) vs corrección de error (→ ERR-NNN.md).

## Stack y herramientas
<Pegar resumen relevante de docs/tech-stack.md>

## Errores conocidos a tener en cuenta
<Lista de lecciones aprendidas extraídas de los errores categoría alucinacion_ia más
recurrentes. Actualizada periódicamente.>
```

### 10.2 Subagentes en `.claude/agents/`

- **`planner.md`** — Agente que ejecuta la fase 1 de `/change-req`.
- **`reviewer.md`** — Agente que ejecuta la fase 2 (sesión limpia, rol adversario).
- **`implementer.md`** — Agente que ejecuta la fase 3 (sesión limpia, ve solo Delta Specs aprobados).
- **`auditor.md`** — Agente que ejecuta `/audit`.

### 10.3 Slash commands en `.claude/commands/`

- `init-project.md`
- `new-req.md`
- `change-req.md`
- `audit.md`
- `rollback.md`
- `sync-schema.md`
- `status.md`
- `compact-req.md` *(opcional, mantenimiento — ver §11.4)*
- `rebuild-index.md` *(opcional, recuperación — ver §11.5)*

Cada slash command es un archivo `.md` cuyo contenido es el prompt que se inyecta al ejecutar el comando, con instrucciones que coordinan los agentes.

---

## 11. Estrategia de carga de contexto

A medida que un proyecto acumula REQs, changes y errores, leer "todo" en cada operación llenaría rápidamente la ventana de contexto de los agentes y, peor aún, mezclaría estados históricos ya superados con el estado vigente. Esta sección define cómo los agentes cargan información de forma disciplinada.

### 11.1 Principios

1. **`INDEX.md` es la única fuente de verdad del estado actual.** Los agentes NUNCA reconstruyen el estado vigente leyendo `initial.md + change-001 + change-002 + …`. Leen el "Estado consolidado actual" del `INDEX.md` y punto.
2. **Los `change-NNN.md` son deltas inmutables de auditoría.** Se leen bajo demanda (rollback, investigación forense, audit profundo). NUNCA en el flujo normal de un cambio nuevo.
3. **Los changes superados se marcan explícitamente** (ver §3.2 lifecycle: `status: superseded`, `superseded_by` y banner ⚠️) para que ni humanos ni agentes los confundan con verdad viva.
4. **Subagentes con contexto aislado.** La sesión principal nunca acumula los escaneos completos: los subagentes hacen el trabajo y devuelven al main session solo el resultado estructurado y compacto.

### 11.2 Tiers de carga

| Tier | Qué carga | Cuándo |
|---|---|---|
| **0** | `ledger.md` global (1 línea por cambio) + `docs/` completos | Siempre — punto de partida de cualquier operación |
| **1** | `INDEX.md` de TODOS los REQs (frontmatter + "Estado consolidado actual" + dependencias) | `/change-req` fase 1 (planner) y `/audit` |
| **2** | `manifest.md` + `INDEX.md` detallado SOLO de REQs marcados como impactados en tier 1 | Bajo demanda tras detectar impacto |
| **3** | `change-NNN.md` históricos, `ERR-NNN.md` antiguos, `initial.md` | Solo en `/rollback`, `/audit` profundo, o si el reviewer pide verificar un histórico |

**Regla práctica:** el planner se mueve en tier 0–1 y profundiza a tier 2 solo donde detectó impacto. El reviewer arranca con el grafo del planner (compacto, tier 0 de su contexto) y sube a tier 1–2 si necesita verificar. El implementer trabaja exclusivamente en tier 2 sobre los REQs afectados, sin ver el plan original ni la discusión.

### 11.3 Aislamiento por subagente

Cada fase de `/change-req` corre como subagente con su propio contexto. Esto es CRÍTICO para no saturar la sesión principal:

- **Planner** recibe: descripción del cambio en lenguaje natural + acceso a tiers 0–1. Devuelve: grafo de impacto estructurado, borradores de Delta Specs por REQ afectado, lista de archivos de código candidatos.
- **Reviewer** recibe SOLO el grafo de impacto del planner (NO los `INDEX.md` crudos). Si necesita verificar algo, sube a tier 1 por su cuenta. Devuelve: veredicto + observaciones.
- **Implementer** recibe SOLO los Delta Specs aprobados + los `manifest.md` de los REQs afectados (tier 2 acotado). Devuelve: diff de código + tests + checklist humano.

La sesión principal (donde interactúa el usuario) **nunca** acumula el tier 1 completo. Solo ve los productos compactos de cada subagente.

### 11.4 Compactación periódica (opcional)

Cuando un REQ acumula muchos changes históricos (recomendación: más de 10), ejecutar `/compact-req REQ-XXX`:

1. El agente lee todos los `change-NNN.md` históricos del REQ.
2. Genera `requirements/<dominio>/REQ-XXX/archive/consolidated-<fecha>.md` resumiendo qué cambió entre el `initial.md` y el último change marcado para compactación.
3. Mueve los `change-NNN.md` compactados a `requirements/<dominio>/REQ-XXX/archive/`.
4. Deja en la raíz del REQ solo: `initial.md` (intocable), changes desde el corte hacia adelante, `INDEX.md` actualizado con referencia al archive, `manifest.md`, `ledger.md`, `errors/`.

El `archive/` queda en git para auditoría histórica pero **no se carga en ningún tier por defecto**. Solo se consulta explícitamente cuando alguien pregunta "¿cómo era este REQ hace 6 meses?".

### 11.5 Si el `INDEX.md` se desactualiza

Si el `INDEX.md` deja de reflejar el estado vigente (humano olvidó actualizarlo tras un cambio, agente falló a mitad de operación, merge conflict mal resuelto), TODA la metodología se rompe: el planner verá un estado falso, el reviewer no detectará impactos reales, el implementer tocará código equivocado.

Mitigaciones:

- **Obligatoriedad en `/change-req`.** El paso 7 (§4.3) exige reescribir el "Estado consolidado actual" del `INDEX.md` ANTES del checkpoint 2. Es bloqueante.
- **`/audit` detecta inconsistencias.** Si `INDEX.md` dice `current_state: change-002` pero existe un `change-003` sin marcar como vigente, o un `change-002` con `status: implementado` pero sin reflejarse en el "Estado consolidado actual" → reporta.
- **Recuperación con `/rebuild-index REQ-XXX`.** El agente reconstruye el INDEX leyendo `initial.md` + todos los `change-NNN.md` en orden, consolidando. Es una operación costosa (lee tier 3 entero del REQ) pero salvavidas para recuperarse de un INDEX corrupto o desactualizado.

### 11.6 Resumen visual del flujo de carga

```
Usuario: "/change-req: la sesión debe durar 30 min"
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ Sesión principal                                            │
│ (solo ve productos compactos de subagentes)                 │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ invoca
┌─────────────────────────────────────────────────────────────┐
│ Planner (subagente, contexto aislado)                       │
│ Lee: tier 0 + tier 1 + tier 2 acotado                       │
│ Devuelve: grafo de impacto + Delta Specs borrador           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ pasa grafo a
┌─────────────────────────────────────────────────────────────┐
│ Reviewer (subagente, contexto limpio)                       │
│ Recibe: SOLO el grafo del planner                           │
│ Lee bajo demanda: tier 1 si necesita verificar              │
│ Devuelve: veredicto + observaciones                         │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ CHECKPOINT 1 humano → INDEX.md y changes actualizados
        ▼ invoca
┌─────────────────────────────────────────────────────────────┐
│ Implementer (subagente, contexto limpio)                    │
│ Recibe: SOLO Delta Specs aprobados + manifests              │
│ Lee: tier 2 acotado a REQs afectados                        │
│ Devuelve: diff + tests + checklist humano                   │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ CHECKPOINT 2 humano → commit
```

---

## Apéndice A — Sistemas de migración por framework

Cuando `/init-project` lee `docs/tech-stack.md`, mapea el framework al sistema de migraciones según esta tabla:

| Framework | Lenguaje | Sistema de migraciones |
|---|---|---|
| Next.js | TypeScript/JS | Prisma `prisma migrate dev` |
| Nuxt | TypeScript/JS | Prisma o Drizzle |
| SvelteKit | TypeScript/JS | Prisma o Drizzle |
| Django | Python | `makemigrations` / `migrate` |
| RedwoodJS | TypeScript/JS | Prisma (integrado) |
| FastAPI | Python | Alembic |
| Laravel | PHP | `artisan migrate` |

**Si el framework no aparece** en esta tabla, la IA debe preguntar al usuario qué sistema de migraciones se usará, ofreciendo opciones similares a Django (sistema declarativo con `makemigrations` autogeneradas a partir de modelos):
- Para Ruby on Rails → `rails db:migrate`
- Para Spring Boot → Flyway o Liquibase
- Para Phoenix (Elixir) → `mix ecto.migrate`
- Para otro → la IA propone Flyway como fallback genérico SQL-based.

---

## Apéndice B — GitHub Action: diagrama ER autogenerado

**Objetivo:** cada vez que `database/schema.sql` (o `schema.prisma` si usas Prisma) cambia en `main`, se regenera automáticamente `database/diagram.mmd` y se commitea de vuelta.

### Opción 1 — Stack con Prisma (Next.js, Nuxt, SvelteKit, RedwoodJS)

Aprovecha `prisma-erd-generator`, que ya genera Mermaid desde `schema.prisma`.

**Paso 1.** Añadir el generador en `prisma/schema.prisma`:
```prisma
generator erd {
  provider = "prisma-erd-generator"
  output   = "../database/diagram.mmd"
  theme    = "default"
}
```

**Paso 2.** Crear `.github/workflows/sync-schema.yml`:
```yaml
name: Regenerate ER diagram

on:
  push:
    branches: [main]
    paths:
      - 'prisma/schema.prisma'

jobs:
  regenerate:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx prisma generate
      - name: Commit diagram if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          if [ -n "$(git status --porcelain database/diagram.mmd)" ]; then
            git add database/diagram.mmd
            git commit -m "chore(db): regenerate ER diagram [skip ci]"
            git push
          fi
```

### Opción 2 — Stack con SQL plano (FastAPI/Alembic, Laravel, raw SQL)

Usa un script Python que parsea `schema.sql` y emite Mermaid. Ejemplo mínimo:

**`scripts/sql_to_mermaid.py`:**
```python
import re
from pathlib import Path

sql = Path('database/schema.sql').read_text()
tables = re.findall(r'CREATE TABLE\s+(\w+)\s*\((.*?)\);', sql, re.DOTALL | re.IGNORECASE)
fks = []
out = ['erDiagram']

for name, body in tables:
    columns = []
    for line in body.split(','):
        line = line.strip()
        if not line or line.upper().startswith(('PRIMARY KEY', 'CONSTRAINT', 'UNIQUE', 'INDEX', 'KEY')):
            continue
        m = re.match(r'(\w+)\s+([\w()]+)', line)
        if m:
            col, typ = m.groups()
            pk = ' PK' if 'PRIMARY KEY' in line.upper() else ''
            columns.append(f'    {typ} {col}{pk}')
        fk = re.search(r'REFERENCES\s+(\w+)', line, re.IGNORECASE)
        if fk:
            fks.append((name, fk.group(1)))
    out.append(f'  {name} {{')
    out.extend(columns)
    out.append('  }')

for a, b in fks:
    out.append(f'  {a} ||--o{{ {b} : "FK"')

Path('database/diagram.mmd').write_text('\n'.join(out))
print('diagram.mmd regenerated')
```

**`.github/workflows/sync-schema.yml`:**
```yaml
name: Regenerate ER diagram

on:
  push:
    branches: [main]
    paths:
      - 'database/schema.sql'

jobs:
  regenerate:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: python scripts/sql_to_mermaid.py
      - name: Commit diagram if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          if [ -n "$(git status --porcelain database/diagram.mmd)" ]; then
            git add database/diagram.mmd
            git commit -m "chore(db): regenerate ER diagram [skip ci]"
            git push
          fi
```

### Cómo visualizar `diagram.mmd`

- **VSCode**: instalar la extensión "Markdown Preview Mermaid Support". Abrir el archivo o embeber en cualquier `.md` con bloque ```` ```mermaid ````.
- **GitHub**: GitHub renderiza Mermaid nativamente en archivos `.md`. Para ver `diagram.mmd` directamente, embeberlo en `database/README.md`:
  ````markdown
  # Esquema de BD
  ```mermaid
  <!-- se actualiza copiando el contenido de diagram.mmd cada commit -->
  ```
  ````
  O bien, configurar el script para que sobrescriba `database/README.md` con el bloque ```` ```mermaid ```` ya incluido (más cómodo para visualizar en GitHub web).

### Notas importantes

- El flag `[skip ci]` en el mensaje del commit evita loops infinitos de CI.
- Si quieres bloquear el merge cuando el diagrama está desactualizado en una PR (en lugar de commitear automáticamente), cambia el job para que `git diff --exit-code database/diagram.mmd` falle y obligue al dev a correr `/sync-schema` local antes de pushear.

---

## Apéndice C — Convenciones de naming y commits

### Branches

| Patrón | Cuándo usar | Quién la crea |
|---|---|---|
| `req/<REQ-ID>-<slug>` | Creación de un REQ nuevo. Ej: `req/REQ-AUTH-001-login-google` | `/new-req` (§4.2) |
| `req/<REQ-ID>-change-<NNN>-<slug>` | Cambio sobre un REQ existente. Ej: `req/REQ-AUTH-001-change-002-session-30min` | `/change-req` single REQ (§4.3) |
| `req/<PRIMARY-REQ-ID>-change-<NNN>-multi-<slug>` | Cambio que afecta a varios REQs. Ej: `req/REQ-AUTH-001-change-002-multi-session-policy` | `/change-req` multi-REQ (§4.3) |
| `err/<ERR-ID>-<slug>` | Fix de un error documentado. Ej: `err/ERR-001-loop-redirect` | Manual (tras clasificación) |
| `rollback/<REQ-ID>-to-<target>` | Reversión de un change. Ej: `rollback/REQ-AUTH-001-to-change-001` | `/rollback` (§4.5) |
| `chore/<slug>` | Tareas de mantenimiento. Ej: `chore/sync-schema` | Manual |

El slug del REQ o ERR en el nombre de branch facilita correlacionar branch ↔ spec. El slug descriptivo corto (3-5 palabras, kebab-case) facilita identificar el cambio sin abrir el PR.

### Lifecycle de una branch

1. **Creación** — La crea el workflow correspondiente al detectar git en el proyecto. Si el proyecto NO está versionado con git, se salta este paso completamente.
2. **Commits** — Los workflows commitean en la rama siguiendo las convenciones de commit (sección siguiente). Para `/change-req`, son dos commits por flujo: uno tras checkpoint 1 (papel, `docs:`) y otro tras checkpoint 2 (código, `feat:` o `fix:`).
3. **Push y PR** — El usuario hace `git push -u origin <branch>` y abre PR en GitHub/GitLab.
4. **Review y merge** — En la plataforma. Al mergear, los cambios entran a `main`.
5. **Limpieza local** — Tras mergear, ejecutar `/cleanup-branches` (§4.8) para borrar la rama local. La rama remota la limpia GitHub/GitLab automáticamente si está configurado.

### Si el proyecto NO está versionado con git

- Todo el flujo de ramas se salta. Las modificaciones se aplican directamente al filesystem.
- Los comandos detectan automáticamente con `git rev-parse --git-dir` y se adaptan.
- No se hacen commits. El usuario decide cuándo (y si) versionar el proyecto.

### Commits

- **Commits** (cuando se active el flujo PR en el futuro):
  - `feat(REQ-AUTH-001): añadir login con Google` — primera implementación.
  - `feat(REQ-AUTH-001 change-002): sesión a 30 min` — implementación de un Delta Spec.
  - `fix(ERR-001): manejar cookies bloqueadas en OAuth callback` — corrección de error.
  - `docs(REQ-AUTH-001): añadir change-002 (sesión 30 min)` — solo cambio de papel, sin código todavía.
  - `refactor(REQ-AUTH-001): extraer SessionExpiryWarning a componente compartido` — refactor sin cambio funcional.
  - `chore(db): regenerate ER diagram` — autogenerado por CI.

El prefijo (`feat`/`fix`/`docs`/`refactor`/`chore`) permite a `/audit` clasificar automáticamente la naturaleza del cambio al analizar git log.

---

## Apéndice D — Roadmap de adopción incremental

Si llevas la metodología a un proyecto greenfield, sigue todos los pasos desde el principio. Si la traes a un proyecto existente (brownfield):

1. **Semana 1** — Llenar `docs/` con vision, tech-stack, personas, constraints.
2. **Semana 2** — Definir los REQs retroactivos para las features existentes (status `implementado`, sin Delta Specs porque ya estaba implementado). Esto produce el "estado actual" como punto de partida.
3. **Semana 3** — Configurar agentes, slash commands, GitHub Action del diagrama, primer `/audit` para detectar drift inicial.
4. **Semana 4 en adelante** — Todo cambio nuevo pasa por `/change-req`. Los errores existentes se documentan retrospectivamente con `ERR-NNN.md` solo si son recurrentes o instructivos.

La activación del flujo de PR review (cuando el equipo lo decida) consiste solo en habilitar branch protection en `main` exigiendo aprobación; el resto de la metodología no cambia.
