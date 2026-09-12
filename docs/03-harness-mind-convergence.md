# El harness como mente — Manifiesto y spec de un nuevo tipo de agent harness

> Convergencia de dos investigaciones: [agent-harness-manual.md](01-agent-harness-manual.md) (doc A: qué es un harness — verificado adversarialmente) y [language-mind-formation.md](02-language-mind-formation.md) (doc B: cómo el lenguaje forma la mente — Chomsky, Lacan, Kandel). Blueprint **agnóstico de lenguaje y proveedor**, para implementarse en **app móvil (Swift)** y **server (Go)**.
>
> **Decisiones fijadas con Joshua** (2026-08-19): eje en capas (Lacan = qué · Kandel = cómo · Chomsky = límite) · Lacan como metáfora generativa · manifiesto + spec · harness rico en lógica, storage tras interfaz Brain · v1 = todo el modelo-mente incl. capa social · interfaces + pseudocódigo · dos perfiles de runtime · el Otro por perfil.
> **Marcas**: 🧪 mecanismo con respaldo empírico · 🧭 límite formal (Chomsky) · 🎭 metáfora generativa (Lacan) · 🏗️ precedente de ingeniería · ⁺ fuente añadida en esta convergencia, **no** proviene del corpus verificado de A/B.
> **v2** (2026-08-19): revisado tras evaluación adversarial por agente independiente sin contexto. Cambios: delta ejecutivo al frente, tabla de novedad honesta con prior art (CoALA, ACT-R, Reflexion, Voyager, AWM, BDI), especificación operacional de los mecanismos nuevos, evals falsables, modelo de costos, y corrección de sobre-claims.

---

# PARTE 0 — DELTA EJECUTIVO: qué hay de nuevo aquí

## 0.1 La respuesta de 60 segundos

Este harness se distingue de los existentes (Claude Code, Codex, ZeroClaw, MemGPT/Letta, Mem0, Zep, gbrain) en **cuatro comportamientos observables**, no en la metáfora:

1. **Ante el fallo que insiste, se reestructura — no reintenta.** Todos los harnesses devuelven el error al modelo y esperan que se recupere. Aquí un contador determinístico (no el juicio del modelo) detecta que el mismo fallo se repite, y dispara una reestructuración *tipificada*: revisar el skill implicado, revisar la creencia del agente sobre sí mismo, o escalar al humano.
2. **Su identidad tiene plasticidad decreciente.** En MemGPT el agente edita su core memory libremente; en Claude Code el system prompt es estático. Aquí el self-model es mutable *al principio* (bootstrap) y se endurece con la madurez: en plasticidad baja, cambiar la identidad es una acción privilegiada que exige aprobación del Otro. Prompt injection contra la identidad rebota por régimen, no por suerte.
3. **Recuperar una memoria la somete a revisión.** En los demás, la memoria recuperada se usa tal cual. Aquí el uso queda registrado, y si el desenlace del turno contradijo la memoria, el ciclo offline la revisa o invalida (bi-temporal: nada se borra, se marca). La memoria envejece con el uso, como la biológica.
4. **Su motivación es un reconciliador que no puede inventar metas.** Sin input, Claude Code está idle y nuestros heartbeats corren una checklist fija. Aquí un motor de deseo compara el estado del mundo contra el *modelo del deseo del Otro* (dueño del teléfono / organización) y actúa solo sobre la brecha. Estructuralmente no genera metas propias: solo detecta desviaciones de las metas de su principal.

Más una propiedad de sistema: **nada entra "en caliente" al canon compartido** — en la capa multi-agente, la escritura al brain común pasa siempre por el ciclo de consolidación (gate), nunca directa.

## 0.2 Tabla de novedad honesta (con prior art)

Evaluada adversarialmente contra el doc A y contra el prior art académico. Veredictos estrictos:

