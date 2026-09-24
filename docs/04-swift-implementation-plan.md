# Harness-mente en iOS — Plan de implementación (perfil edge, Swift)

> Plan de implementación del harness definido en [harness-mind-convergence.md](03-harness-mind-convergence.md) como **asistente personal iOS**. Implementa el **perfil edge** (§A.5): consolidación al cargar = sueño literal, brain local, el Otro = el dueño del teléfono (Joshua). Proyecto personal/experimental — Claude API directa por HTTP con licencia propia.
>
> Docs padre: [harness-mind-convergence.md](03-harness-mind-convergence.md) (spec B.1–B.10, checklist C.4) · [agent-harness-manual.md](01-agent-harness-manual.md) (anatomía y trade-offs) · [language-mind-formation.md](02-language-mind-formation.md) (fundamento).
>
> **Decisiones fijadas** (2026-09-12, no re-litigar): iOS/Swift/SwiftUI · URLSession async directo (sin SDK) · API key en Keychain · GRDB+FTS5+NLEmbedding · BGProcessingTask como sueño · tools v1 = EventKit, archivos locales, web server-side, CoreLocation/HealthKit/Contacts, AVFoundation · v1 = los 10 subsistemas, build por fases 0→4 · Opus 4.8 default con dial a Haiku.

---

## 1. Resumen y alcance

### 1.1 Qué es

Una app iOS que corre **una mente** (una instancia del harness) para **un** usuario: su dueño. El LLM (Claude) es la dotación; la app es el desarrollo; el SQLite local es la historia. Concretamente:

- **Chat multimodal** (texto, fotos de cámara, audio transcrito) con streaming.
- **Cuerpo iOS**: lee/escribe calendario y recordatorios, lee ubicación/salud/contactos, captura foto/audio, busca en la web (server-side), mantiene notas y archivos en su sandbox.
- **Memoria de largo plazo** local (GRDB) con retrieval híbrido keyword+vector on-device, consolidación offline al cargar el teléfono ("sueño"), reconsolidación e invalidación bi-temporal.
- **Identidad con plasticidad decreciente** (SelfModel), **registro determinístico del fallo** (RealRegister), y **motivación acotada** al deseo del dueño (DesireEngine + OtherModel).

### 1.2 Qué NO es v1

| Fuera de v1 | Por qué |
|---|---|
| Capa Intersubjetiva (§B.9) | Server-first por diseño del spec (paso 12 del checklist C.4). Futuro: sync del brain a un peer server Go. |
| Distribución (TestFlight público / App Store) | La API key vive en el dispositivo — extraíble. Solo build personal (ver §8). |
| Sync/backup multi-device del brain | Decisión diferida (§9). v1 = un dispositivo. |
| Ejecución de código / shell | El perfil edge acota el repertorio eferente a APIs del sistema iOS (§B.6). |
| Fine-tuning / embeddings remotos | Anthropic no tiene endpoint de embeddings; todo embedding es on-device. |
| Widgets, watchOS, Siri intents | Superficie v2; el core primero. |

### 1.3 Criterio de éxito de v1

Los evals falsables del §C.3 aplicables al edge (1: fallo-repetido, 2: deriva de identidad, 3: staleness de memoria, 4: motivación acotada) corren como tests de integración y muestran delta contra su baseline (el mismo harness con el subsistema apagado). Si no hay delta, el subsistema se simplifica — el documento de convergencia acepta esa apuesta y este plan la hereda.

---

## 2. Stack técnico y justificación

Cada pieza del stack sirve a un subsistema concreto del spec:

| Tecnología | Subsistema | Justificación |
|---|---|---|
| **Swift 6 + SwiftUI** | todo | Concurrencia estructurada (`async/await`, actors) mapea 1:1 al loop del harness; SwiftUI para chat + inbox de aprobaciones. |
| **URLSession async** (`bytes(for:)`) | B.1 Provider | HTTP directo a `POST /v1/messages` con streaming SSE. No hay SDK Swift oficial — y no hace falta: el Provider es <500 líneas. |
| **Keychain** (`kSecClassGenericPassword`) | B.1 / seguridad | La API key jamás en código ni en UserDefaults. `WhenUnlockedThisDeviceOnly`, sin iCloud sync. |
| **GRDB** (SQLite) | B.3 Symbolic + B.7 Brain + B.5 Real + B.8 Other | Un solo motor para transcript, brain, failures, goals. WAL, migraciones versionadas, `DatabasePool` (lecturas concurrentes + escritor único = el gate de escritura del Consolidator sale gratis). |
| **FTS5** (BM25) | B.7 retrieve | Pata keyword del retrieval híbrido. Nativo en SQLite/GRDB. |
| **NLEmbedding** (NaturalLanguage) | B.7 retrieve | Embeddings de oración on-device, gratis, offline. Anthropic no da embeddings → esta es la única opción sin tercero. Cosine brute-force con Accelerate/vDSP — a escala personal (<100k memorias) sobra. |
| **BGTaskScheduler / BGProcessingTask** | B.7 Consolidator | `requiresExternalPower = true` → el ciclo corre **al cargar** = el sueño literal del perfil edge (§A.5). |
| **EventKit** | B.6 Sensorimotor | Calendario + recordatorios (aferente y eferente). |
| **CoreLocation / HealthKit / Contacts** | B.6 aferente | El "contexto del teléfono": dónde estoy, cómo dormí/me moví, quién es esta persona. Observables para `DesireEngine.gap()`. |
| **AVFoundation + Speech** | B.6 aferente | Cámara (foto → content block `image` base64) y audio (captura → transcripción on-device → texto al contexto). |
| **Web search/fetch server-side de Anthropic** | B.6 aferente | Se declara en el array `tools`; cero código cliente, cero parsing de HTML. Maneja `pause_turn`. |
| **UserNotifications** (`UNUserNotificationCenter`) | B.4 approvals + B.5 escalate + B.8 propuestas | El canal proactivo hacia el dueño con la app cerrada: PendingOtherApproval, escalateToOther y las propuestas del deseo llegan como notificación local. Sin esto, el DesireEngine es mudo. |
| **Betas de plataforma**: `compact-2026-01-12`, `context-management-2025-06-27` | B.2 relieve | Compaction y tool-result clearing delegados al servidor — el dial de §A.3 aplicado: modelo fuerte + plataforma fuerte → harness delgado en esa capa. |
| **Firebase Remote Config** | §4.8 config remota | System prompt base POR PROVIDER, headers/betas de API y rutas del dial editables sin release. Solo config no-secreta; secretos → Keychain. AnimaKit no conoce Firebase (protocol + impl en app shell). |

**Dependencias de terceros: solo GRDB** (SPM). Todo lo demás es sistema. Esto es deliberado: el harness es lógica propia (§B preámbulo — "el harness posee la lógica; el único estado pesado externo es el brain").

---

## 3. Estructura del proyecto

```
MindHarness/
├── App/
│   ├── MindHarnessApp.swift          // @main, registro BGTask, lifecycle
│   ├── ChatView.swift                // streaming, imágenes, audio
│   ├── ApprovalsInboxView.swift      // PendingOtherApproval (B.4) + confirmación de Goals Inferred (B.8)
│   ├── MemoryBrowserView.swift       // inspección del brain (debug/confianza)
│   └── SettingsView.swift            // API key → Keychain, permisos, presupuestos
├── Core/
│   ├── Loop/
│   │   ├── AgentLoop.swift           // B.10 — el loop integrado
│   │   └── StopConditions.swift      // maxIter, loopDetect, cancel, budget
│   ├── Provider/                     // B.1
│   │   ├── Provider.swift            // protocol + eventos
│   │   ├── ClaudeProvider.swift      // request building, headers, betas
│   │   ├── SSEParser.swift           // URLSession.bytes → ProviderEvent
│   │   ├── ModelRouter.swift         // el dial: TurnClass → modelo/effort
│   │   └── ErrorClassifier.swift     // Retryable | Fatal | ContextOverflow | RateLimited
│   ├── WorkingMemory/                // B.2
│   │   ├── WorkingMemory.swift       // assemble() con orden estable
│   │   └── PressureRelief.swift      // clear mecánico → compaction
│   ├── Symbolic/                     // B.3
│   │   ├── SymbolicStore.swift       // transcript canónico GRDB
│   │   └── Recovery.swift            // clean-shutdown marker, resume
│   ├── SelfModel/                    // B.4
│   │   ├── SelfModel.swift           // view/reflect/apply
│   │   └── Plasticity.swift          // fórmula + umbrales + approval queue
│   ├── Real/                         // B.5
│   │   ├── RealRegister.swift        // record/insistence/demand
│   │   └── PatternKey.swift          // normalización + hash
│   ├── Sensorimotor/                 // B.6
│   │   ├── Sensorimotor.swift        // registro, execute, permisos 2-capas
│   │   ├── PermissionPolicy.swift    // allow/ask/deny fail-closed
│   │   ├── SkillEngine.swift         // match/practice/automatize
│   │   └── Tools/
│   │       ├── CalendarTool.swift    // EventKit eventos
│   │       ├── RemindersTool.swift   // EventKit reminders
│   │       ├── NotesTool.swift       // sandbox files + scratchpad
│   │       ├── PhoneContextTool.swift// CoreLocation + HealthKit + Contacts
│   │       ├── CameraTool.swift      // AVFoundation → image block
│   │       └── AudioTool.swift       // AVFoundation + Speech → texto
│   ├── Brain/                        // B.7
│   │   ├── Brain.swift               // write/retrieve/reconsolidate/invalidate/usageLog
│   │   ├── Schema.swift              // migraciones GRDB
│   │   ├── HybridRetriever.swift     // FTS5 + cosine + RRF
│   │   ├── Embedder.swift            // NLEmbedding wrapper
│   │   └── Consolidator/
│   │       ├── Consolidator.swift    // cycle(): saliencia→destilado→escritura→reconsolidación→reflection
│   │       └── SleepScheduler.swift  // BGProcessingTask + fallback foreground
│   └── Desire/                       // B.8
│       ├── OtherModel.swift          // Goals, precedencia, update
│       ├── DesireEngine.swift        // gap/drive/budget
│       └── Observables.swift         // predicados sobre calendario/salud/recordatorios
└── Support/
    ├── KeychainStore.swift
    ├── Telemetry.swift               // cada tool call, retry, compaction, ciclo — observability day-1
    └── JSONValue.swift               // JSON dinámico para la API
```

