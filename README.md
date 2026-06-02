# SDD Template para Claude Code

> Template clonable para construir proyectos de software con **Spec-Driven Development**, operado al 100% con Claude Code y vibe coding.

## Cómo opera

Tú das la dirección en lenguaje natural ("ahora la sesión debe durar 30 min", "añade pago con Stripe", "este error está roto"). Claude Code, con esta metodología, detecta el impacto, propone el plan, modifica los requisitos, implementa el código, genera tests, valida y commitea.

Tú apruebas en dos checkpoints (papel e implementación). **Cero programación manual.**

## ¿Qué es SDD?

**Spec-Driven Development**: la especificación en Markdown es la única fuente de verdad. El código es subproducto. Toda evolución empieza actualizando el spec.

Diseñado para absorber **cambios de última hora a alta velocidad** — como un MVP en evolución continua — sin destruir lo ya construido. Permite tener una historia completa, auditable y reversible de cómo el producto evolucionó.

## ¿Por qué este template?

- **Operativo, no documental.** Los comandos, skills y agentes ya están cableados. No tienes que escribir cómo invocar la metodología — solo tipear el comando o describir lo que quieres.
- **Bibliografía integrada.** El folder `sdd/` es la "biblioteca" que los agentes consultan literalmente cada vez que se invocan. La metodología se ejecuta a sí misma.
- **Diseñado para vibe coding seguro.** Los principios y reglas duras protegen contra los errores típicos de la IA (alucinaciones, regresiones, drift).

## Estructura del repo (template)

```
.
├── README.md                          ← Estás aquí
├── CLAUDE.md                          ← Instrucciones para Claude Code en el proyecto
├── .claude/
│   ├── settings.json                  ← Permisos y configuración
│   ├── agents/                        ← 4 subagentes (planner, reviewer, implementer, auditor)
│   ├── commands/                      ← 8 slash commands (acciones)
│   └── skills/                        ← 3 skills (autodescubiertas)
├── sdd/                               ← Biblioteca metodológica modular
│   ├── README.md                      ← Índice maestro
│   ├── concepts/                      ← Principios, estructura, convenciones, errores, validación, contexto
│   ├── templates/                     ← Plantillas REQ, INDEX, error, ADR, ledgers, docs/
│   ├── workflows/                     ← Un manual por cada workflow
│   └── reference/                     ← Migraciones, GitHub Actions, commits, adopción brownfield
├── requisitos.md                      ← Versión monolítica histórica (referencia comparativa)
└── Investigacion.md                   ← Investigación previa que informó la metodología
```

Tras ejecutar `/init-project`, se generan:
```
├── docs/                              ← Vision, tech-stack, personas, constraints
├── requirements/                      ← REQs versionados por dominio
├── database/                          ← Schema + diagrama ER autogenerado + migraciones
├── brand/                             ← Identidad visual + voz
├── decisions/                         ← ADRs
├── ledger.md                          ← Bitácora global de cambios
└── .github/workflows/sync-schema.yml  ← Diagrama ER autogenerado en CI
```

## Cómo se usa

### 1. Clonar para un proyecto nuevo
```bash
git clone <este-template> mi-proyecto
cd mi-proyecto
git remote remove origin
git remote add origin <repo-del-producto>
```

### 2. Abrir Claude Code y bootstrapear
Tipea:
```
/init-project
```

Claude crea `docs/` con plantillas vacías y se detiene. Llenas los 4 archivos (vision, tech-stack, personas, constraints) y vuelves a ejecutar `/init-project`. Claude termina el setup, configura el GitHub Action del diagrama ER, y te entrevista para esbozar los primeros REQs.

### 3. Operar en lenguaje natural

A partir de aquí, hablas y la IA ejecuta:

| Lo que tú dices | Lo que pasa |
|---|---|
| *"Crea un requisito para login con Google"* | Claude propone estructura → tú apruebas → `/new-req` crea la carpeta. |
| *"La sesión debe durar 30 min en vez de 60"* | `/change-req` arranca el flujo de 3 agentes (planner → reviewer → implementer) con 2 checkpoints. Cambio aplicado y commiteado. |
| *"Hay un bug en el login con cookies bloqueadas"* | La skill `sdd-classify-issue` te ayuda a decidir si es error o cambio de requisito. Si error, registra `ERR-NNN.md` con causa raíz, prompt original (si fue IA), acción preventiva. |
| *"¿Cómo va el proyecto?"* | La skill `sdd-status` reporta: REQs por estado, errores abiertos, drift candidato, actividad reciente. |
| *"Revierte el último cambio de auth"* | `/rollback` muestra plan (commits + migración inversa), tú apruebas, ejecuta. |
| *"Hay drift entre código y spec"* | `/audit` detecta inconsistencias y clasifica si son cambios no documentados o errores. |
| *"Limpia las ramas que ya mergeé"* | `/cleanup-branches` detecta PRs mergeados y borra las ramas locales correspondientes. |

### Flujo de git automático

Si el proyecto está versionado con git, cada workflow que modifica archivos crea su propia rama:

| Workflow | Rama creada | Tras mergear PR |
|---|---|---|
| `/new-req` | `req/<REQ-ID>-<slug>` | Ejecutar `/cleanup-branches` |
| `/change-req` (single REQ) | `req/<REQ-ID>-change-<NNN>-<slug>` | Ejecutar `/cleanup-branches` |
| `/change-req` (multi-REQ) | `req/<PRIMARY>-change-<NNN>-multi-<slug>` | Ejecutar `/cleanup-branches` |
| `/rollback` | `rollback/<REQ-ID>-to-<target>` | Ejecutar `/cleanup-branches` |

Si el proyecto **no** está versionado con git, todo este flujo se salta automáticamente. Los workflows detectan con `git rev-parse --git-dir` y se adaptan.

## Comandos disponibles

Los comandos son **acciones explícitas** que tú invocas tipeando `/comando`.

| Comando | Para qué | Riesgo |
|---|---|---|
| `/init-project` | Bootstrap del proyecto | Bajo (solo crea) |
| `/new-req` | Crear requisito desde descripción natural | Bajo |
| `/change-req` | Cambio con 3 agentes y 2 checkpoints | Medio (con checkpoints) |
| `/audit` | Detectar drift entre código y spec | Bajo (solo reporta) |
| `/rollback` | Revertir un change | **Alto** (con checkpoint) |
| `/sync-schema` | Regenerar diagrama ER local | Bajo |
| `/compact-req` | Compactar changes históricos | Bajo (mueve a archive/) |
| `/rebuild-index` | Recuperar INDEX desde changes | Medio (con checkpoint) |
| `/cleanup-branches` | Borrar ramas locales con PR mergeado (solo si hay git) | Bajo (con confirmación) |

Cada comando carga sus archivos de `sdd/` correspondientes — ver "Mapa de operaciones" en `sdd/README.md`.

## Skills disponibles (autodescubiertas)

Las skills son **helpers proactivos**: Claude las activa solo cuando detecta que aplican. **No ejecutan cambios destructivos** — solo guían y reportan.

| Skill | Cuándo se activa | Para qué |
|---|---|---|
| `sdd-status` | *"¿cómo va el proyecto?"*, *"estado"*, *"qué falta"* | Reporta estado del proyecto |
| `sdd-classify-issue` | *"hay un bug"*, *"no funciona"*, *"error en X"* | Clasifica si es error o cambio de requisito antes de derivar al flujo correcto |
| `sdd-guide` | Primera interacción en sesión / *"¿qué es esto?"* | Onboarding: principios + reglas duras + mapa de comandos |

## Subagentes

Los subagentes son **trabajadores especializados** invocados por los comandos. Tú no los invocas directamente.

| Agente | Rol | Lo usa |
|---|---|---|
| `planner` | Detecta impacto, propone Delta Specs ADDED/MODIFIED/REMOVED | `/change-req` fase 1 |
| `reviewer` | Revisión adversaria del plan del planner | `/change-req` fase 2 |
| `implementer` | Implementa código, genera tests, smoke test | `/change-req` fase 3 |
| `auditor` | Detecta drift y clasifica cambios vs errores | `/audit` |