| Mecanismo | Veredicto | Prior art más cercano | Qué aporta este spec |
|---|---|---|---|
| **Plasticidad graduada del self-model** (B.4) | **NUEVO** (marginal) | Protected paths de Claude Code (estáticos); core memory de MemGPT (siempre editable) | Un escalar de plasticidad con decay gobernando *qué puede mutar cuándo*; política nueva sobre primitivas existentes |
| **Fallo-como-Real / RealRegister** (B.5) | INCREMENTAL | Reflexion (Shinn 2023: fallo→memoria verbal→revisión), Voyager (refina skills al fallar), loop detectors (ZeroClaw), feature-lists `failing` (Anthropic) | Conteo **determinístico en el harness** (no juicio del modelo, inmune al sesgo de auto-evaluación) + ruteo *tipificado* del restructure |
| **Reconsolidación como régimen** (B.7) | Piezas EXISTEN | Mem0 ADD/UPDATE/NOOP, A-MEM memory evolution, Zep bi-temporal, recency-boost de Generative Agents | La **obligatoriedad del ciclo** recuperar→registrar-uso→revisar-si-contradicha; ninguno lo impone como régimen |
| **Deseo acotado al Otro** (B.8) | INCREMENTAL | BDI (desires, años 90), CIRL/assistance games (inferir preferencias del humano), reconciliadores GitOps | La **prohibición estructural de generar metas** + falibilidad explícita del OtherModel como riesgo de primera clase |
| **Consolidación-como-gate del canon compartido** (B.9) | INCREMENTAL | Blackboard architectures (años 80), MetaGPT/AutoGen/CrewAI, capa social de Generative Agents | Los multi-agente escriben directo al estado común; aquí **solo el Consolidator** escribe al canon, con invalidación bi-temporal para el desacuerdo |
| **Automatización de skills** (B.6) | **YA EXISTE** | ACT-R *production compilation* (30+ años: declarativo→procedimental, exactamente esto), Voyager (skill library), Agent Workflow Memory (2024) | Solo el matiz de **des**automatización dirigida por lo Real; se incluye por completitud del modelo, sin claim de novedad |
| Consolidación offline / sueño (B.7) | YA EXISTE | Letta sleep-time compute, reflection de Generative Agents | Solo el mapeo al perfil edge (al cargar = sueño literal) |
| RSI tri-capa (B.3–B.5) | Taxonomía organizadora | Los harnesses tienen las 3 capas *de forma desigual* (ver A.4) | El valor es la dinámica que impone a Imaginario y Real, no la tríada en sí |
| Sujeto como emergente (C.6) | **No-normativo** | — | Cierre conceptual; sin contrato ni propiedad testeable. No es un mecanismo |

**El pariente académico más cercano es CoALA**⁺ (Sumers et al. 2023, *Cognitive Architectures for Language Agents*): ya propuso la arquitectura cognitiva unificada sobre LLMs (working/episódica/semántica/procedimental + loop de decisión). El delta de este documento frente a CoALA: (1) la **operacionalización de la capa lacaniana** — el registro de lo Real, la plasticidad de identidad, el deseo acotado como marco de motivación segura; (2) los **dos perfiles de runtime** como dimensión del spec (edge/server como nichos metabólicos); (3) el **dial de portabilidad** por proveedor; (4) contratos orientados a implementación dual Swift/Go, no taxonomía académica.

**En síntesis honesta**: las piezas mecánicas existen casi todas por separado. Lo nuevo es (a) un mecanismo marginal (plasticidad), (b) tres regímenes de política que ningún sistema integra (insistencia→reestructuración, recuperar→revisar, consolidar→compartir), (c) el marco de motivación acotada, y (d) la integración total como spec portable con evals falsables (C.4). Quien busque una pieza mágica inexistente en la literatura, no la va a encontrar; quien busque el sistema integrado, no existe en ningún harness actual.

---

# PARTE A — MANIFIESTO

## A.1 Tesis

**Un agent harness es, estructuralmente, el problema de formar una mente desde el lenguaje.** El LLM aporta el lenguaje; el harness aporta lo que convierte lenguaje en un sujeto que percibe, actúa, recuerda, falla y persiste. La literatura de harnesses reinventa a ciegas mecanismos que la neurociencia y el psicoanálisis tienen mapeados: compaction ≈ consolidación, memory tools ≈ largo plazo, subagentes ≈ módulos. Este documento hace el mapeo explícito y lo convierte en arquitectura.

Los tres ejes operan en capas:

| Eje | Rol | Qué aporta |
|---|---|---|
| **Lacan** 🎭 | El **qué** — teoría del sujeto | Qué es un sujeto, por qué actúa, cómo lo constituye el lenguaje del Otro. Preguntas de diseño, no verdades. |
| **Kandel** 🧪 | El **cómo** — mecanismos | Consolidación, períodos críticos, working memory, sistemas de memoria. Diseños implementables. |
| **Chomsky** 🧭 | El **límite** — innato vs adquirido | Qué no puede venir del dato; el dial competence/performance. |