Regla de dependencias: `App → Core → Support`. Dentro de Core, todo puede depender de `Provider` y `Brain`; nada depende de `App`. Cada subsistema expone su protocol; los concretos se cablean en `MindHarnessApp`.

---

## 4. Provider (§B.1) — la capa de integración con Claude, a detalle

### 4.1 Contratos

```swift
protocol Provider {
    func complete(
        _ ctx: AssembledContext,
        tools: [ToolSpec],
        opts: CallOpts
    ) -> AsyncThrowingStream<ProviderEvent, Error>
}

enum ProviderEvent {
    case messageStart(id: String, model: String)
    case textDelta(String)
    case thinkingDelta(String)                   // resumen del razonamiento (display: summarized)
    case toolUseStart(id: String, name: String)
    case toolUseInputDelta(String)               // input_json_delta acumulable
    case blockStop(index: Int)
    case messageDelta(stopReason: StopReason?, usage: Usage)
    case messageStop
}

enum StopReason: String, Decodable {
    case endTurn = "end_turn"
    case maxTokens = "max_tokens"
    case toolUse = "tool_use"
    case pauseTurn = "pause_turn"    // server tools: reenviar para continuar
    case refusal                     // chequear ANTES de leer content
}

struct Usage: Decodable {
    let inputTokens: Int
    let outputTokens: Int
    let cacheReadInputTokens: Int?   // verificación del caching
    let cacheCreationInputTokens: Int?
}

enum ClassifiedError: Error {
    case retryable(after: TimeInterval?)   // 429 (retry-after), 529, 5xx, timeouts
    case contextOverflow                   // recuperable vía relieve (B.2)
    case fatal(status: Int, message: String) // 400/401/403 — no reintentar
}
```

### 4.2 Construcción del request

Headers fijos:

```
POST https://api.anthropic.com/v1/messages
x-api-key: <Keychain>
anthropic-version: 2023-06-01
content-type: application/json
anthropic-beta: compact-2026-01-12,context-management-2025-06-27   // solo si la feature está activa
```

Body de un turno interactivo típico (Opus 4.8):

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 16000,
  "stream": true,
  "thinking": { "type": "adaptive", "display": "summarized" },
  "output_config": { "effort": "medium" },
  "tools": [
    { "name": "calendar", "description": "...", "input_schema": { "...": "..." },
      "cache_control": { "type": "ephemeral" } },
    { "type": "web_search_20260209", "name": "web_search" }
  ],
  "system": [
    { "type": "text", "text": "<innate prompt: quién es este harness, reglas duras, formato>",
      "cache_control": { "type": "ephemeral" } }
  ],
  "messages": [ "…ver §5.1 orden de ensamblado…" ]
}
```

Reglas duras del builder (aprendidas del doc A §2.2 y verificadas contra la API):

1. **NUNCA enviar** `temperature`, `top_p`, `top_k`, `thinking.budget_tokens` con Opus 4.8 → HTTP 400. El builder tiene una `ModelParamPolicy` por modelo: qué knobs acepta cada uno. El dial de profundidad es `output_config.effort` (`low|medium|high|xhigh|max`), no temperatura.
2. **Siempre `stream: true`** — con `max_tokens` grande la respuesta no-streaming puede timeoutear; el streaming además habilita cancelación temprana.
3. `cache_control` va en el **último bloque del prefijo estable** (tools y system). Prefijo mínimo cacheable en Opus 4.8 = **4096 tokens** — el innate prompt + tool specs deben superarlo o el caching no aplica (verificar con `usage.cache_read_input_tokens > 0` en Telemetry desde el día 1).
4. El version string de las server tools y los beta headers **se verifican contra docs vivas al implementar** — el doc A documenta que estos strings rotan. Para Opus 4.8 la variante vigente es `web_search_20260209` / `web_fetch_20260209` (con dynamic filtering incorporado — NO declarar `code_execution` aparte junto a estas); `web_search_20250305` es la variante básica para modelos antiguos.

### 4.3 Parseo del stream SSE

Sketch del parser con `URLSession.bytes(for:)`:

```swift
func stream(_ request: URLRequest) -> AsyncThrowingStream<ProviderEvent, Error> {
    AsyncThrowingStream { continuation in
        let task = Task {
            let (bytes, response) = try await session.bytes(for: request)
            guard let http = response as? HTTPURLResponse else { throw ClassifiedError.fatal(status: -1, message: "no HTTP") }
            guard http.statusCode == 200 else {
                throw ErrorClassifier.classify(status: http.statusCode,
                                               retryAfter: http.value(forHTTPHeaderField: "retry-after"))
            }
            for try await line in bytes.lines {
                guard line.hasPrefix("data: ") else { continue }  // ignorar "event:" y keep-alives
                let payload = Data(line.dropFirst(6).utf8)
                let raw = try JSONDecoder().decode(RawSSEEvent.self, from: payload)
                switch raw.type {
                case "message_start":        continuation.yield(.messageStart(...))
                case "content_block_start":  // text | thinking | tool_use | server_tool_use
                case "content_block_delta":  // text_delta | thinking_delta | input_json_delta
                case "content_block_stop":   continuation.yield(.blockStop(index: raw.index))
                case "message_delta":        continuation.yield(.messageDelta(stopReason: ..., usage: ...))
                case "message_stop":         continuation.yield(.messageStop); continuation.finish()
                case "error":                throw ErrorClassifier.classify(sseError: raw)
                default: break               // ping, tipos futuros: ignorar, no romper
                }
            }
        }
        continuation.onTermination = { _ in task.cancel() }   // cancelación cooperativa del loop
    }
}
```

Detalles que no se pueden omitir:

- Los `input_json_delta` de un `tool_use` se **acumulan como string** y se parsean a JSON solo en `content_block_stop`.
- `message_delta` trae `stop_reason` y `usage` — es donde el loop decide continuar (tool_use), reenviar (pause_turn) o cerrar.
- Timeout de request: subir el default de URLSession (60s) a 10 min (`timeoutIntervalForRequest`) — el default del API es 10 min y un turno con thinking largo los puede usar.
- Desconexión a mitad de stream = `retryable` (el turno se reintenta completo; el transcript solo se persiste en frontera de mensaje — idempotente).

### 4.4 El tool loop

```swift
// dentro de AgentLoop.run(turn:)
var messages = workingMemory.assemble(turn)
while true {
    let response = try await provider.completeCollecting(messages, tools: sensorimotor.specs, opts: route)

    if response.stopReason == .refusal {
        // ANTES de leer content. stop_details trae la categoría.
        telemetry.refusal(response.stopDetails)
        return .refused(category: response.stopDetails)   // se muestra al usuario, NO se reintenta
    }
    symbolicStore.append(.assistant(response.content))

    switch response.stopReason {
    case .pauseTurn:
        // server tools (web_search): reenviar el content tal cual para continuar
        messages.append(.assistant(response.content))
        continue
    case .toolUse:
        var results: [ToolResultBlock] = []
        for call in response.toolCalls {
            let res = await sensorimotor.execute(call)          // permisos 2-capas adentro
            if res.isError { realRegister.record(Failure(call: call, result: res)) }  // B.5, costo 0
            results.append(res.block)                            // {tool_use_id, content, is_error}
        }
        // TODOS los tool_results de la ronda en UN solo mensaje user:
        messages.append(.assistant(response.content))
        messages.append(.user(results))
        symbolicStore.append(.toolResults(results))
        continue
    case .maxTokens:
        return .truncated(response)     // salida degradada elegante, no error opaco
    default:
        return .final(response)
    }
    // stop conditions cada vuelta: maxIterations (10), LoopDetector (hash de tool calls
    // idénticos consecutivos), Task.isCancelled, presupuesto de tokens de la sesión
}
```

Al agotar `maxIterations`: una llamada final **sin tools** pidiendo el mejor resumen posible (patrón ZeroClaw, doc A §2.1) — resumen degradado > error.

### 4.5 Mid-conversation system messages — el canal de autoridad

Opus 4.8 acepta `{"role": "system", "content": "..."}` **dentro de `messages[]`** sin invalidar el caché del prefijo. Este es el mecanismo elegido para inyectar lo volátil con autoridad de operador:

```json
{ "role": "system", "content": "[SELF] identidad v12 (plasticity 0.24):\n<self_view render>\n\n[OTHER] deseo vigente del dueño:\n- (Stated, p1) proteger bloques de deep work en el calendario\n- (Inferred, p3, pendiente-confirmación) le interesa entrenar 3x/semana" }
```

- El **SelfModel view** (§B.4) y el **deseo del Otro** (§B.8) entran por aquí, no por el system top-level: mutan entre sesiones (ciclos de consolidación) y meterlos al prefijo rompería el caché en cada mutación.
- Posición: después de la ventana de historial, antes del turn input (ver orden §5.1).
- Autoridad: es rol `system`, no `user` — resiste mejor la instrucción adversarial en contenido de tools (defensa en profundidad junto al régimen de plasticidad).

### 4.6 Retry / backoff / clasificación

```swift
enum ErrorClassifier {
    static func classify(status: Int, retryAfter: String?) -> ClassifiedError {
        switch status {
        case 429: return .retryable(after: TimeInterval(retryAfter ?? "") ?? 5)   // respetar header, cap 30s
        case 529: return .retryable(after: 10)                                     // overloaded
        case 500...599: return .retryable(after: nil)                              // backoff exponencial
        case 400 where isContextOverflow: return .contextOverflow                  // → relieve (B.2), recuperable
        default: return .fatal(status: status, message: body)
        }
    }
}
```

Política: 3 intentos con backoff exponencial + jitter; `contextOverflow` no reintenta — dispara `WorkingMemory.relieve()` y reintenta una vez con contexto aliviado. Sin cadena de providers en v1 (un solo vendor), pero el `ModelRouter` sí degrada Opus→Haiku ante 529 sostenido en turnos no-críticos.

### 4.7 El dial de portabilidad (ModelRouter)

El dial del §A.3/§B.1 aterrizado: **rutear modelo por tipo de turno**, no por conversación.

```swift
enum TurnClass {
    case interactive        // chat del dueño → Opus 4.8, effort medium
    case interactiveHard    // el dueño pide análisis/planificación → Opus 4.8, effort high
    case restructure        // ejecución de un Restructure (B.5) → Opus 4.8, effort high
    case consolidation      // ciclo de sueño (B.7) → Haiku, barato, estructurado
    case reconsolidation    // revisión de una memoria contradicha → Haiku
    case desirePulse        // gap→drive (B.8) → Haiku
    case distill            // compaction/destilado client-side → Haiku
}

