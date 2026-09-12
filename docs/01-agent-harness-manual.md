# Agent Harness — Manual técnico

> Qué es un harness de agentes con LLMs, cómo está construido, y cómo se compara lo que operamos (OpenClaw, ZeroClaw) con Claude Code, Codex CLI, Gemini CLI, Aider, SWE-agent y Hermes.
>
> **Metodología**: investigación deep-research con verificación adversarial (102 agentes, 20 fuentes, 93 claims extraídos, 25 verificados con 3 votos independientes cada uno) + research complementario sobre docs oficiales + lectura del código de los harnesses que operamos en producción (ZeroClaw y una plataforma multi-agente construida sobre OpenClaw).
> **Marcas de confianza**: ✅ = claim verificado adversarialmente (3-0 o 2-1) · 📄 = fuente primaria oficial verificada, sin pasada adversarial · 📝 = fuente secundaria/blog · 🔍 = verificado en el código de harnesses que operamos en producción (las referencias file:line internas se omiten en esta edición pública).
> Fecha: 2026-08-17.

---

## 1. Definición: qué es un harness

**Un agent harness es todo el código determinístico que rodea al LLM y lo convierte en un agente**: el loop de ejecución, las herramientas, la gestión del contexto, el estado, la memoria, los permisos y la recuperación de fallos. El modelo decide *qué* hacer; el harness define *qué es posible hacer*, *cómo se ejecuta*, y *qué ve el modelo en cada turno*.

Tres formulaciones convergentes de la literatura:

1. **Scaffold de 4 componentes** — el paper de taxonomía de coding agents (arXiv 2604.03515) define el scaffold/harness como "the scaffolding code that surrounds the language model" y lo descompone en: **control loop**, **tool definitions**, **state management** y **context strategy**. 📄
2. **Agent-Computer Interface (ACI)** — el paper de SWE-agent (NeurIPS 2024) establece la base empírica: los LM agents son "a new category of end users with their own needs and abilities" que se benefician de interfaces construidas específicamente para ellos. Cambiar **solo la interfaz** (no el modelo) movió el resultado +10.7 puntos en SWE-bench frente a un shell plano. **El harness importa tanto como el modelo.** ✅ [arxiv.org/abs/2405.15793]
3. **La versión mínima** — "There isn't [a secret]. It's an LLM, a loop, and enough tokens." Un agente de edición de código funcional cabe en <400 líneas, la mayoría boilerplate (ampcode, "How to build an agent"). 📝

### Harness vs agente vs LLM

Anthropic distingue **workflows** (LLMs y tools orquestados por rutas de código predefinidas) de **agents** (el LLM dirige dinámicamente su propio proceso y uso de herramientas). El building block universal es el **augmented LLM**: LLM + retrieval + tools + memoria. ✅ [anthropic.com/engineering/building-effective-agents]

- **LLM**: la función `tokens → tokens`. Sin estado, sin efectos.
- **Agente**: el sistema completo que persigue un objetivo — LLM + harness + entorno.
- **Harness**: la capa intermedia. Es lo que tú construyes. El agente "existe" solo cuando el harness corre el loop.

> ⚠️ Nota de atribución: ni *Building Effective Agents* ni *Effective Context Engineering* usan la palabra "harness"; el mapeo harness = código orquestador es síntesis de este manual (y del paper 2604.03515, que sí lo llama scaffold). El término sí es wording oficial en *Effective harnesses for long-running agents* (Anthropic) y en "Codex harness" (OpenAI).

---

## 2. Anatomía de un harness

### 2.1 El agent loop

La especificación mínima: **un LLM usando tools en bucle con feedback del entorno, más stopping conditions**. Anthropic lo formula como `gather context → take action → verify work → repeat`. ✅ [building-effective-agents, claude-agent-sdk docs]

El inner loop concreto (idéntico en todos los harnesses estudiados):

```
history = [system_prompt, user_message]
loop:
    response = llm(history, tools)
    if response has no tool_calls:      # respuesta final
        return response.text
    for call in response.tool_calls:
        result = execute(call)           # con permisos/sandbox
        history.append(tool_result)
    check stopping_conditions            # max iterations, cancelación, loop detection
```

Implementaciones reales:

- **Codex CLI**: el loop es una **state machine** que orquesta usuario/modelo/tools en fases: prompt assembly (system instructions + tools + MCP + AGENTS.md + entorno local) → inference con streaming de SSE (reasoning, tool calls, respuesta) → ejecución de tools con retorno de errores al modelo para retry → assistant message final cierra el paso. Termina cuando el modelo emite un assistant message en vez de un tool call. 📝 [pragmaticengineer.com/p/how-codex-is-built, swequiz.com]
- **SWE-agent**: la clase `Agent` con `forward()` que "prompts the model and executes its action" en la shell de SWE-ReX. 📄
- **ZeroClaw**: `run_tool_call_loop()` 🔍 con `for iteration in 0..max_iterations` (default 10, `DEFAULT_MAX_TOOL_ITERATIONS`), tres salidas: respuesta sin tool calls, iteraciones agotadas, o cancellation token. Al agotar iteraciones **no falla en seco**: hace una llamada final sin tools pidiendo el mejor resumen posible. Detección de bucles adicional con `LoopDetector` (ventana de repeticiones) + hash de outputs idénticos consecutivos.

**Primitivas de control componibles**: el paper 2604.03515 identifica 5 (ReAct, generate-test-repair, plan-execute, multi-attempt retry, tree search); 11 de 13 sistemas analizados **componen varias** en vez de usar una sola. 📄 El loop plano tool-use (ReAct) es la base; las demás se anidan encima.

**Decisiones de diseño del loop**:
| Decisión | Opciones | Trade-off |
|---|---|---|
| Stopping condition | max iterations / presupuesto de tokens / cancelación | Muy bajo corta tareas legítimas; muy alto quema presupuesto en bucles |
| Al agotar presupuesto | fallar / pedir resumen final sin tools (ZeroClaw) | El resumen degradado > error opaco para canales de chat |
| Tool errors | abortar / devolver el error al modelo | Devolverlo permite auto-recovery; es el estándar en todos los harnesses estudiados |
| Streaming | eventos SSE por fase / respuesta completa | Streaming = UX + cancelación temprana; complejidad de parsing |

### 2.2 Relación con el LLM