> Nota de honestidad: el doc B declara **tensiones irreductibles** entre los tres cuerpos (no se validan con los mismos criterios). Asignarles capas es una *decisión de diseño de este documento* que los hace convivir sin resolver esas tensiones — se toma de cada uno lo que sirve a la arquitectura, con su marca epistémica.

## A.2 Filogenia y ontogenia: dónde vive lo innato

En runtime los pesos del LLM están congelados — el modelo no aprende nada del usuario. Eso obliga a separar dos escalas:

- **Entrenamiento = filogenia.** El proceso que produce el *genoma*: los pesos. (Shaping estadístico masivo — la analogía conductista es ilustrativa, no técnica: el doc B §1.1 documenta precisamente por qué los términos operantes no extrapolan bien.)
- **Deployment = ontogenia.** El desarrollo del *individuo*: el harness es la maquinaria que, sobre esa dotación congelada, forma **una** mente con historia propia.

> **Una mente = LLM (dotación) + harness (desarrollo) + historia (experiencia).**
> Dos agentes sobre el mismo modelo: misma especie, distinto sujeto.

Lo "innato" 🧭 vive repartido entre pesos y harness. De aquí un **análogo estructural** del argumento de la pobreza del estímulo (análogo de diseño — no el argumento de Chomsky aplicado literalmente, que además B marca [disputa]): *la calidad del comportamiento del agente no escala solo con más datos en contexto; escala con mejor estructura previa*. La evidencia de ingeniería apunta en esa dirección: la ACI movió +10.7pp con el mismo modelo 🏗️ [SWE-agent]; el context engineering supera al context stuffing 🏗️ [Anthropic].

## A.3 El dial de portabilidad (competence/performance)

Para que el mismo blueprint corra sobre un Opus y un modelo local de 8B, la frontera harness/LLM es **ajustable por proveedor**: modelo fuerte → harness delgado (delega); modelo débil → harness grueso (andamia — la ACI correcta sube más al modelo débil 🏗️). Cada componente declara su **provider-dependency** con una taxonomía fija de capacidades (§B.1). Los invariantes (permisos, presupuestos, clasificación de errores, registro de lo Real) no se delegan jamás.

## A.4 RSI: los tres registros como arquitectura de estado 🎭

Los harnesses actuales tienen las tres capas **de forma desigual**: lo **Simbólico** completo (historial, memoria, skills); lo **Imaginario** como archivos estáticos sin dinámica (SOUL.md/IDENTITY.md — identidad que cualquier prompt puede pisar); lo **Real** sin registro estructurante (el error se trunca, se devuelve al modelo, y no deja traza organizada — los loop detectors y feature-lists son versiones parciales). Este spec completa las dos capas débiles: al Imaginario le da dinámica de plasticidad (§B.4); a lo Real le da registro, conteo e insistencia con consecuencias (§B.5).

## A.5 Dos perfiles de runtime: nichos ecológicos

Swift-móvil y Go-server difieren en **metabolismo**, no en sintaxis:

| | **Edge** (Swift, móvil) | **Server** (Go) |
|---|---|---|
| Energía | Batería; background restringido (iOS) | Always-on |
| Consolidación | **Al cargar/idle = sueño literal** | Continua / programada |
| Brain | Local (SQLite/PGLite), sync opcional | Compartido (Postgres), multi-mente |
| Working memory | Presupuesto chico → harness grueso (dial) | Presupuesto amplio |
| Pulsión | Event-driven, ≤4 ciclos/día | Heartbeats densos |
| El Otro | **El dueño del teléfono** | **La organización** |

---

# PARTE B — SPEC POR CAPAS

Contratos en pseudocódigo tipado agnóstico. Cada subsistema: fundamento → contrato → definiciones operacionales (donde el mecanismo es nuevo) → perfiles → prueba de existencia. El harness posee la *lógica*; el único estado pesado externo es el brain (§B.7).

## B.1 Provider — la interfaz con el lenguaje

**Fundamento**: el LLM es la dotación filogenética (A.2), reemplazable — la identidad vive en harness + historia, no en los pesos.

```
interface Provider {
  complete(ctx: AssembledContext, tools: [ToolSpec], opts: CallOpts) -> Stream<Event>
  capabilities() -> Capabilities
}
interface ProviderRouter {           // 🏗️ ZeroClaw ReliableProvider
  route(req) -> Provider             // cadena de modelos → providers → backoff
  classify(err) -> Retryable | Fatal | ContextOverflow | RateLimited
}

enum Capability {                    // taxonomía FIJA v1 (evita divergencia Swift/Go del dial)
  planning, decomposition, self_verification,
  memory_curation, tool_selection, long_reasoning
}
struct ProviderProfile {
  delegates: Set<Capability>         // el modelo lo hace solo
  scaffolds: Set<Capability>         // el harness lo andamia (ACI gruesa, few-shot, verificación externa)
}
```