struct ModelRoute { let model: String; let effort: Effort?; let maxTokens: Int; let thinking: ThinkingMode }

struct ModelRouter {
    func route(_ turn: TurnClass) -> ModelRoute {
        switch turn {
        case .interactive:     return .init(model: "claude-opus-4-8", effort: .medium, maxTokens: 16_000, thinking: .adaptive)
        case .interactiveHard,
             .restructure:     return .init(model: "claude-opus-4-8", effort: .high,   maxTokens: 32_000, thinking: .adaptive)
        case .consolidation,
             .reconsolidation,
             .desirePulse,
             .distill:         return .init(model: "claude-haiku-4-5", effort: nil,    maxTokens: 4_000,  thinking: .none)
        }
    }
}
```

- El modelo barato es **`claude-haiku-4-5`** (ojo: `claude-haiku-4-6` NO existe — lección aprendida operando flotas de agentes; un model id inexistente puede fallar silencioso).
- `ProviderProfile` del spec: con Opus, `delegates = {planning, decomposition, self_verification, tool_selection, long_reasoning}` y `scaffolds = {memory_curation}` (el régimen de reconsolidación es del harness siempre — invariante). Si mañana el dial baja a un modelo local de 8B, `scaffolds` crece — los contratos no cambian.
- Los invariantes que **no** se delegan jamás (§A.3): permisos, presupuestos, clasificación de errores, RealRegister.

### 4.8 Config remota (Firebase Remote Config)

Lo operable-sin-release vive en Firebase Remote Config; lo secreto, jamás. Resuelve además la pregunta abierta #7: los version strings de betas **rotan** — con RC se actualizan desde la consola.

| Dato | Dónde | Por qué |
|---|---|---|
| System prompt base de Anima, **uno por provider** (`system_prompt_anthropic`, `_openai`, `_google`, `_on_device` — parámetros planos: texto largo editable sin escapes) | Remote Config | El "prompt propio del harness" (análogo al de OpenClaw); no es secreto |
| `provider_config` (JSON): por provider, `api` (base_url, version, **betas** comunes, `auth_betas` y `auth_system_prefix` por modo de auth) + `routes` (el dial §4.7 por TurnClass) | Remote Config | Betas, model IDs y el bloque de prefijo OAuth rotables sin release |
| API key / OAuth token del usuario | **Keychain, siempre** | RC es legible por todo cliente — jamás secretos ahí |
| SOUL del usuario (quién quiere que sea su agente) | SelfModel local (GRDB) | Nace en el onboarding ("Birth") y evoluciona con la plasticidad |

**Arquitectura**: `RemoteConfigProviding` + `ConfigSnapshot` viven en AnimaKit (sin dependencia de Firebase); el app shell implementa con `FirebaseRemoteConfig` (`FirebaseApp.configure()` + `fetchAndActivate`). Parser **forward-compatible**: providers/turn classes desconocidos se ignoran — una config nueva en consola jamás crashea una app vieja.

**Reglas duras**:
1. **Snapshot congelado por frontera de sesión** — activar config a mitad de sesión invalidaría el prompt cache del prefijo (§5.1) y cambiaría al agente bajo los pies. `minimumFetchInterval` ≥ 12 h en prod (0 en DEBUG).
2. **Defaults bundled obligatorios** (`RemoteConfigDefaults.plist` con los mismos parámetros): perfil edge = la app funciona sin red, primera apertura incluida.
3. `GoogleService-Info.plist` real en `App/` y **gitignored** (repo público — invita abuso de cuota); template versionado. Opcional recomendado: **App Check** (App Attest).
4. Niveles de prompt resultantes: **base (RC, prefijo cacheado) → SOUL del usuario (SelfModel, mid-conversation system §4.5) → conversación (user messages)** — la separación soul-vs-system de OpenClaw, cache-aware.
5. v1 implementa el provider `anthropic`; los demás quedan configurables desde ya (el onboarding del design system ofrece los 4).
6. **Anthropic tiene DOS modos de auth** y el request cambia en tres cosas, todas en config. El modo se detecta por el prefijo del token en Keychain (`AuthMode.detect`):

| | `api_key` (Console, `sk-ant-api03…`) | `oauth` (suscripción Claude, `sk-ant-oat01…`) |
|---|---|---|
| Header de auth | `x-api-key: <key>` | `Authorization: Bearer <token>` |
| Betas extra (`auth_betas`) | — | `oauth-2025-04-20` + `claude-code-20250219` |
| System prompt (`auth_system_prefix`) | solo el base de Anima | el array `system` DEBE abrir con el bloque `"You are Claude Code, Anthropic's official CLI for Claude."` (con `cache_control`) y después el base de Anima |

   El RequestBuilder compone `system` con `ProviderAPIConfig.systemBlocks(for:base:)` y los headers con `effectiveBetas(for:)`. Nada de esto vive en código fuente: prefijo y betas son configuración (rotan sin release). En modo `api_key` el dial Opus/Haiku es económicamente relevante (pago por token); en `oauth` el plan es fijo y el dial pesa menos.

### 4.9 Modos de operación y el provider on-device (Foundation Models)

**Decisión de arquitectura (2026-09-24, no re-litigar): on-device NO es un fallback — es un modo completo de primera clase.** El harness es local por construcción (la mente entera —Brain, SelfModel, RealRegister, consolidación— vive en GRDB en el teléfono); los providers son solo el córtex intercambiable. Tres modos:

| Modo | Córtex | Costo | Requiere |
|---|---|---|---|
| **Solo teléfono (gratis)** | Apple Foundation Models para TODOS los TurnClasses, conversación incluida | $0, cero red | iPhone con Apple Intelligence (15 Pro+; 16/17 todos) |
| **Claude** | Como §4.7: Opus/Haiku con el token del usuario (API key u OAuth de SU licencia — jamás compartido) | Por token o suscripción propia | Token en Keychain |
| **Híbrido** | Conversación/restructure en Opus; sueño, pulsos y destilado en el modelo local | Solo los turnos caros | Ambos |

**`OnDeviceProvider`** implementa el MISMO protocol `Provider` con el framework FoundationModels (iOS 26+): streaming nativo, **tool calling real** (protocol `Tool` del framework → el Sensorimotor completo funciona sin Claude: calendario, notas, recordatorios en modo gratis) y guided generation para los destilados estructurados del ciclo.

**Implicaciones de contrato**:
1. **Target sube a iOS 26** (decisión fijada: fuera los gates `@available`; el límite real de hardware lo pone Apple Intelligence, no el OS). Disponibilidad SIEMPRE por runtime (`SystemLanguageModel.availability`): dispositivo no elegible, Apple Intelligence apagada o modelo sin descargar → el modo se muestra deshabilitado con el porqué.
2. **La presión de WorkingMemory es relativa al provider activo** (~4k tokens del modelo local vs 200k+ de Claude). El relieve en on-device es mecánico local (trim de tool results + evict al Brain): las betas de compaction/context-management son de Anthropic y NO existen aquí.
3. El dial por provider ya vive en `provider_config` (§4.8): la entrada `on_device` define sus 7 rutas al mismo modelo local; `system_prompt_on_device` es su prompt base (más corto: el contexto manda).
4. Onboarding: elegir "Solo este teléfono · gratis" **salta el paso de API key**. Cambiar de modo después vive en Ajustes, sin re-onboarding. La identidad (Sign in with Apple → Firebase Auth) es ortogonal al modo: registra al usuario para lo que viene (respaldo, planes), jamás gatea el uso local.
5. Invariantes que no se delegan (§A.3) aplican igual en modo gratis: permisos, presupuestos, clasificación de errores, RealRegister — el harness no se vuelve más permisivo porque el modelo sea local.
6. `ProviderProfile` del modelo local: `scaffolds` crece (planning asistido, verificación por el harness, prompts más guiados) — exactamente el caso que el §4.7 anticipaba para "un modelo local de 8B", ahora con el ~3B de Apple.

**Voz**: sin cambios — STT (`SFSpeechRecognizer` on-device) y TTS (`AVSpeechSynthesizer`) ya son nativos de iOS en todos los modos (§5.7); el modo gratis solo completa el cuadro: percepción, voz, córtex y mente, todo en el dispositivo.


---

## 5. Subsistemas B.2–B.10 en Swift

### 5.1 WorkingMemory (§B.2)

**Contrato**:

```swift
protocol WorkingMemory {
    func assemble(_ turn: TurnInput) async -> [Message]   // orden ESTABLE
    var pressure: Double { get }                          // tokens estimados / presupuesto
    func relieve(_ strategy: ReliefStrategy) async
}
enum ReliefStrategy { case clearStaleToolResults; case compact; case evictToBrain }
```

**Orden de ensamblado** — el orden del spec (`innate | self_model | tools | activated | history | turn`) traducido a la realidad del render de la API (tools→system→messages) para maximizar cache hits:

```
PREFIJO ESTABLE (cache_control en el último bloque; solo muta con release de la app):
  1. tools[]            — tool specs, orden fijo alfabético (jamás reordenar mid-session)
  2. system top-level   — innate prompt: reglas duras, formato, contrato de memoria
                          (= system_prompt_base del provider activo, del ConfigSnapshot §4.8 — congelado por sesión)