Cada uno corre como subproceso con **contexto aislado**. El main session solo ve sus resultados compactos. Esto protege la ventana de contexto y aplica la "regla adversaria" del reviewer (no ve el contexto del planner).

## Los 7 principios fundamentales

1. La especificación es la única fuente de verdad. El código es subproducto.
2. Actualizar el spec ANTES que el código.
3. Cambios atómicos versionados, nunca destructivos. Historia navegable.
4. Los errores son aprendizaje institucional. Causa raíz + prompt detonante + acción preventiva.
5. La IA propone, el humano decide. Dos checkpoints (papel → implementación).
6. Trazabilidad bidireccional: cada REQ sabe qué código implementa.
7. Cambio de requisito ≠ corrección de error.

Detalle en `sdd/concepts/principles.md`.

## Las 5 reglas duras (no negociables)

1. `requirements/**/initial.md` jamás se modifica.
2. `change-*.md` solo se modifican para marcar `superseded`.
3. `INDEX.md` "Estado consolidado actual" es la única fuente de verdad del estado vigente.
4. `database/diagram.mmd` es autogenerado.
5. Cambio de requisito ≠ corrección de error.

Estas viven en `CLAUDE.md` para que Claude las recuerde en cada sesión.

## Estrategia de carga de contexto (cómo no saturar a Claude)

Los agentes cargan información **en tiers** para no leer todo en cada operación:

| Tier | Qué carga | Quién lo usa |
|---|---|---|
| **0** | `ledger.md` global + `docs/` | Siempre |
| **1** | `INDEX.md` de todos los REQs (estado consolidado actual) | Planner, auditor |
| **2** | `manifest.md` de REQs impactados | Implementer, planner (bajo demanda) |
| **3** | `change-NNN.md` históricos, `initial.md` | Solo `/rollback`, `/rebuild-index`, audit profundo |

Esto evita saturar la ventana de contexto y mezclar estados ya superados con el vigente. Detalle en `sdd/concepts/context-loading.md`.

## Validación de cambios (3 capas obligatorias)

Todo cambio implementado por `/change-req` pasa por 3 capas antes del commit:

1. **Tests automáticos** generados desde los criterios de aceptación del Delta Spec (NO desde el código que el implementer escribió — evita auto-validación circular).
2. **Checklist humano** que tú marcas a ojo.
3. **Smoke test** en dev server local.

Detalle en `sdd/concepts/validation.md`.

## Adoptar SDD en un proyecto existente (brownfield)

Si quieres aplicar SDD a un proyecto que ya tiene código:
1. Lee `sdd/reference/adoption-roadmap.md`.
2. Plan en 4 semanas: documentación base → REQs retroactivos → config técnica → operación normal.

## Validación de la metodología misma

Este repo mantiene **dos versiones** de la metodología en paralelo:

- **`requisitos.md`** — monolítico, 1076 líneas, referencia comparativa.
- **`sdd/`** — modular, 32 archivos, lo operativo.

Ambas describen exactamente lo mismo. El modular es el que los agentes consumen. El monolítico se mantiene como referencia para validar que ningún concepto se perdió en la división.

## Por dónde empezar a leer

- **Si quieres entender la filosofía:** `sdd/concepts/principles.md`.
- **Si quieres ver el flujo crítico:** `sdd/workflows/change-req.md`.
- **Si quieres el deep dive completo:** `requisitos.md` (monolítico).
- **Si quieres saber qué cargar para cada operación:** "Mapa de operaciones" en `sdd/README.md`.

## Créditos

Metodología destilada a partir de investigación sobre SDD (OpenSpec, GitHub Spec Kit), ReqToCode, Refactor Ledger, Behavioral Diff y prácticas operativas para IA generativa. Adaptada al flujo de vibe coding con Claude Code.

Detalle en `Investigacion.md`.