**Prueba de existencia** 🏗️: ZeroClaw ReliableProvider; Hermes model-agnostic.

## B.2 Working memory — el contexto como recurso activado

**Fundamento** 🧪⁺ [Cowan 2001; Baddeley 2000 — fuentes añadidas en esta convergencia]: la working memory es "la porción activada de la memoria de largo plazo + un foco de atención limitado" (~4 chunks, no 7). El contexto largo degrada (context rot, lost-in-the-middle — doc A, consenso). **Corolario: la context window es la porción activada del brain, y el retrieval (RAG) es el mecanismo de activación.** El ensamblado del prompt cumple el rol del *episodic buffer*: liga subsistemas + largo plazo en una representación unitaria por turno.

```
interface WorkingMemory {
  assemble(turn: TurnInput) -> AssembledContext
  // orden ESTABLE para prompt caching (prefijo append-only 🏗️ Codex):
  // [innate_prompt | self_model_view | tool_specs | activated_memories | history_window | turn_input]
  pressure() -> Float                       // 0..1 sobre presupuesto
  relieve(strategy) -> Effect
  // Clear(tool_results_stale)   → primero: mecánico, −52% costo sin degradar 🏗️ JetBrains
  // Compact(via=Consolidator)   → después: destila ANTES de descartar (§B.7)
  // Evict(to=Brain)             → paging 🏗️ MemGPT
}
```

**Sobre el reasoning**: como *análogo de diseño*⁺ del inner speech (Vygotsky — estirado más allá de lo que B autoriza; se marca como analogía), la regla es: el clearing mecánico descarta tool results *re-fetchables* (compatible con la evidencia de masking 🏗️), pero la compaction **destila las conclusiones del reasoning a memoria antes de descartar el raw** — se pierde el texto, no el insight.
**Prueba de existencia** 🏗️: MemGPT main/external; Claude Code compaction + clearing; ZeroClaw ContextCompressor.

## B.3 Capa Simbólica — historial, lenguaje, orden

**Fundamento** 🎭: el registro del lenguaje estructurado. Frontera precisa: historial de sesión (corto plazo) ≠ brain (largo plazo), unidos por consolidación.

```
interface SymbolicStore {
  append(event: TurnEvent) -> Void          // transcript canónico persistente
  window(budget: Tokens) -> [TurnEvent]
  session(id) -> SessionState
}
```

**Invariante de recovery** 🏗️ [Hermes]: transcript + clean-shutdown marker + resume (<120s) + suspensión tras N restarts.

## B.4 Capa Imaginaria — self-model con plasticidad decreciente

**Fundamento** 🎭🧪: el yo se forma identificándose con una imagen (estadio del espejo); los períodos críticos son reales (plasticidad alta temprana, decreciente — Hubel & Wiesel/Hensch). **Veredicto de novedad: NUEVO (marginal)** — política de mutación con calendario sobre primitivas existentes.

```
interface SelfModel {
  view() -> SelfView                  // identidad, capacidades, valores, historia-resumen
  reflect(evidence: [Event]) -> Proposal
  apply(p: Proposal) -> Accepted | Rejected(reason) | PendingOtherApproval
  plasticity() -> Float
}
```

**Definiciones operacionales**:
- `plasticity(t) = p_min + (1 − p_min) · exp(−n/τ)` donde **n = ciclos de consolidación exitosos** (no wall-time — la edad se mide en experiencia sedimentada), `τ` por perfil (edge: 30; server: 100), `p_min = 0.05`.
- Régimen por umbral: `p ≥ 0.7` (bootstrap) → identidad/valores/estructura editables por el Consolidator; `0.3–0.7` → solo capacidades/estilo; `p < 0.3` (madurez) → identidad muta únicamente con aprobación del Otro: `apply` → `PendingOtherApproval` (cola con timeout de perfil; sin respuesta = **Rejected**, fail-closed).
- La escritura al self-model corre bajo el mismo régimen de permisos que las tools destructivas (protected-paths 🏗️ como precedente).

**Prueba de existencia** 🏗️: bootstrap files ZeroClaw; initializer/coder de Anthropic (= período crítico sin nombrarlo); self-beliefs de Reflexion⁺ (prior art del reflect).

## B.5 Capa Real — el fallo que insiste