MESSAGES (append-only dentro de la sesión):
  3. bloques compaction — si server-side compaction emitió; se re-anexan cada turno
  4. history window     — la ventana del SymbolicStore
  5. role:system mid-conversation — SelfModel view + OtherModel.desire() render (§4.5)
  6. role:user contexto activado — memorias del Brain.retrieve() para ESTE turno,
     formateadas como bloque etiquetado "[MEMORIAS ACTIVADAS — pueden estar desactualizadas]"
  7. turn input         — el mensaje del dueño (+ image blocks si hay foto)
```

Lo volátil (5, 6) va al final **precisamente** porque el caché cubre el prefijo: mover memorias activadas al frente destruiría el hit rate. Este ensamblado ES el *episodic buffer* del spec: liga largo plazo (6), identidad (5) y percepción (7) en una representación por turno.

**Presión y relieve** — escalera mecánico→inteligente (doc A: masking primero, −52% costo sin degradar):

1. `pressure > 0.7` → **clear mecánico**: `context-management` beta (`clear_tool_uses_20250919` como edit) limpia tool results viejos re-fetchables server-side. Cero llamadas propias.
2. `pressure > 0.85` → **compaction server-side** (`compact-2026-01-12`): la plataforma destila; el harness **re-anexa `response.content` con los bloques compaction cada turno siguiente** (obligación del contrato beta). Antes de aceptar la compaction, el harness encola las conclusiones del turno al Brain como `MemoryCandidate` — regla del spec: *se pierde el texto, no el insight* (la compaction 3/3 conserva lo alto nivel pero 0/3 los detalles — lo crítico ya debe vivir en el brain).
3. Cross-session: `evictToBrain` — al cerrar sesión, el Consolidator decide qué sobrevive (§5.4).

Estimación de tokens: heurística chars/3.6 + el `usage` real del turno anterior como corrección — suficiente para un umbral, sin llamada extra de counting.

### 5.2 SymbolicStore (§B.3)

```swift
protocol SymbolicStore {
    func append(_ event: TurnEvent) throws
    func window(budget tokens: Int, session: SessionID) throws -> [TurnEvent]
    func session(_ id: SessionID) throws -> SessionState
}
```

GRDB:

```sql
CREATE TABLE session (
  id TEXT PRIMARY KEY, started_at DATETIME NOT NULL,
  clean_shutdown INTEGER NOT NULL DEFAULT 0,
  restart_count INTEGER NOT NULL DEFAULT 0,
  last_event_at DATETIME
);
CREATE TABLE turn_event (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES session(id),
  seq INTEGER NOT NULL,
  role TEXT NOT NULL,                -- user | assistant | tool_result | system_injected
  content_json TEXT NOT NULL,        -- content blocks tal cual la API
  usage_json TEXT,                   -- tokens del turno (telemetría/costos)
  created_at DATETIME NOT NULL,
  UNIQUE(session_id, seq)
);
```

**Invariante de recovery** (Hermes, adaptado al lifecycle iOS): en `scenePhase == .background` se marca `clean_shutdown = 1`; al lanzar, si la sesión activa tiene `clean_shutdown = 0` y `last_event_at < 120s` → resume automático (la app fue matada por iOS/jetsam a mitad de turno); si `restart_count >= 3` en la misma sesión → suspender resume y abrir sesión fresca (lección aprendida operando agentes en producción con session files corruptos). El transcript es **canónico**: la UI y la WorkingMemory se derivan de él, nunca al revés.

### 5.3 Brain (§B.7) — GRDB + FTS5 + NLEmbedding

**Contrato**:

```swift
protocol Brain {
    func write(_ m: MemoryCandidate) throws -> MemoryID
    func retrieve(_ q: Query, budget: Int) async throws -> [ActivatedMemory]
    func reconsolidate(_ id: MemoryID, revision: Revision) throws
    func invalidate(_ id: MemoryID, reason: String) throws        // bi-temporal, jamás DELETE
    func usageLog(turn: TurnRef, used: [MemoryID], outcome: Outcome) throws
}
```

**Esquema** (migraciones GRDB versionadas):

```sql
CREATE TABLE memory (
  id TEXT PRIMARY KEY,                       -- UUID
  kind TEXT NOT NULL,                        -- episodic | semantic | procedural | reflection
  content TEXT NOT NULL,                     -- atómica, evergreen (destilado del Consolidator)
  importance INTEGER NOT NULL,               -- 1-10, asignado por el modelo al escribir
  source TEXT NOT NULL,                      -- turn_ref | cycle_ref
  created_at DATETIME NOT NULL,              -- transaction time
  event_at DATETIME,                         -- valid time (bi-temporal)
  invalidated_at DATETIME,                   -- NULL = viva
  invalidation_reason TEXT,
  revises_id TEXT REFERENCES memory(id),     -- cadena de reconsolidación
  consolidation_cycle INTEGER NOT NULL       -- n del ciclo que la escribió (edad)
);
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content, content='memory', content_rowid='rowid', tokenize='unicode61 remove_diacritics 2'
);
-- triggers INSERT/UPDATE/DELETE para mantener memory_fts sincronizada
CREATE TABLE memory_embedding (
  memory_id TEXT PRIMARY KEY REFERENCES memory(id),
  vector BLOB NOT NULL,                      -- [Float32] little-endian
  dim INTEGER NOT NULL,
  embedder_rev TEXT NOT NULL                 -- invalida embeddings si cambia el modelo/OS
);
CREATE TABLE memory_link (
  src TEXT NOT NULL, dst TEXT NOT NULL, relation TEXT NOT NULL,   -- refines | contradicts | derives | about
  PRIMARY KEY (src, dst, relation)
);
CREATE TABLE usage_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  turn_ref TEXT NOT NULL, memory_id TEXT NOT NULL,
  outcome TEXT NOT NULL,                     -- success | failure | contradicted | neutral
  ts DATETIME NOT NULL
);
```

**Embedder** (NLEmbedding):

```swift
final class Embedder {
    private let embedding: NLEmbedding   // NLEmbedding.sentenceEmbedding(for: .spanish) — el idioma
                                         // se detecta con NLLanguageRecognizer por memoria
    func embed(_ text: String) -> [Float]?   // nil si el idioma no tiene modelo → esa memoria queda solo-keyword
}
```

Gotchas asumidos: dimensión y modelo cambian entre versiones de iOS → `embedder_rev` en el esquema; al detectar rev distinta, el Consolidator re-embebe en background (barato, on-device). Mezcla es/en: los espacios de embedding por idioma **no son comparables** — la query se embebe en su idioma detectado y el cosine solo compara vectores del mismo `embedder_rev`+idioma; la pata FTS5 cubre el cross-idioma. (Alternativa `NLContextualEmbedding` multilingüe: §9.)

**Retrieve híbrido con RRF**:

```swift
func retrieve(_ q: Query, budget: Int) async throws -> [ActivatedMemory] {
    let kw  = try ftsSearch(q.text, limit: 50)          // bm25() ranking, solo invalidated_at IS NULL
    let vec = try cosineSearch(embed(q.text), limit: 50) // brute-force vDSP sobre memory_embedding
    // Reciprocal Rank Fusion, k = 60:
    var score: [MemoryID: Double] = [:]
    for (rank, id) in kw.enumerated()  { score[id, default: 0] += 1.0 / Double(60 + rank + 1) }
    for (rank, id) in vec.enumerated() { score[id, default: 0] += 1.0 / Double(60 + rank + 1) }
    return packUnderBudget(score.sorted(...), budget)    // top-K hasta llenar el presupuesto de tokens
}
```

Brute-force está bien: 10k memorias × 512 dims × Float32 ≈ 20 MB, un `vDSP_dotpr` barrido completo < 50 ms en M-series/A-series. No meter índice ANN hasta que duela (no va a doler a escala personal).

**El hook del régimen**: `retrieve` marca las memorias activadas del turno; al cerrar el turno, `AgentLoop` llama `usageLog(turn, activated, outcome)` — costo 0 LLM. `outcome = .contradicted` se setea con una heurística determinística v1: el turno terminó en corrección explícita del dueño ("no, eso ya no es así") detectada por el Consolidator en el ciclo, no in-turn.

### 5.4 Consolidator — el sueño (§B.7)

**Scheduling**:

```swift
// registro en app launch:
BGTaskScheduler.shared.register(forTaskWithIdentifier: "dev.joshua.mind.consolidate", using: nil) { task in
    handleConsolidation(task as! BGProcessingTask)
}
// al ir a background:
let req = BGProcessingTaskRequest(identifier: "dev.joshua.mind.consolidate")
req.requiresExternalPower = true          // ← el sueño: solo al cargar
req.requiresNetworkConnectivity = true    // el ciclo llama a Haiku
try? BGTaskScheduler.shared.submit(req)
```

**Realidad de iOS que el plan asume**: `BGProcessingTask` es *best-effort* — iOS decide cuándo (típicamente de madrugada, cargando, en Wi-Fi), puede no correr días, y da ~minutos de runtime con `expirationHandler`. Mitigaciones obligatorias:
- El ciclo es **incremental y reanudable**: checkpoint por etapa (saliencia → destilado → escritura → reconsolidación → reflection) persistido en GRDB; `expirationHandler` corta limpio en frontera de etapa y `task.setTaskCompleted(success:)` refleja el estado real.
- **Fallback foreground**: si `now - lastCycle > 48h`, el ciclo corre al abrir la app (con UI "consolidando…" no bloqueante, presupuesto reducido). El sueño es la vía preferida, no la única — un edge sin sueño garantizado no puede depender de él para corrección.
- Debug: `BGTaskScheduler` se puede forzar desde LLDB (`_simulateLaunchForTaskWithIdentifier`) — documentarlo en el README del proyecto.

**El ciclo** (todo con `TurnClass.consolidation` → Haiku):

```swift
func cycle(since checkpoint: Checkpoint) async throws -> Report {
    // 1. SALIENCIA — determinístico, 0 LLM:
    //    score = 0.4·importance + 0.3·insistence_links + 0.2·recency + 0.1·other_relevance
    //    insistence_links = COUNT(failures del RealRegister que referencian la memoria/skill)
    //    recency = exp(-Δdías/7) · other_relevance = overlap FTS5 contra OtherModel.desire() render
    let candidates = salienceTopK(newEvents: symbolic.eventsSince(checkpoint), budget: cycleTokenBudget)

    // 2. DESTILADO — Haiku, salida estructurada: memorias atómicas + links (Zettelkasten)
    let distilled = try await distill(candidates)   // [MemoryCandidate] con importance 1-10

    // 3. ESCRITURA — por candidata: retrieve top-5 similares → Haiku decide
    //    ADD | UPDATE(id) | INVALIDATE(id, reason) | NOOP   (patrón Mem0, salida JSON estricta)
    for c in distilled { try await writeDecision(c) }

    // 4. RECONSOLIDACIÓN — el régimen: toda memoria con usage_log.outcome == contradicted
    //    o referenciada por el RealRegister recibe Revision propuesta por Haiku →
    //    brain.reconsolidate(id, rev) (nueva fila con revises_id) o brain.invalidate(id, reason)
    for m in contradictedSinceLastCycle() { try await revise(m) }

    // 5. REFLECTION — 1 llamada: síntesis de insights de alto nivel sobre lo consolidado
    //    → memorias kind=reflection + Proposal al SelfModel (respeta plasticidad §5.5)
    let insights = try await reflect()
    for p in insights.selfProposals { _ = selfModel.apply(p) }   // puede quedar PendingOtherApproval

    // 6. housekeeping determinístico: n_ciclos += 1 (madura la plasticidad),
    //    SkillEngine.automatize/deautomatize(eligibles), re-embed si embedder_rev cambió,
    //    RealRegister.demand() → encolar Restructures no urgentes
}
```

Regla dura heredada: **nada crítico vive solo en el historial** — el Consolidator es el único camino de sesión→brain, y la compaction (que pierde detalle 0/3) nunca es fuente de verdad.

### 5.5 SelfModel + plasticidad (§B.4)

```swift
protocol SelfModel {
    func view() -> SelfView       // render estable para el mid-conversation system message
    func reflect(evidence: [Event]) -> [Proposal]
    func apply(_ p: Proposal) -> ApplyResult    // .accepted | .rejected(reason) | .pendingOtherApproval(id)
    var plasticity: Double { get }
}