**Prompt assembly**: cada turno, el harness reconstruye el prompt completo: system prompt + tool definitions + archivos de contexto (CLAUDE.md/AGENTS.md/GEMINI.md) + historial + entorno. En ZeroClaw: `build_system_prompt_with_mode()` inyecta los bootstrap files del workspace (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`) + `MEMORY.md` solo en la sesión main 🔍.

**Prompt caching**: el contenido estático (instructions, tools) va **al frente** del prompt para que cada turno haga cache hit sobre todo el contexto previo. En Codex, bugs de ordenamiento de tools causan cache misses costosos. 📝 [swequiz.com] Corolario: toda mutación del prefijo (reordenar tools, editar system prompt mid-session) invalida el caché — el harness debe tratar el prefijo como append-only dentro de una sesión.

**Abstracción de provider**: si el harness soporta múltiples modelos, necesita un trait/interfaz. ZeroClaw: trait `Provider` (`chat`, `chat_with_tools`, `stream_chat`, `warmup`) con ~15 providers implementados y un router 🔍. Hermes es model-agnostic explícito ("switch with `hermes model` — no code changes"). 📄

**Retries y resiliencia ante el API** (el LLM también falla): ZeroClaw envuelve todo en `ReliableProvider` con estrategia de 3 niveles — cadena de modelos → cadena de providers → retry con backoff exponencial; clasifica errores (4xx no-reintentables excepto 429/408; context-window y tool-schema errors SÍ recuperables), respeta `Retry-After` capado a 30s, y rota API keys round-robin en rate limits 🔍. Un bridge Claude-native que operamos hace lo equivalente: retry de sesión + cadena de fallback de modelos configurable, y los rate limits bypassean el retry de sesión para cambiar de modelo en vez de quemar el mismo token 🔍.

### 2.3 Context management — el problema central

**El contexto es un recurso finito con retornos decrecientes.** Anthropic define context engineering como "curating and maintaining the optimal set of tokens during LLM inference" — todo el estado (system prompt, tools, MCP, historial), no solo el prompt. La motivación es el **context rot**: a más tokens, menor recall preciso, por el costo de atención n² del transformer. Corroborado empíricamente por el estudio de Chroma (18 modelos frontier degradan con longitud de input). ✅ [effective-context-engineering-for-ai-agents]

Las **tres técnicas núcleo** y cuándo usar cada una (modelo de selección del cookbook oficial de Anthropic): ✅

| Técnica | Qué hace | Cuándo |
|---|---|---|
| **Compaction** | Comprime toda la ventana vía summarization | La ventana creció demasiado |
| **Tool-result clearing** | Reemplaza tool_results viejos con placeholders (edición mecánica, costo cero) | Datos stale re-fetchables dentro de la ventana |
| **Memory** | Mueve información fuera de la ventana a storage persistente | Debe sobrevivir entre sesiones |

**Compaction** — tres variantes:
- *Client-side* (Claude Code / Agent SDK): pasa el historial al modelo para comprimir detalles críticos (decisiones de arquitectura, bugs no resueltos) y descartar tool outputs redundantes. ✅
- *Server-side Anthropic*: `compact_20260112` (beta) destila el historial en un content block tipado al cruzar un umbral configurable (mín 50K, default 150K tokens) — el harness delega la summarization a la plataforma. ✅ [docs de compaction]
- *Server-side OpenAI*: endpoint `/responses/compact` que devuelve un item `encrypted_content` **opaco** que codifica el entendimiento latente del modelo — más pequeño que un resumen textual y privacy-preserving. 📝 [swequiz.com]

**Trade-off de fidelidad medido**: en el test del cookbook, los hechos de alto nivel sobreviven la summarization (3/3) pero los detalles específicos se pierden (0/3). ✅ La compaction no es lossless — lo crítico debe vivir en archivos/memoria, no en el historial.

**Tool-result clearing** (`clear_tool_uses_20250919`): en el demo oficial redujo el contexto 67% (128,740 → 43,060 tokens). "Una de las formas más seguras y de menor impacto de compaction". El agente re-llama la tool si necesita el dato. ✅

**Números independientes (JetBrains Research, SWE-bench Verified, 500 instancias, harnesses SWE-agent y OpenHands)**: 📄
- Observation masking (ventana rodante con placeholders) y LLM summarization recortan costos **>50%** vs contexto sin acotar.
- Masking, la técnica más simple, **no degrada**: 52% más barato con solve rates 2.6% *mayores* (Qwen3-Coder 480B).
- La summarization tiene costos ocultos: trayectorias ~15% más largas y >7% del costo por instancia en generar resúmenes.
- Híbrido (masking + summarization ocasional): +7% eficiencia sobre masking puro, +11% sobre summarization pura.

**Lección de diseño**: empieza con la técnica mecánica (clearing/masking) antes que la inteligente (summarization). Es más barata, más predecible, y empíricamente no pierde rendimiento.

**Enfoque alternativo — comprimir el input, no el historial**: el repo map de Aider. Un mapa conciso del repo (clases/funciones más importantes con signatures) construido parseando el AST con tree-sitter, seleccionado con un algoritmo de graph ranking sobre el grafo de dependencias entre archivos, con presupuesto `--map-tokens` (default 1k) ajustado dinámicamente según el estado del chat. 📄 [aider.chat/docs/repomap.html] Da conciencia estructural sin cargar el codebase.

**En nuestros harnesses** 🔍: ZeroClaw tiene `ContextCompressor::compress_if_needed()` que **persiste los summaries en memoria antes de descartar mensajes** — mitiga directamente el trade-off de fidelidad 3/3-vs-0/3 — más trim preventivo de tool results cuando el estimado supera el budget. Hermes tiene micro-compaction opt-in que pliega el intercambio más viejo en un running summary tras cada turno, protegiendo system prompt y prompts del usuario verbatim; trade-off documentado: rompe el prompt cache cada turno. 📄

### 2.4 Tools: la primitiva de ejecución

"Tools are the primary building blocks of execution for your agent. Tools are prominent in Claude's context window, making them the primary actions Claude will consider." ✅ [building-agents-with-the-claude-agent-sdk]

Las tool definitions son parte del contexto: definen el **espacio de acción** del modelo. El diseño de tools ES diseño de ACI (§1).

**El rango real es enorme** (paper 2604.03515): desde **cero tools LLM-callable** (Aider, que edita vía formatos de diff textual — search/replace blocks estilo git conflict markers, udiff, whole-file) hasta **37 clases de acción** (Moatless Tools). 6 de 13 agentes usan registro de tools completamente estático. 📄

**Detalles de ACI que mueven benchmarks** (SWE-agent): file viewer de 100 líneas por turno con scroll/search; editor que **rechaza ediciones sintácticamente inválidas con un linter** antes de aplicarlas; output vacío reemplazado por "Your command ran successfully and did not produce any output" (los modelos alucinan con outputs vacíos). 📄 Es decir: guardrails determinísticos dentro de la tool > confiar en que el modelo lo haga bien.

**Implementación típica** 🔍 (ZeroClaw): trait `Tool` (`name`, `description`, `parameters_schema`, `execute → ToolResult{success, output, error}`); registro como `Vec<Box<dyn Tool>>`; **wrappers de política** que envuelven tools (`RateLimited(PathGuarded(ShellTool))`) — composición de permisos como decorators, no ifs dentro de la tool.

**MCP (Model Context Protocol)**: el estándar para inyectar tools de terceros. Codex lo configura en `~/.codex/config.toml` compartido entre superficies, con approval por-server (`auto`/`prompt`/`writes`/`approve`) y lee el campo `instructions` del server como guidance. 📄 Caveat de escala: muchas tool definitions inflan el contexto; Anthropic propone code APIs como alternativa al tool-calling directo ("Code execution with MCP"). ✅ Mitigación en este mismo harness que lees: deferred tools + ToolSearch (cargar schemas on demand).

**Tools vs Skills** (patrón OpenClaw 🔍): Tools = permisos/capacidades (en `openclaw.json`); Skills = conocimiento procedural (`SKILL.md` con frontmatter `name`/`description`, el modelo lo invoca cuando la description matchea); TOOLS.md = convenciones en prompt. **Instalar un skill no otorga permisos.** Tres fuentes: shared (core), per-agent, y ClawHub/OCI externos instalados en boot.

### 2.5 Memoria

Tres mecanismos complementarios (Anthropic): ✅

1. **Archivos de contexto insertados íntegros al inicio** — CLAUDE.md "naively dropped into context up front". El modelo híbrido de Claude Code.
2. **Retrieval just-in-time** — el agente mantiene identificadores ligeros (paths, queries) y carga datos vía tools (glob/grep) en vez de pre-cargar todo.
3. **Memory tool** (`memory_20250818`, GA) — storage persistente **client-side**: el modelo decide qué guardar vía operaciones de archivo (view/create/str_replace/insert/delete/rename); el harness provee el backend; protocol prompt auto-inyectado ("ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING ELSE").

**El archivo de instrucciones como estándar**: AGENTS.md es de facto estándar cross-harness (Claude Code es el único agente mayor que no lo usa — usa CLAUDE.md). 📝 Mecánica de Codex: descubrimiento jerárquico (global → git root → cwd), concatenación root-down con los archivos más cercanos al final (prioridad de override), máximo un archivo por directorio, corte a 32 KiB. 📄 Gemini CLI: idéntico patrón con GEMINI.md, más imports modulares (`@./file.md`) y filename configurable (acepta `["AGENTS.md", "GEMINI.md"]`). 📄

**Distinción clave**: session state (historial del turno, efímero o resumible) ≠ memoria persistente (sobrevive sesiones). ZeroClaw los separa físicamente 🔍: sesiones en SQLite WAL (`sessions/sessions.db`), memoria en su propio subsistema — trait `Memory` (store/recall/forget/export...) con backends SQLite/Qdrant y **búsqueda híbrida vector + keyword (FTS5/BM25)**, expuesta además por REST API autenticada (`GET/POST/DELETE /api/memory`). Hermes: FTS5 session search + LLM summarization para recall cross-session, y skills como memoria procedural. 📄

### 2.6 Subagentes y orquestación

Dos beneficios arquitectónicos: **paralelización** y **aislamiento de contexto** — cada subagente tiene su propia context window y devuelve al orquestador solo lo relevante. En el patrón de research de Anthropic, un subagente explora decenas de miles de tokens pero retorna un resumen de ~1,000-2,000. ✅ [claude-agent-sdk, effective-context-engineering]

El subagente es la respuesta arquitectónica al context rot: el trabajo sucio (grep sweeps, lectura masiva) ocurre en contextos desechables; el contexto del orquestador acumula solo conclusiones.

**Dos formas de spawnearlos** (análisis del issue de Hermes): 📄
- **In-process**: clones del runtime con toolsets restringidos (Hermes `delegate_task` → `AIAgent` hijos en el mismo proceso). Barato, con visibilidad de tool-calls.
- **Cross-CLI vía PTY**: orquestar harnesses ajenos como procesos de terminal (spawn de PTY, inyección por stdin con cooldown, captura de stdout en chunks, auto-approve enviando "y"). Desventajas estructurales: parsing frágil entre versiones, cero visibilidad de tool-calls, autenticación por CLI, latencia de spawn.

En nuestros harnesses 🔍: ZeroClaw comparte el **presupuesto de iteraciones entre padre y subagentes** vía `AtomicUsize` — evita que el fan-out multiplique el costo sin control — y tiene delegate/swarm con estrategias Sequential/Parallel. OpenClaw: `subagents.maxConcurrent` en config.

### 2.7 Manejo de fallos

Capas de fallo distintas, mecanismos distintos:

| Capa | Fallo | Mecanismo | Ejemplo |
|---|---|---|---|
| API del LLM | 429/5xx/timeout | Retry + backoff, fallback de modelo/provider, rotación de keys | ZeroClaw `ReliableProvider` 🔍; cadena de fallback de modelos en nuestro bridge 🔍 |
| Tool | Exit ≠ 0, excepción | Devolver el error al modelo (truncado) y seguir el loop | Todos; ZeroClaw `truncate_tool_result` |
| Loop | Bucles, estancamiento | Loop detection, max iterations, watchdog, timeout escalado | ZeroClaw `LoopDetector` + stall watchdog 🔍 |
| Estado del filesystem | Edición destructiva | **Checkpoints**: commits automáticos (Aider: commit descriptivo por edición + `/undo`; commitea cambios pre-existentes antes de tocar nada 📄) o shadow git repo (Gemini CLI: snapshot en `~/.gemini/history/<hash>` antes de cada modificación, `/restore` revierte archivos + conversación + tool call 📄) |
| Proceso | Crash del harness | Transcript canónico en SQLite + recovery al boot: Hermes marca `.clean_shutdown`, y tras crash setea `resume_pending` en sesiones activas <120s, detecta loops de restart (3+ → suspend) y auto-continúa 📄 |
| Sesión | Corrupción del session file | Evict + sesión fresca (patrón verificado en producción 🔍; lección propia: renombrar el session file corrupto) |
| Contexto | Overflow | Compaction/trim preventivo (§2.3); errores de context-window clasificados como *recuperables* en ZeroClaw 🔍 |

**Modos de fallo específicos de long-running agents** (Anthropic, verificados ✅):
1. **Compaction no basta**: incluso Opus 4.5 en loop sobre el Agent SDK a través de múltiples context windows falla en construir una app production-quality con solo un prompt de alto nivel. La arquitectura propuesta separa un **initializer agent** (setup del entorno: init.sh, progress file, commit inicial) de un **coding agent** (progreso incremental por sesión con updates estructurados). ✅ [effective-harnesses-for-long-running-agents]
2. **Declaración prematura de completitud**: instancias posteriores ven progreso parcial y declaran el trabajo terminado. Mitigación: **feature list en JSON con todo inicialmente `failing`** — outline explícito de qué es "terminado". ✅
3. **Sesgo de auto-evaluación**: los agentes elogian su propio trabajo con confianza aunque sea mediocre. **Separar generador de evaluador** es una palanca fuerte (mitiga, no corrige — el self-preference bias persiste parcialmente incluso con judges separados, Panickssery et al. NeurIPS 2024). ✅

### 2.8 Seguridad: permisos y sandboxing

**Modelo de dos capas independientes** (convergencia Claude Code / Codex):
- **Permission layer**: se evalúa **antes** de ejecutar, sobre la descripción de la acción (el string del comando). Define *cuándo se pide aprobación*.
- **Sandbox**: lo aplica el OS **sobre el proceso corriendo** — "holds regardless of what the model chose to run and even if an allowed command does more than its name suggests". Define *qué es técnicamente posible*. 📄 [code.claude.com/docs/en/sandboxing; learn.chatgpt.com/docs/agent-approvals-security]

**Permission models**:
- Claude Code: read-only estricto por defecto; reglas allow/ask/deny evaluadas **deny → ask → allow** (primer match gana, la especificidad NO altera el orden — un deny amplio no admite excepciones); comandos compuestos divididos por `&&`/`||`/`;`/`|` y cada subcomando debe matchear independientemente; working directory boundary para escritura. Anti prompt-injection: **fail-closed matching** (lo no reconocido pide aprobación), detección de command injection que fuerza aprobación incluso para comandos allowlisted, y context window aislada para web fetch. 📄 Además: "Permission rules are enforced by Claude Code, not by the model. Instructions in CLAUDE.md don't change what Claude Code allows" — el prompt moldea la *intención*, los permisos acotan la *capacidad*. 📄
- Codex: tres approval modes (Read-only / Auto / Full Access), red **apagada por defecto**. Decisión asumida de producto: "We take a stance with the sandboxing that hurts us in terms of general adoption. However, we do not want to promote something that could be unsafe by default." 📝 Extra: un subagente **Guardian** (LLM separado) evalúa el riesgo de cada tool call. 📄 [2604.03515]
- Gemini CLI: `default`/`auto_edit`/`yolo`/`plan`; allowlist de shell con prefix matching, split de comandos encadenados, y deny chequeado antes que allow. 📄

**Sandboxing — primitivas por OS** (convergencia total entre vendors): **Seatbelt** (`sandbox-exec`) en macOS; **bubblewrap** + seccomp (o Landlock) en Linux; Gemini CLI añade Docker/Podman y gVisor. 📄 Anthropic liberó su runtime como open source (`@anthropic-ai/sandbox-runtime`). ✅ [claude-code-sandboxing]

- **Aislamiento de red por proxy**: internet solo vía unix socket → proxy **fuera** del sandbox que impone la allowlist de dominios; cero dominios pre-aprobados; credenciales git/signing keys viven fuera del sandbox. ✅ Limitación documentada: el proxy decide por hostname sin inspeccionar TLS → riesgo de domain fronting. 📄
- **Resultado medible**: el sandboxing redujo permission prompts **84%** en uso interno de Anthropic — resuelve la *approval fatigue* reemplazando aprobación por-acción con fronteras predefinidas. ✅
- **Letra pequeña** (análisis independiente 📝): el sandbox de Claude Code solo confina **Bash** — Read/Write/WebFetch/MCP/hooks los gobierna la capa de permisos; failure mode default-permisivo (sin bubblewrap disponible, advierte y sigue sin sandbox) vs fail-closed; límites de recursos: solo wall-clock timeout, sin caps de memoria/CPU/PID. Codex fuerza `.git` y `.codex` read-only con precedencia Deny > Write > Read.
- **Protected paths**: aun en directorios escribibles, Claude Code deniega escrituras a su propia config (`.claude/settings*`, hooks, `.mcp.json`, shell startup files, `.git/hooks`) — bloquea la auto-escalación de permisos. 📄

En nuestros harnesses 🔍: ZeroClaw implementa el mismo patrón multi-backend — `detect::create_sandbox` elige bubblewrap/landlock/firejail/seatbelt/docker, shell con env-var filtering, IAM **deny-by-default** por roles, allowlist de dominios por workspace, autonomy levels (Supervised/Full), prompt guard, leak detector, e-stop con OTP. OpenClaw: `tools.exec.security` explícito (gotcha: si falta, degrada a `deny` con aprobación manual por comando), allowlists de Slack por canal/usuario.

### 2.9 Extensibilidad: hooks, plugins, skills

Puntos de extensión sin tocar el core: **hooks** (interceptar el lifecycle — pre/post tool call, session start; Claude Code hooks, ZeroClaw `[hooks]`, OpenClaw `hooks.internal`: session-memory, command-logger, boot-md 🔍), **plugins** (con allow/deny lists en ZeroClaw 🔍), **skills** (conocimiento procedural cargado por relevancia — §2.4), y **MCP** (tools de terceros — §2.4). Codex expone el harness completo como **app server JSON-RPC sobre stdio** con primitivas Item/Turn/Thread y approvals que pausan un turn hasta que el cliente responde allow/deny 📝 — el harness como librería embebible (mismo patrón que el Claude Agent SDK: "el harness de Claude Code expuesto como SDK").

---

## 3. Comparativa de harnesses

| | Loop | Contexto | Memoria/instrucciones | Fallos/recovery | Seguridad |
|---|---|---|---|---|---|
| **Claude Code** | Agent SDK loop (gather→act→verify) | Compaction automática + tool-result clearing | CLAUDE.md up-front + JIT retrieval + memory tool | Retry API; compaction al acercarse al límite | allow/ask/deny fail-closed + sandbox Seatbelt/bubblewrap + proxy de red (−84% prompts) |
| **Codex CLI** | State machine en Rust, core compartido entre CLI/web/IDE | `/responses/compact` server-side (`encrypted_content` opaco); caching por prefijo estático | AGENTS.md jerárquico (estándar de facto) | Errores de tool → modelo reintenta | 3 approval modes, red off por defecto, sandbox nativo por OS, Guardian LLM |
| **Gemini CLI** | Core package: prompt assembly + tool orchestration | Compresión automática cerca del límite; fallback a flash en rate limit | GEMINI.md jerárquico + imports `@./` | **Shadow git repo** con snapshot pre-edición + `/restore` | 4 approval modes, allowlist de shell, 5 métodos de sandbox |
| **Aider** | Sin tools LLM-callable: edición por diff textual | **Repo map** (tree-sitter + graph ranking, budget dinámico) | — | **Git como red de seguridad**: commit por edición + `/undo` | No documenta sandbox; git es el safety net |
| **SWE-agent** | `forward()` → acción en shell (SWE-ReX) | `HistoryProcessor` comprime pre-query | Templates YAML deterministas + state command | — (superseded por mini-swe-agent) | Aislamiento por contenedor Docker/remoto |
| **Hermes (Nous)** | Sesión → `AIAgent` cacheado en LRU (preserva prompt cache) → `run_conversation()` | Micro-compaction opt-in (protege prompts del usuario verbatim) | Memoria curada por el agente + FTS5 + Context Files + skills (agentskills.io) | **Crash recovery**: `.clean_shutdown`, `resume_pending` <120s, suspend tras 3 restarts | Command approval, DM pairing, 7 terminal backends (local→Docker→Modal...) |
| **ZeroClaw** 🔍 | `run_tool_call_loop`, max 10 iter default, budget compartido con subagentes, loop detector | `ContextCompressor` (persiste summaries a memoria) + trim preventivo + history pruner | Bootstrap files (AGENTS/SOUL/TOOLS/IDENTITY/USER.md) + MEMORY.md + memoria híbrida vector+BM25 con REST API | `ReliableProvider` 3 niveles, key rotation, salida elegante al agotar iteraciones, stall watchdog | Sandbox multi-backend, IAM deny-by-default, autonomy levels, prompt guard, e-stop |
| **OpenClaw** | En el npm (gateway maneja el loop); config declarativa JSON5 | `compaction.mode: safeguard` | Workspace persistente + skills 3 capas + HEARTBEAT.md | `suppressToolErrors`, session reset por idle, crons nativos para tareas críticas | Slack allowlists, `tools.exec.security`, gateway token |

Nota diferencial: casi todos los harnesses de la tabla son *coding agents* interactivos; ZeroClaw/OpenClaw/Hermes son *harnesses de canal* (Slack/WhatsApp/gateway) always-on — por eso invierten más en crash recovery, sesiones por peer, heartbeats y multi-canal, y menos en checkpointing de filesystem.

---

## 4. Cómo construir un harness desde cero

### 4.1 El esqueleto mínimo (nivel 0)

Tres building blocks: **el modelo, el loop de tool-use, y suficientes tokens**. Con 3 tools (read_file, list_files, edit_file por string replacement) ya edita código de forma autónoma; <400 líneas. 📝 [ampcode] Esto valida la arquitectura; nada de lo demás es prerequisito para empezar.

### 4.2 La escalera de madurez

Cada peldaño responde a un fallo que aparece en producción:

1. **Loop + tools + stopping conditions** — el mínimo viable. Errores de tool → de vuelta al modelo.
2. **Resiliencia de API** — retry/backoff, clasificación de errores, fallback de modelo/provider. Primer fallo real que verás en producción.
3. **Permisos** — deny-by-default, allowlists, aprobación humana para lo destructivo. Fail-closed: lo no reconocido pide aprobación.
4. **Context management** — primero mecánico (clearing/masking de tool results viejos: −52% costo sin degradar 📄), después compaction con summarization. Persiste los summaries en memoria antes de descartar (patrón ZeroClaw) para mitigar la pérdida de detalle (0/3 en el test de Anthropic).
5. **Memoria e instrucciones** — AGENTS.md jerárquico + memoria persistente con retrieval JIT. Lo crítico vive en archivos, no en el historial.
6. **Checkpoints** — git automático (Aider) o shadow repo (Gemini) antes de ediciones. Deshacer > prevenir.
7. **Subagentes** — cuando el context rot pega: aislar la exploración, retornar destilados. Presupuesto compartido para acotar el fan-out.
8. **Sandbox** — primitivas del OS (Seatbelt/bubblewrap) + proxy de red. Reduce la approval fatigue (−84%) y acota el blast radius de prompt injection.
9. **Long-running structure** — initializer/coder separados, feature list con estado `failing`, generador ≠ evaluador.

### 4.3 Decisiones de diseño clave

| Decisión | Trade-off |
|---|---|
| **Framework/SDK vs API directa** | El SDK regala compaction/tools/permisos; la API directa da control total del contexto y del caching. (Ojo: el argumento "Anthropic recomienda APIs directas" como soporte para harness propio fue **refutado 0-3** en la verificación — no usar esa cita.) |
| **Tools nativas vs texto** | Native tool-calls: parsing robusto, dependes del provider. Tags en texto (ZeroClaw `<tool_call>`): portable a cualquier modelo, parsing propio. ZeroClaw soporta ambos 🔍. |
| **Compaction client vs server-side** | Server-side (Anthropic beta / Codex): menos código, resumen opaco. Client-side: controlas qué sobrevive. |
| **Masking vs summarization** | Masking: barato, reversible (re-fetch), no degrada. Summarization: comprime más, pero +15% trayectorias y pérdida de detalle. Híbrido gana en el estudio de JetBrains. 📄 |
| **Subagentes in-process vs cross-CLI (PTY)** | In-process: visibilidad y latencia. PTY sobre harnesses ajenos: reusas capacidades completas, pagas parsing frágil y opacidad. 📄 |
| **Fail-open vs fail-closed** | En permisos: fail-closed siempre (Claude Code). En sandbox: Claude Code es fail-open (warn + continue sin bubblewrap), Codex fail-closed. Para agentes autónomos sin humano en el loop: fail-closed. |
| **Sesión continua vs context resets** | Sin resolver en la literatura verificada: los claims sobre "context anxiety" y su mitigación en modelos nuevos fueron refutados (1-2) — no hay versión estable de ese trade-off. Decide empíricamente. |
| **Prompt cache como restricción de diseño** | Prefijo estático al frente, append-only. Cualquier feature que mute el prefijo mid-session (reordenar tools, editar system prompt) tiene un costo oculto real. |

### 4.4 Checklist de producción (lo que la literatura + nuestros harnesses dicen que no puede faltar)

- [ ] Stopping conditions múltiples: max iterations + loop detection + timeout + cancelación cooperativa
- [ ] Salida degradada elegante al agotar presupuesto (resumen final, no error)
- [ ] Clasificación de errores de API (retryable vs no; context overflow = recuperable)
- [ ] Truncado de tool results antes de anexarlos al historial
- [ ] Prefijo de prompt estable (caching) y assembly determinista
- [ ] Transcript canónico persistente (SQLite/jsonl) + recovery al boot
- [ ] Permisos deny-by-default con matching fail-closed
- [ ] Lo crítico en archivos/memoria, nunca solo en el historial (la compaction lo va a perder)
- [ ] Verificación separada de la generación (evaluador ≠ generador)
- [ ] Observabilidad day-1: cada tool call, cada retry, cada compaction traceada

---

## 5. Fuentes

**Verificadas adversarialmente (workflow deep-research, 3 votos por claim)**:
- Anthropic Engineering: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) · [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) · [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) · [Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- Anthropic docs/cookbook: [Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) · [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing) · [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) · [Context engineering cookbook](https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools)
- [SWE-agent: Agent-Computer Interfaces (arXiv 2405.15793, NeurIPS 2024)](https://arxiv.org/abs/2405.15793)

**Primarias verificadas (sin pasada adversarial)**:
- [Taxonomía de coding-agent scaffolds (arXiv 2604.03515)](https://arxiv.org/pdf/2604.03515) · [Claude Code security](https://code.claude.com/docs/en/security) · [permissions](https://code.claude.com/docs/en/permissions) · [sandboxing](https://code.claude.com/docs/en/sandboxing) · [Codex approvals/security](https://learn.chatgpt.com/docs/agent-approvals-security) · [Codex AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) · [agents.md](https://agents.md/) · [Gemini CLI docs](https://www.geminicli.com/docs/) (sandbox, checkpointing, GEMINI.md) · [Aider repomap](https://aider.chat/docs/repomap.html) · [git integration](https://aider.chat/docs/git.html) · [edit formats](https://aider.chat/docs/more/edit-formats.html) · [SWE-agent docs (ACI, architecture, tools)](https://github.com/SWE-agent/SWE-agent) · [Hermes agent README + docs](https://github.com/NousResearch/hermes-agent) (session-lifecycle, micro-compaction) · [JetBrains Research: Efficient context management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

**Secundarias/blogs**:
- [How Codex is built (Pragmatic Engineer)](https://newsletter.pragmaticengineer.com/p/how-codex-is-built) · [Codex architecture (swequiz)](https://www.swequiz.com/articles/openai-codex-architecture) · [How to build an agent (ampcode)](https://ampcode.com/notes/how-to-build-an-agent) · [Sandboxing comparison (Koukyosyumei)](https://medium.com/@Koukyosyumei/how-claude-code-and-codex-sandbox-untrusted-code-ba39b493046a) · [Aider architecture (simranchawla)](https://simranchawla.com/understanding-ai-coding-agents-through-aiders-architecture/)

**Código de producción** 🔍: ZeroClaw (runtime open source que operamos) y una plataforma multi-agente corporativa construida sobre OpenClaw — las referencias internas (rutas, file:line, configs) se omiten en esta edición pública.

**Claims refutados en verificación (NO citar)**: (1) "Anthropic recomienda APIs directas sobre frameworks" como argumento pro-harness-propio (0-3); (2) caracterizaciones específicas de "context anxiety" y su eliminación en modelos nuevos (1-2); (3) sensibilidad temporal: los beta headers y version strings (`compact_20260112`, `clear_tool_uses_20250919`, `memory_20250818`) cambian — verificar docs vivas antes de implementar.