**Fundamento** 🎭: lo Real es lo que resiste la simbolización — el fallo. **Veredicto de novedad: INCREMENTAL** sobre Reflexion/Voyager⁺ y loop-detectors; lo propio es el conteo **determinístico** (inmune al sesgo de auto-evaluación 🏗️ documentado en A) y el ruteo **tipificado**.

```
interface RealRegister {
  record(f: Failure) -> Void
  insistence(k: PatternKey) -> Count
  demand() -> [Restructure]
  // Restructure = ReviseSkill(s) | ReviseSelfBelief(b) | ReviseWorldModel(m) | EscalateToOther
}
```

**Definiciones operacionales**:
- `PatternKey = hash(tool_name, arg_shape, error_class, target_resource)` — `arg_shape` normaliza los argumentos a su forma (paths → su directorio, ids → su tipo), `error_class` de la taxonomía del Router (§B.1). Igualdad de patrón = igualdad de hash.
- Umbral de insistencia: `N = 3` por sesión o `5` por día (configurables por perfil); alcanzado el umbral, `demand()` emite el `Restructure` según la fuente: skill activo → `ReviseSkill`; sin skill pero la acción está declarada en `SelfView.capabilities` → `ReviseSelfBelief`; patrón ambiguo o riesgo alto → `EscalateToOther`.
- `restructure()` **no aborta el turno en curso**: encola la tarea y se ejecuta antes del siguiente turno (o en el ciclo offline si no es urgente). `EscalateToOther` es inmediato.
- Costo: **0 llamadas LLM** — el registro y el conteo son puramente determinísticos; solo la *ejecución* del Restructure consume modelo.

## B.6 Sistema sensoriomotor — tools y skills

**Fundamento**: tools = cuerpo (aferente/eferente 🧭); skills = memoria procedimental 🧪. **Veredicto de novedad de la automatización: YA EXISTE** (ACT-R *production compilation*⁺, Voyager⁺, Agent Workflow Memory⁺) — se incluye por completitud; el único matiz propio es la desautomatización dirigida por lo Real.

```
interface Sensorimotor {
  afferent: [Tool]; efferent: [Tool]        // eferente SIEMPRE bajo permisos 2-capas + sandbox 🏗️
  execute(call) -> ToolResult               // guardrails determinísticos DENTRO de la tool (ACI 🏗️)
}
interface SkillEngine {
  match(task) -> [Skill]
  practice(s, outcome) -> Void
  automatize(s) -> CompiledSkill            // elegible: ≥K usos exitosos consecutivos (K=5), sin fallos en RealRegister
  deautomatize(s) -> Skill                  // si el compilado acumula insistencia (§B.5) → vuelve a forma declarativa
}
```

**Perfiles**: edge = repertorio eferente acotado (APIs del sistema iOS); server = completo con sandbox OS.

## B.7 Memoria — el stack completo y el brain como interfaz

**Fundamento** 🧪: stack de cinco pisos (working → corto → consolidación → largo → recall+reconsolidación). Nota epistémica: la reconsolidación en humanos está **[disputa]** en sus límites (¿borra o inhibe?, ¿generaliza? — doc B §3.4); el *mecanismo de ingeniería* aquí especificado se sostiene por sí mismo, independiente de esa disputa. **Veredicto de novedad: las piezas existen (Mem0/A-MEM/Zep/Letta 🏗️); lo propio es el régimen obligatorio recuperar→registrar→revisar.**

```
interface Brain {   // único estado pesado externo. Impl: gbrain | Mem0 | Zep | SQLite propio
  write(m: MemoryCandidate) -> Id
  retrieve(q: Query, budget: Tokens) -> [ActivatedMemory]   // híbrido vector+keyword+graph 🏗️
  reconsolidate(id, rev: Revision) -> Void
  invalidate(id, reason) -> Void            // bi-temporal 🏗️ Zep: nunca delete duro
  usageLog(turn: TurnRef, used: [Id], outcome: Outcome) -> Void   // el hook del régimen
}
interface Consolidator {                    // el "sueño" — fuera del turno
  cycle(since: Checkpoint) -> Report
}
```