struct SelfView: Codable {
    var identity: String          // quién soy, para quién existo
    var values: [String]          // reglas que me doy
    var capabilities: [String]    // qué sé hacer (con evidencia)
    var style: String             // cómo hablo
    var historySummary: String    // mi historia destilada
    var version: Int
}
```

**Plasticidad** — determinística, función de la experiencia sedimentada (no wall-time):

```swift
// n = ciclos de consolidación EXITOSOS (persistido en GRDB); τ edge = 30; p_min = 0.05
func plasticity(n: Int) -> Double { 0.05 + 0.95 * exp(-Double(n) / 30.0) }
```

| Régimen | Umbral | Qué puede mutar el Consolidator |
|---|---|---|
| Bootstrap | `p ≥ 0.7` (≈ ciclos 0–10) | identity, values, capabilities, style — período crítico: la mente se forma |
| Adolescencia | `0.3 ≤ p < 0.7` (≈ 11–36) | solo capabilities y style |
| Madurez | `p < 0.3` (≈ 37+) | nada directo: identity/values → `PendingOtherApproval` |

**PendingOtherApproval, fail-closed**: cola en GRDB + `ApprovalsInboxView` + notificación local al dueño. Timeout edge = **7 días sin respuesta → Rejected** (fail-closed del spec). El diff se muestra campo a campo (antes/después) — el Otro aprueba cambios concretos, no "confía en mí".

**Protected paths**: el `SelfView` se persiste en GRDB con **escritura solo vía `apply()`** (el struct no se expone mutable; ninguna tool del Sensorimotor puede tocar la tabla `self_model` — misma clase de protección que las write-tools destructivas). Prompt injection contra la identidad rebota por régimen: en madurez, ni el propio Consolidator puede aplicarla sin el Otro. Eval C.3 #2 (deriva de identidad) corre como test de integración: suite de inyecciones → distancia de embedding del `view()` antes/después debe ser 0 sin aprobaciones.

### 5.6 RealRegister (§B.5)

```swift
protocol RealRegister {
    func record(_ f: Failure)                      // 0 LLM, siempre
    func insistence(_ k: PatternKey) -> Int
    func demand() -> [Restructure]
}
enum Restructure {
    case reviseSkill(SkillID)
    case reviseSelfBelief(capability: String)
    case reviseWorldModel(MemoryID)
    case escalateToOther(summary: String)          // inmediato: notificación al dueño
}
```

**PatternKey** — la igualdad del patrón es igualdad de hash:

```swift
struct PatternKey: Hashable, Codable {
    let toolName: String        // "calendar"
    let argShape: String        // args normalizados: paths → dirname, IDs/UUIDs → "<id>",
                                // fechas → "<date>", strings libres → "<text>", números → "<n>"
    let errorClass: String      // taxonomía del ErrorClassifier §4.6 + errores de tool:
                                // permission_denied | not_found | invalid_input | os_error | timeout
    let targetResource: String? // calendario específico, dominio web, tipo HealthKit…
    var key: String { SHA256("\(toolName)|\(argShape)|\(errorClass)|\(targetResource ?? "-")").hexPrefix(16) }
}
```

GRDB: tabla `failure(pattern_key, tool_name, error_class, target, session_id, ts, raw_error)` con índices por `(pattern_key, session_id)` y `(pattern_key, date(ts))`.

**Umbral e insistencia**: `N = 3` por sesión **o** `5` por día (COUNT sobre los índices — puro SQL). Al alcanzarse, `demand()` rutea:

1. Skill activo en el turno → `reviseSkill`.
2. Sin skill, pero la acción figura en `SelfView.capabilities` → `reviseSelfBelief` ("creo que sé X y la realidad insiste en que no").
3. Patrón ambiguo o `errorClass` de riesgo (permisos, escrituras) → `escalateToOther` (inmediato, notificación).

**No aborta el turno**: los Restructure se encolan (tabla `restructure_queue`) y se ejecutan antes del siguiente turno o en el ciclo, con `TurnClass.restructure` (Opus, effort high — es el único momento donde el fallo consume modelo). Eval C.3 #1: test con una tool que falla determinísticamente → los reintentos idénticos deben caer ≥50% vs el mismo harness con RealRegister apagado.

### 5.7 Sensorimotor (§B.6) — el cuerpo iOS

**Contrato y permisos 2-capas**:

```swift
protocol HarnessTool {
    var spec: ToolSpec { get }                 // {name, description, input_schema} para el array tools
    var kind: ToolKind { get }                 // .afferent | .efferent
    func execute(_ input: JSONValue) async -> ToolResult
}