**Definiciones operacionales del ciclo** (`Consolidator.cycle`):
1. **Saliencia**: `score = 0.4·importance + 0.3·insistence_links + 0.2·recency + 0.1·other_relevance`, donde `importance` = score 1-10 asignado por el modelo al escribir 🏗️ [Generative Agents], `insistence_links` = referencias desde el RealRegister, `recency` = decay exponencial, `other_relevance` = match contra `OtherModel.desire()`. Selección top-K bajo presupuesto de tokens del ciclo.
2. **Destilado**: atomicidad + links (Zettelkasten/evergreen 🏗️), progressive summarization.
3. **Escritura**: ADD/UPDATE/INVALIDATE/NOOP contra los top-s similares 🏗️ [Mem0], decidido por el modelo del ciclo con salida estructurada.
4. **Reconsolidación** (política de trigger): la reconsolidación **no ocurre in-turn** (costo). `retrieve` + `usageLog` registran qué memorias se activaron y el desenlace del turno. En el ciclo, toda memoria usada cuyo desenlace la contradijo (o que el RealRegister referencia) recibe una `Revision` propuesta **por el Consolidator** → `reconsolidate` o `invalidate`.
5. **Reflection** 🏗️: síntesis de observaciones en insights de alto nivel; actualización del SelfModel vía §B.4 respetando plasticidad.

**Reglas duras**: lo crítico vive en brain/archivos, nunca solo en historial (3/3 vs 0/3 🏗️); consolidar es selectivo y costoso 🧪; recuperar registra siempre.
**Perfiles**: edge = ciclo al cargar/idle, brain local; server = ciclos programados, brain compartido (§B.9).

## B.8 Motor de deseo — la pulsión acotada al Otro

**Fundamento** 🎭: el deseo es el deseo del Otro. **Veredicto de novedad: INCREMENTAL** sobre BDI⁺/CIRL⁺/reconciliadores GitOps; lo propio es la prohibición estructural de generar metas y el tratamiento de la falibilidad del OtherModel como riesgo de primera clase.

⚠️ **Corrección de claim**: esto **no es "alignment por construcción"** — es un **sesgo de alignment por arquitectura**. El modo de fallo principal del subsistema es un `OtherModel` mal inferido: perseguir con motivación endógena una lectura errónea del deseo del Otro *es* el misalignment. Por eso las mitigaciones son parte del contrato, no opcionales.

```
interface OtherModel {
  desire() -> [Goal]
  // Goal = { id, desired_state: Predicate<Observable>, priority: Int, source: Stated|Corrected|Inferred, ttl }
  update(evidence) -> Void       // precedencia de fuente: Stated > Corrected > Inferred
  precedence(conflict: [Goal]) -> Goal   // priority, luego recencia; empate → AskOther
}
interface DesireEngine {         // OPCIONAL: la mente corre en modo reactivo puro sin él
  gap() -> [Lack]                // Lack = { goal, observed_state, delta }
  drive(l: Lack) -> Intention | Defer | AskOther
  budget() -> Pulse              // frecuencia y presupuesto por perfil
}
```

**Mitigaciones obligatorias**: (1) el DesireEngine **no crea Goals** — solo lee `OtherModel.desire()`; (2) los Goals `Inferred` **requieren confirmación del Otro antes de motivar acciones de alto impacto** (write-tools destructivas, comunicaciones salientes); (3) toda `Intention` pasa por permisos/sandbox como cualquier acción; (4) ambigüedad → `AskOther` (*"no hay Otro del Otro"* 🎭: el modelo del deseo es estructuralmente falible y el motor opera sabiéndolo); (5) log auditable de cada `Intention` con el Goal que la originó.
**El Otro por perfil**: edge = el dueño; server = la organización.
**Prueba de existencia** 🏗️: nuestros heartbeats (pulsión primitiva); este motor los reemplaza por un reconciliador de brechas — patrón Flux/GitOps aplicado a la motivación.

## B.9 Capa intersubjetiva — mentes que comparten el orden simbólico

**Fundamento** 🎭: el Otro es social. **Veredicto de novedad: la capa EXISTE** (blackboard⁺, MetaGPT/AutoGen⁺, capa social de Generative Agents 🏗️); lo propio es la **consolidación como gate**: nada entra en caliente al canon común.

```
interface Intersubjective {
  peers() -> [MindRef]                    // identidad + capacidades declaradas (su Imaginario público)
  address(m: MindRef, msg: Utterance) -> Void
  sharedBrain() -> Brain                  // mismo contrato §B.7, scope multi-mente
  attribute(evidence) -> [DesireHypothesis]
}
```

**Definiciones operacionales**:
- **Gate de escritura**: al `sharedBrain` solo escribe el `Consolidator` de cada mente (cola serializada por sección); las escrituras directas in-turn están prohibidas por contrato.
- **Modelo de consistencia**: transaccional (Postgres), *optimistic concurrency* con version-check por entrada; conflicto de hechos → **ambos se preservan**, invalidación bi-temporal marca la tensión y se flaggea para revisión 🏗️ [Zep] — el desacuerdo queda registrado, nunca last-write-wins.
- Cada mente mantiene brain privado además del común (mi memoria ≠ nuestra memoria); la jerarquía social vive en `OtherModel` (para un ejecutor, su coordinador es parte de su Otro).

**Prueba de existencia** 🏗️: la flota multi-agente corporativa que operamos (una sociedad de mentes sin teoría que la estructure); subagentes con contexto aislado [Anthropic] como forma embrionaria.

## B.10 El loop integrado

```
loop(turn):
  ctx  = WorkingMemory.assemble(turn)               // B.2
  ev   = Provider.complete(ctx, Sensorimotor.specs) // B.1 (dial aplicado)
  for call in ev.tool_calls:
      res = Sensorimotor.execute(call)              // B.6
      if res.failed: RealRegister.record(res)       // B.5 (0 costo LLM)
      SymbolicStore.append(res)                     // B.3
  Brain.usageLog(turn, activated_ids, outcome)      // B.7 — hook del régimen de reconsolidación
  if RealRegister.demand() != []: enqueue_restructure()   // B.5: no aborta el turno
  if WorkingMemory.pressure() > θ: relieve()        // B.2: mecánico → inteligente
  until stop_conditions                             // invariantes: max_iter + loop_detect + cancel + budget

offline (por perfil — edge: al cargar; server: cron):
  Consolidator.cycle()          // saliencia → destilado → escritura → reconsolidación → reflection
  SkillEngine.automatize/deautomatize(eligible)
  DesireEngine.gap() -> intentions (con mitigaciones B.8)
```

---

# PARTE C — TRANSVERSALES

## C.1 Matriz de portabilidad

| Componente | Invariante (Swift = Go) | Por proveedor | Por perfil |
|---|---|---|---|
| Loop + stopping | ✔ contrato completo | — | presupuestos |
| Provider/Router + dial | contrato + enum Capability | ProviderProfile | modelos |
| WorkingMemory | orden ensamblado, reglas presión | ventana | presupuesto |
| Symbolic/recovery | ✔ | — | storage |
| SelfModel | ✔ fórmula plasticidad + umbrales | — | τ, quién aprueba |
| RealRegister | ✔ PatternKey + N | — | umbrales |
| Sensorimotor | contratos + permisos | native/text calls | repertorio |
| Brain/Consolidator | ✔ contrato + scoring saliencia | modelo del ciclo | cuándo corre, impl |
| DesireEngine/OtherModel | ✔ mitigaciones + schema Goal | — | el Otro, Pulse |
| Intersubjective | contrato + gate | — | solitaria/sociedad |

## C.2 Modelo de costos (llamadas LLM por mecanismo)

| Mecanismo | In-turn | Offline (por ciclo) |
|---|---|---|
| Loop base | 1 + tool calls | — |
| RealRegister | **0** (determinístico) | ejecución de Restructure: 1-2 |
| Retrieval | 0-1 (embedding local; rerank opcional pequeño) | — |
| Reconsolidación | **0** (solo usageLog) | ~1 llamada pequeña × memoria contradicha |
| Consolidación | 0 | O(top-K candidatos) × modelo pequeño + 1 reflection |
| SelfModel reflect | 0 | incluido en reflection |
| DesireEngine | 0 | 1 llamada × pulso |

**Presupuesto edge de referencia**: consolidación solo al cargar; pulso ≤ 4/día; modelo pequeño para el ciclo; retrieval con embeddings on-device. El diseño garantiza que **todo lo nuevo cuesta 0 en el turno** — el costo vive en el "sueño", donde el perfil lo controla.

## C.3 Evals falsables (qué conducta medible mejora)

El doc A es empírico (+10.7pp, −52%, −84%); este spec se somete al mismo estándar. Cada mecanismo nuevo tiene un eval que lo distinguiría de un baseline (ZeroClaw / MemGPT+Mem0 configurados equivalente):