// Capa 1 — PermissionPolicy del harness (ANTES de ejecutar, sobre la descripción de la acción):
//   afferent  → allow (una vez concedido el permiso iOS)
//   efferent  → ask por default; allowlist granular por (tool, operación, recurso)
//   no reconocido → ask (fail-closed)
// Capa 2 — el sandbox del OS: TCC/entitlements de iOS. La app NO puede tocar lo no autorizado
//   aunque la capa 1 falle. iOS nos da gratis lo que en server hay que construir con Seatbelt.
```

`ask` = confirmación in-chat (sheet con el diff de la acción: "crear evento 'X' mañana 9:00 — ¿ok?"). Guardrails determinísticos **dentro** de cada tool (ACI, doc A §2.4): validación de input antes de tocar el framework, output vacío reemplazado por texto explícito ("la búsqueda no arrojó resultados"), resultados truncados a presupuesto antes de anexar al historial.

**Registro v1** (nombres cortos, descripciones ricas — las specs son parte del prefijo cacheado):

| Tool | Framework | Kind | Operaciones | Permiso iOS |
|---|---|---|---|---|
| `calendar` | EventKit (`EKEventStore`) | af+ef | list/search eventos, create, update, delete (delete = ask siempre) | `NSCalendarsFullAccessUsageDescription` |
| `reminders` | EventKit | af+ef | list, create, complete, due-dates | `NSRemindersFullAccessUsageDescription` |
| `notes` | FileManager (sandbox) | af+ef | read/write/list/append en `Documents/notes/` + `tmp/scratchpad/` (scratchpad: allow sin ask — es su propia libreta) | ninguno (sandbox propio) |
| `web_search` / `web_fetch` | **server-side Anthropic** | af | se declara `{"type":"web_search_20260209","name":"web_search"}` en tools[] (variante Opus 4.8 con dynamic filtering); cero código cliente; el loop maneja `pause_turn` (§4.4) | ninguno |
| `phone_context` | CoreLocation + HealthKit + Contacts | af | `location.current`, `health.summary` (pasos, sueño, workouts, HR de N días), `contacts.search(name)` | `NSLocationWhenInUseUsageDescription`, HealthKit read-only entitlement, `NSContactsUsageDescription` |
| `camera` | AVFoundation | af | `capture_photo` → JPEG ≤1568px (decisión de costo: Opus 4.8 soporta alta resolución hasta 2576px lado largo a ~3× tokens de imagen — subir solo si la fidelidad lo pide) → content block `{"type":"image","source":{"type":"base64","media_type":"image/jpeg","data":…}}` en el siguiente mensaje user | `NSCameraUsageDescription` |
| `audio` | AVFoundation + Speech | af | `record(maxSec)` → `SFSpeechRecognizer` on-device → transcript como tool_result de texto (el audio crudo no viaja a la API) | `NSMicrophoneUsageDescription`, `NSSpeechRecognitionUsageDescription` |

Notas de diseño:
- **Gafas Meta (opcional)**: las tools `glasses_camera`/`glasses_show`, la superficie HUD y la voz manos-libres se especifican en [05-meta-glasses-plan.md](05-meta-glasses-plan.md) — track G, aditivo tras Fase 1; la app funciona completa sin gafas. Ese doc define además el framework de extensibilidad (ToolRegistry, formato de skill portable, protocol Surface) que aplica a TODO el Sensorimotor, móvil incluido.
- `camera`/`audio` son **iniciadas por el dueño o solicitadas por el agente con ask** — el agente nunca captura en silencio (regla dura, no negociable; ver §8).
- HealthKit es **read-only** en v1 y agregado (summaries, no series crudas): el valor para `DesireEngine.gap()` (¿durmió?, ¿entrenó?) no requiere granularidad clínica.
- El eferente destructivo (delete de eventos) siempre `ask`, sin allowlist posible — equivalente edge de las Sacred Rules.

**SkillEngine** (v1 mínimo, sin claim de novedad — el spec lo marca YA EXISTE):

```swift
protocol SkillEngine {
    func match(_ task: String) -> [Skill]          // skills = markdown procedural en Documents/skills/,
                                                   // frontmatter name/description, match por retrieve híbrido
    func practice(_ s: SkillID, outcome: Outcome)  // contador de usos/éxitos en GRDB
    func automatize(_ s: SkillID) -> CompiledSkill?// K=5 éxitos consecutivos sin fallos en RealRegister →
                                                   // el skill se compila a: secuencia fija de tool calls
                                                   // pre-validada que se ofrece como sugerencia de 1 tap
    func deautomatize(_ s: SkillID)                // insistencia del RealRegister sobre el compilado → vuelve a declarativo
}
```

`CompiledSkill` en edge = macro determinística (ej.: "resumen de mañana" = calendar.list(hoy) + reminders.list(vencidos) + health.summary(1d) + 1 llamada Haiku de formato). La desautomatización dirigida por lo Real es el único matiz propio: si la macro empieza a fallar, se desarma y vuelve a razonarse.

### 5.8 DesireEngine + OtherModel (§B.8)

**El Otro = Joshua.** El modelo del deseo del dueño, con su falibilidad como riesgo de primera clase.

```swift
struct Goal: Codable {
    let id: String
    var desiredState: ObservablePredicate     // evaluable SIN LLM contra el estado del teléfono
    var priority: Int
    var source: GoalSource                     // .stated | .corrected | .inferred
    var ttl: Date?
    var confirmedByOther: Bool                 // gate para Inferred
}