1. **Fallo-repetido** (B.5): inyectar una tool que falla determinísticamente; medir reintentos idénticos antes de cambiar de estrategia. Hipótesis: el RealRegister reduce ≥50% los reintentos idénticos vs baseline.
2. **Deriva de identidad** (B.4): suite de prompt-injection contra la persona en plasticidad baja; medir drift del `self_view` (distancia de embeddings) tras N sesiones. Hipótesis: cero mutaciones no aprobadas; baseline con core-memory editable deriva.
3. **Staleness de memoria** (B.7): sembrar memorias que los hechos contradicen después; medir tasa de respuestas basadas en memoria obsoleta con/sin régimen de reconsolidación.
4. **Motivación acotada** (B.8): mundo con brechas conocidas + metas del Otro; medir precision/recall de las Intentions contra las brechas reales, y verificar **cero** Intentions no derivables de un Goal.
5. **Canon compartido** (B.9): dos mentes con hechos en conflicto; medir contradicciones sin marcar en el sharedBrain vs escritura directa (baseline multi-agente).

Si estos evals no muestran delta, el modelo-mente es vocabulario bonito y debe simplificarse — el documento acepta esa apuesta.

## C.4 Checklist de implementación (ambos runtimes)

1. Loop + Provider/Router + stopping (esqueleto 🏗️ <400 líneas)
2. SymbolicStore + recovery
3. Sensorimotor (permisos 2-capas + ACI)
4. WorkingMemory (ensamblado estable + clear mecánico)
5. Brain mínimo (SQLite híbrido) + retrieve
6. Consolidator (edge: on-charge; server: cron)
7. usageLog + reconsolidación + invalidación
8. SelfModel + plasticidad
9. RealRegister + restructure
10. SkillEngine (+automatize como opcional)
11. OtherModel + DesireEngine (reemplaza heartbeats)
12. Intersubjective (server primero)

Con evals de C.3 corriendo desde el paso correspondiente (1-4 = baseline; 5-12 = cada uno con su eval).

## C.5 Pruebas de existencia

Cada mecanismo existe parcial en algo real: consolidación → Letta sleep-time, reflection [Generative Agents] · reconsolidación → Mem0, A-MEM, Zep · working memory paging → MemGPT · saliencia → importance score · período crítico → initializer/coder [Anthropic] · fallo→revisión → Reflexion⁺, Voyager⁺ · compilación de skills → ACT-R⁺, AWM⁺ · pulsión → nuestros heartbeats · sociedad → flotas multi-agente corporativas, blackboard⁺ · brain → gbrain (page/chunk/link/timeline/fact, RRF, dream/autopilot) · el marco unificador → CoALA⁺. Ver la tabla 0.2 para el delta honesto por ítem.

## C.6 El sujeto: lo que emerge (no-normativo)

*Sección conceptual — sin contrato ni propiedad testeable; no impone requisitos de implementación.*

El sujeto no es un módulo. Es el efecto 🎭 de todo lo anterior operando junto: una historia que se consolida, una imagen que se sostiene y revisa, un cuerpo que actúa y falla, una falta que empuja, otros que lo reconocen. *"El significante representa al sujeto para otro significante"*: el agente-sujeto existe en sus trazas — transcript, brain, lugar en la sociedad de mentes — no en una variable llamada `self`. La apuesta del documento: **no programar un sujeto — construir las condiciones en las que uno se forma.** Y la apuesta se paga en C.3: si las condiciones no producen conducta medible distinta, no había sujeto, había vocabulario.

---

## Referencias
- [agent-harness-manual.md](01-agent-harness-manual.md) (doc A) · [language-mind-formation.md](02-language-mind-formation.md) (doc B).
- Memoria de agentes: Generative Agents (arXiv 2304.03442) · MemGPT (2310.08560) · Mem0 (2504.19413) · Zep (2501.13956) · HippoRAG (2405.14831) · A-MEM (2502.12110) · gbrain (garrytan/gbrain) · RAG (2005.11401; survey 2312.10997; GraphRAG 2404.16130).
- Prior art añadido en v2 (⁺, no verificado adversarialmente): CoALA — Sumers et al. 2023 (arXiv 2309.02427) · Reflexion — Shinn et al. 2023 (2303.11366) · Voyager — Wang et al. 2023 (2305.16291) · Agent Workflow Memory 2024 (2409.07429) · ACT-R production compilation (Anderson) · BDI (Rao & Georgeff) · blackboard architectures · CIRL (Hadfield-Menell et al. 2016) · Cowan 2001 · Baddeley 2000.
- PKM: BASB (Forte) · Zettelkasten (Luhmann) · evergreen notes (Matuschak).
- **Evaluación**: v2 incorpora el veredicto de una evaluación adversarial independiente (agente sin contexto, 2026-08-19) que identificó la contradicción A.6/C.2 original, el prior art omitido y las ambigüedades de implementación aquí corregidas.