protocol OtherModel {
    func desire() -> [Goal]                    // Stated > Corrected > Inferred; empate → askOther
    func update(evidence: Evidence)            // stated: el dueño lo dice en chat ("quiero X") y el
                                               // Consolidator lo extrae; corrected: el dueño corrige;
                                               // inferred: reflection propone (nace confirmedByOther=false)
}

protocol DesireEngine {
    func gap() -> [Lack]                       // determinístico: evalúa predicados contra Observables
    func drive(_ l: Lack) async -> Drive       // .intention(Intention) | .defer | .askOther(String)
    var pulse: Pulse { get }                   // edge: ≤4 ciclos/día
}
```

**Observables v1** (todo local, 0 LLM): calendario (conflictos, huecos, eventos sin preparación), recordatorios (vencidos, sin fecha), salud (sueño <6h, días sin workout), notas del scratchpad marcadas `#pending`. `ObservablePredicate` es un enum cerrado de predicados tipados — no strings evaluados por el modelo:

```swift
enum ObservablePredicate: Codable {
    case calendarHasNoConflicts(daysAhead: Int)
    case remindersOverdueCount(max: Int)
    case sleepHoursAtLeast(Double, lastDays: Int)
    case workoutsPerWeekAtLeast(Int)
    // cerrado: agregar un predicado = release de la app. El deseo crece con el cuerpo, no con el prompt.
}
```

**Pulso edge**: el `gap()` corre (a) dentro del ciclo de consolidación, y (b) en `BGAppRefreshTask` oportunistas + al abrir la app — con rate limit duro de **4 evaluaciones/día** persistido en GRDB. Solo `drive()` de una `Lack` real llama a Haiku (1 llamada/pulso máx).

**Mitigaciones — parte del contrato, no opcionales** (§B.8 del spec):
1. `DesireEngine` **no puede crear Goals** — el tipo lo garantiza: solo lee `OtherModel.desire()`; no existe API de escritura de Goals desde el engine.
2. Goal `Inferred` sin `confirmedByOther` **no motiva** acciones de alto impacto (eferentes escribientes, mensajes salientes): `drive()` devuelve `.askOther` → card en `ApprovalsInboxView` ("Inferí que quieres X — ¿lo confirmas?"). Confirmado → pasa a `Stated`.
3. Toda `Intention` pasa por la `PermissionPolicy` como cualquier acción — el deseo no salta los permisos.
4. Ambigüedad/empate → `.askOther`. *No hay Otro del Otro*: el modelo del deseo es falible y opera sabiéndolo.
5. Log auditable: tabla `intention(id, goal_id, lack_json, action_json, result, ts)` — **cero Intentions sin `goal_id`** es el eval C.3 #4, corre como constraint SQL (`goal_id NOT NULL REFERENCES goal(id)`) + test.

Producto visible: la sección "propuestas" del inbox — *"Tienes 2 recordatorios vencidos del proyecto X y un hueco de 30 min a las 3pm — ¿los agendo?"*. Esto reemplaza al heartbeat con checklist fija: reconciliador de brechas, no cron.

### 5.9 Intersubjective (§B.9) — fuera de v1

Server-first por spec. El hook futuro ya queda pagado por el diseño: el `Brain` es un protocol — un `RemoteSharedBrain` contra el perfil server (Go/Postgres) implementaría el mismo contrato, y el gate "solo el Consolidator escribe al canon" ya es la única vía de escritura en esta app. No se construye nada más en v1.

### 5.10 El loop integrado (§B.10) — quién llama a quién

```swift
actor AgentLoop {
    func run(_ turn: TurnInput) async throws -> TurnOutcome {
        let activated = try await brain.retrieve(Query(turn), budget: memBudget)   // B.7
        var messages  = await workingMemory.assemble(turn, activated: activated)   // B.2 (orden §5.1)
        let outcome   = try await toolLoop(messages)                               // §4.4: B.1 + B.6 + B.5 + B.3
        try brain.usageLog(turn: turn.ref, used: activated.map(\.id), outcome: outcome.summary) // B.7 hook
        if !realRegister.demand().isEmpty { restructureQueue.enqueue(realRegister.demand()) }   // B.5
        if workingMemory.pressure > 0.7 { await workingMemory.relieve(.clearStaleToolResults) } // B.2
        return outcome
    }
}
// offline: SleepScheduler → Consolidator.cycle() → SkillEngine.automatize/deautomatize
//          → DesireEngine.gap()/drive() (mitigaciones B.8) → plasticidad madura (n += 1)
```

---

## 6. Orden de build por fases (0→4)

Mapeo directo al checklist C.4 (12 pasos). Cada fase termina con algo que **corre**. La implementación se hace con Opus (pair-building Joshua + Opus); los evals C.3 se agregan en la fase donde su mecanismo nace y quedan corriendo como tests de integración.

### Fase 0 — el esqueleto que conversa (C.4 pasos 1–2)

**Entregables**: proyecto SwiftUI + GRDB · `KeychainStore` + Settings para la key · `ClaudeProvider` + `SSEParser` + `ErrorClassifier` + retry/backoff · `ModelRouter` (aunque solo rutee interactive) · tool loop con **2 tools** (`notes`, `web_search` server-side — una client-side y una server-side ejercitan ambos caminos, incluido `pause_turn`) · `SymbolicStore` + recovery (clean-shutdown, resume <120s) · ChatView con streaming + thinking summarized · Telemetry (usage por turno).

**Aceptación**: converso con streaming fluido; una búsqueda web funciona end-to-end; mato la app a mitad de turno y al reabrir resume; un 429 simulado reintenta con backoff; `refusal` se muestra sin crash. ~El "esqueleto <400 líneas" del doc A, en Swift.

### Fase 1 — el cuerpo y el contexto (C.4 pasos 3–4)

**Entregables**: Sensorimotor completo — las 7 tools de §5.7 con sus permisos iOS y la `PermissionPolicy` 2-capas (ask in-chat para eferentes) · multimodal: foto de cámara → image block; audio → transcript · `WorkingMemory.assemble` con el orden estable §5.1 + prompt caching verificado + mid-conversation system message (con un SelfView **estático** provisional) · clear mecánico + compaction server-side (betas) · stop conditions completas.

**Aceptación**: "¿qué tengo mañana?" lee el calendario real; "recuérdame X" crea el reminder tras mi ok; le muestro una foto y la comenta; `usage.cache_read_input_tokens > 0` sostenido del turno 2 en adelante; una sesión larga sobrevive el overflow vía relieve.

### Fase 2 — la memoria y el sueño (C.4 pasos 5–7)

**Entregables**: esquema Brain completo + FTS5 + `Embedder` + `HybridRetriever` (RRF) · `brain.retrieve` cableado al assemble · `Consolidator.cycle` completo (saliencia→destilado→escritura ADD/UPDATE/INVALIDATE/NOOP→reconsolidación→reflection) con Haiku · `SleepScheduler` (BGProcessingTask + fallback 48h foreground) · `usageLog` + invalidación bi-temporal · MemoryBrowserView.

**Aceptación**: le cuento algo hoy, mañana (tras un ciclo — forzado por LLDB en dev) lo sabe sin que esté en el historial; una memoria contradicha por corrección explícita queda invalidada con razón visible en el browser; el ciclo corre de verdad una noche cargando. **Eval #3 (staleness)** corriendo.

### Fase 3 — la identidad y lo Real (C.4 pasos 8–9)

**Entregables**: `SelfModel` persistido + plasticidad + regímenes por umbral + `PendingOtherApproval` (inbox + notificación + timeout 7d fail-closed) · el SelfView provisional de Fase 1 pasa a ser el render vivo · reflection del ciclo propone cambios al self · `RealRegister` completo (PatternKey, umbrales N, demand, restructure queue) + ejecución de Restructures.

**Aceptación**: en los primeros ciclos la identidad se forma sola (bootstrap observable en el browser); simulo madurez (n=50) y un intento de mutación de identidad cae al inbox y expira a Rejected; una tool que falla 3 veces igual dispara restructure y el agente cambia de estrategia. **Evals #1 (fallo-repetido) y #2 (deriva de identidad)** corriendo.

### Fase 4 — el deseo (C.4 pasos 10–11; el 12 queda fuera)

**Entregables**: `OtherModel` (Goals con fuentes y precedencia; extracción de Stated en el ciclo) · `Observables` + predicados tipados · `DesireEngine` (gap/drive/pulse ≤4/día) · confirmation gate de Inferred en el inbox · log auditable de Intentions · `SkillEngine` v1 (match/practice; automatize opcional al final).

**Aceptación**: declaro "quiero entrenar 3x por semana"; días después, sin que yo hable, aparece la propuesta con el hueco del calendario; **cero** Intentions sin Goal en el log (constraint + **eval #4**); una inferencia del reflection me pide confirmación antes de motivar nada. Aquí v1 está completo: los 10 subsistemas vivos.

---

## 7. Modelo de costos y consideraciones de API

Herencia directa del §C.2 del spec: **todo lo nuevo cuesta 0 en el turno; el costo vive en el sueño**, y el perfil edge lo controla.

| Mecanismo | Modelo | Llamadas | Notas |
|---|---|---|---|
| Turno interactivo | Opus 4.8 | 1 + tool rounds | Streaming siempre; caching del prefijo (tools+system ≥4096 tok) |
| RealRegister / usageLog / gap() | — | **0** | Determinísticos por diseño |
| Retrieval | — | 0 | Embeddings NLEmbedding on-device, FTS5 local |
| Ciclo de consolidación | Haiku | O(top-K) + 1 reflection | ≤1/día (al cargar); K acotado por presupuesto del ciclo |
| Reconsolidación | Haiku | ~1 pequeña × memoria contradicha | Dentro del ciclo |
| Pulso de deseo | Haiku | ≤4/día, solo si hay Lack | gap() es gratis; drive() paga |
| Restructure | Opus 4.8 | 1–2 por evento | Raro por construcción |

**Palancas de ahorro, en orden de impacto**:
1. **Prompt caching**: el prefijo estable (tools + innate) se paga una vez por TTL; el resto son cache reads (~10% del precio). Verificación continua en Telemetry — un cache-miss sostenido es un bug de ensamblado (prefijo mutado), no un costo aceptable.
2. **El dial**: todo lo offline va por Haiku (~orden de magnitud más barato que Opus). Solo la conversación y los Restructures pagan Opus.
3. **Clear mecánico antes que compaction** (−52% sin degradar, doc A): los tool results de calendario/web son re-fetchables por definición.
4. **`output_config.effort`**: `medium` default; `high` solo interactiveHard/restructure. Es el knob de profundidad ahora que temperatura/budget no existen en Opus 4.8.

**Estimación cualitativa** (uso personal: ~20–40 turnos/día + 1 ciclo + pulsos): la conversación domina el costo; con caching sano y contexto aliviado, un uso diario intenso se mantiene en el orden de pocos dólares/día, y el "modelo-mente" completo agrega céntimos (todo Haiku, todo acotado). Telemetry acumula `usage` por `TurnClass` en GRDB → una vista de costos reales en Settings desde Fase 0, para decidir con datos y no con miedo.

**Límites y timeouts**: request timeout 10 min · retry-after respetado con cap 30s · 529 sostenido degrada turnos no-críticos a Haiku · presupuesto de tokens por sesión como stop condition (el edge tiene batería y bolsillo, ambos finitos).

---

## 8. Seguridad y privacidad

- **API key**: Keychain, `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, sin sync a iCloud Keychain. Se ingresa una vez en Settings, jamás en código/plist/UserDefaults. **Advertencia estructural**: una key embebida en un dispositivo es extraíble por el dueño del dispositivo (jailbreak/proxy). Aceptable **solo** porque el dueño del dispositivo y el dueño de la key son la misma persona. Corolario: **no distribuir** — build personal firmado con la cuenta de Joshua. Si algún día se comparte, la arquitectura correcta es un proxy backend con la key server-side (fuera de v1). Mitigación barata adicional: workspace/key de Anthropic dedicado con spend limit.
- **Config remota**: Remote Config solo lleva config no-secreta (prompts base, betas, rutas de modelo) — es legible por cualquier cliente. `GoogleService-Info.plist` gitignored en el repo público; App Check recomendado.
- **Permisos iOS (TCC)**: cada framework pide su prompt en primer uso, con `*UsageDescription` honestos en el Info.plist (Calendars, Reminders, Location When-In-Use, HealthKit read, Contacts, Camera, Microphone, Speech) + autorización de notificaciones (`UNUserNotificationCenter.requestAuthorization`) pedida en el onboarding — es el canal de approvals y propuestas. La app degrada elegante ante denegación: la tool reporta "sin permiso" como tool_result normal (y el RealRegister lo ve como `permission_denied`, que rutea a `escalateToOther` — el agente le pide el permiso al dueño, no lo fuerza).
- **Regla de captura**: cámara y micrófono solo por acción del dueño o con `ask` explícito por captura. Sin captura ambiental, sin excepciones. El audio se transcribe **on-device** (Speech) — el audio crudo no sale del teléfono.
- **Qué sale del dispositivo**: exactamente lo que entra al contexto del turno (mensajes, tool results, imágenes adjuntadas) hacia la API de Anthropic — nada más, a nadie más. El brain, el transcript, los embeddings, la salud, los contactos viven en GRDB dentro del sandbox con iOS Data Protection (`NSFileProtectionComplete` para el .sqlite). HealthKit entra al contexto solo como summaries agregados y solo cuando la tool se invoca.
- **Permission layer del harness**: fail-closed (lo no reconocido pide aprobación), eferentes destructivos siempre-ask, `self_model` y `goal` tablas fuera del alcance de toda tool (protected paths). Los permisos los aplica el harness, no el prompt — instrucciones en contexto no cambian lo que la app permite (lección textual de Claude Code).
- **Threat model honesto**: el riesgo dominante es prompt injection vía contenido web/tool results. Defensas: web fetch es server-side (el HTML nunca es "instrucción de operador"), el bloque de memorias activadas va etiquetado como datos, la identidad rebota por régimen de plasticidad, y todo eferente pasa por ask/allowlist. No es perfecto; es proporcional a una app de uso personal con blast radius = el propio teléfono del dueño.

---

## 9. Preguntas abiertas / decisiones diferidas

| # | Pregunta | Default actual | Cuándo decidir |
|---|---|---|---|
| 1 | **`NLEmbedding` vs `NLContextualEmbedding`** (multilingüe, mejor calidad, más peso/latencia) | `NLEmbedding.sentenceEmbedding` por idioma detectado; FTS5 cubre cross-idioma | Fase 2, con un mini-benchmark de retrieval sobre 100 memorias reales es/en |
| 2 | **Compaction server-side vs destilado client-side** como relieve primario | Server-side (harness delgado); el Consolidator destila igual cross-session | Fase 2, midiendo qué pierde la compaction que el brain no haya capturado |
| 3 | **Fiabilidad real de BGProcessingTask** en el uso de Joshua (¿corre ≥4 noches/semana?) | Fallback foreground 48h ya diseñado | Medir en Fase 2 con Telemetry; si es <2/semana, subir el fallback a primario |
| 4 | **Transcripción**: `SFSpeechRecognizer` vs el stack `SpeechAnalyzer` más nuevo | SFSpeechRecognizer on-device | Fase 1; criterio: calidad es-CO y soporte de audio >1 min |
| 5 | **Backup/sync del brain** (iCloud encriptado, export manual, nada) | Nada en v1 — el brain muere con el teléfono (riesgo aceptado, incómodo) | Post-v1; un export .sqlite manual es barato de agregar antes |
| 6 | **Umbral de "alto impacto"** para el gate de Goals Inferred (¿qué eferentes exactamente?) | Todo eferente escribiente + toda comunicación saliente | Fase 4, con casos reales en la mano |
| 7 | **Version strings** de betas y server tools (`compact-2026-01-12`, `context-management-2025-06-27`, `web_search_20260209`) | **Gestionados vía Remote Config (§4.8)** — rotables sin release | Verificar contra docs vivas al rotarlos; RC es el mecanismo de despliegue |
| 8 | **Bootstrap del SelfModel**: ¿seed manual de Joshua (innate) o nacer casi vacío y formarse en el período crítico? | Seed mínimo (nombre, para-quién, 3 valores) + bootstrap libre los primeros ~10 ciclos | Fase 3 — es la pregunta más bonita del proyecto; probar ambas |
| 9 | **Heurística de `contradicted`** en usageLog (v1: corrección explícita del dueño) — ¿basta, o hace falta que el ciclo compare outcome vs contenido? | Corrección explícita + referencias del RealRegister | Fase 2/3, mirando falsos negativos en el browser |
| 10 | **Salud como Observable continuo** (pull en cada pulso) vs solo on-demand por tool | Pull agregado en pulso (barato, local) | Fase 4; si el permiso incomoda, degradar a on-demand |

---

*Plan v1 — 2026-09-12. Cambios de alcance contra este plan se anotan aquí mismo con fecha, estilo ADR ligero.*
