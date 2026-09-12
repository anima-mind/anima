# Anima × Meta glasses — las gafas como segundo cuerpo (plan)

> Extensión **opcional** de `anima-ios` ([doc 04](04-swift-implementation-plan.md)) a las Meta Ray-Ban Display + Neural Band vía el **Wearables Device Access Toolkit (DAT)**. La app funciona completa sin gafas; con gafas, la misma mente gana un segundo cuerpo.
>
> **Fuente técnica**: `glasses/relay/docs/meta-glasses-dat-swift.md` — conocimiento del DAT SDK **verificado en hardware real** durante el desarrollo de Relay (SDK 0.9.0, 2026-08-03). Todo dato del SDK citado aquí proviene de ese doc o del `.swiftinterface` real; lo no verificado se marca ⚠️.
>
> **Decisiones fijadas**: gafas = superficie opcional del Sensorimotor (§B.6 del [spec](03-harness-mind-convergence.md)) + nueva superficie de presentación · capability detection con degradación elegante · DAT SDK nativo (no la vía web-apps) · pin `exactVersion 0.9.0` · la extensibilidad (tools/skills/surfaces) se define aquí como **framework único de toda la app** — móvil y gafas.

---

## 1. Tesis: segundo cuerpo, no segunda mente

El modelo del DAT coincide punto por punto con el modelo-mente del spec:

| Hecho del DAT (verificado) | Lectura anima |
|---|---|
| "La app NO se instala en las gafas; corre en el iPhone" | La mente vive en un solo lugar (anima-ios) |
| "El teléfono es la fuente de verdad; las gafas son un HUD sin estado" | Un solo transcript (B.3), un solo brain (B.7) — las gafas no tienen historia propia |
| La app manda árboles de vistas; recibe interacciones como callbacks | Las gafas son **órganos**: eferente (HUD, parlantes) y aferente (cámara POV, mic) |
| El display duerme, se desconecta, se recalienta | El cuerpo se cansa y duele — señales para el RealRegister (B.5) |

En términos del doc 02: las gafas son una **prótesis** — extensión del esquema corporal, no un segundo sujeto. La neurociencia de la incorporación de herramientas (el espacio peripersonal se extiende hasta la punta de la herramienta) es el análogo exacto: la misma mente, con el cuerpo extendido hasta la cara. 🎭 Una sola consecuencia práctica gobierna todo el diseño: **cero estado en las gafas; toda pérdida de link es recuperable re-enviando desde el teléfono.**

Impacto por subsistema (nada del core cambia; todo es aditivo):

| Subsistema | Qué gana con gafas | Qué NO cambia |
|---|---|---|
| B.6 Sensorimotor | Tools `glasses_camera` (af) y `glasses_show` (ef); voz manos-libres como modalidad de entrada | El registro, los permisos 2-capas, el contrato `HarnessTool` |
| B.2 WorkingMemory | 1 línea de estado corporal en el system message volátil ("gafas: conectadas, batería OK") | El orden de ensamblado y el caching (la línea va en la zona volátil) |
| B.5 Real | Nuevos `errorClass`: `glasses_unavailable`, `glasses_thermal`, `glasses_battery`, `glasses_version_mismatch` | PatternKey, umbrales, ruteo |
| B.8 Desire | Las propuestas y approvals se proyectan al HUD como cards con botones (confirmación por captouch/pinch) | El pulso ≤4/día, las mitigaciones, el gate de Inferred |
| B.3/B.7 | Los turnos por gafas entran al mismo transcript; las fotos POV se destilan a memoria en el ciclo | Una historia, un brain |
| B.4 SelfModel | Nada (el yo no depende del cuerpo activo) | Todo |

---

## 2. Restricciones duras del DAT (las reglas que gobiernan el diseño)

Destilado de lo verificado en hardware (doc Relay §3–§9). Estas son **leyes físicas** del proyecto, no preferencias:

1. **Pin `exactVersion 0.9.0`** (SPM, `meta-wearables-dat-ios`) calzado con la DAT app instalada en las gafas. La verdad es el handshake, no la doc. Antes de bumpear el pin: actualizar primero la DAT app de las gafas. Mínimo **iOS 17.2**.
2. **UN `AutoDeviceSelector` + UNA `DeviceSession`**, reutilizados; `stop()` siempre en teardown o el device queda reclamado (`noEligibleDevice` eterno — sesión zombie).
3. **Elegibilidad = `addCompatibilityListener`**, no solo link state (la compatibilidad llega tarde; decidir solo con link da falsos negativos).
4. En 0.9 **`stateStream()`/`errorStream()` terminan al llegar `.stopped`** — re-suscribir al recrear sesión; tokens en `ListenerTokenBag` **nuevo** por generación (bag viejo → `cancelAll()` async; no reusar el mismo bag).
5. **Cada `display.send()` reemplaza TODA la vista** — no hay update parcial. El display **duerme por inactividad** → re-enviar contenido al despertar (observar state). Scroll vertical lo maneja el sistema; horizontal no existe.
6. **Vocabulario visual cerrado**: `Text` (.heading/.body/.meta, color solo primary/secondary), `Icon` (catálogo cerrado ~115 glyphs — no hay mic, inbox ni github), `Button` (.primary/.secondary/.outline), `ButtonGroup` (0.9; sin padding propio), `Image` (URI), `FlexBox` (background solo .none/.card). Root = FlexBox o VideoPlayer.
7. **Input abstraído**: la app solo recibe `Button.onClick`, `FlexBox.onTap`, `onPlaybackEvent`. Sin gestos crudos. El **back físico TERMINA la sesión siempre** → tratarlo como salida (teardown limpio); navegación "atrás" = botón explícito renderizado por la app.
8. **Audio = Bluetooth del sistema, no DAT**: A2DP (salida estéreo 44.1/48k) y HFP (bidireccional, **8 kHz mono**, beamforming que aísla la voz del usuario) son **mutuamente excluyentes**; la ruta HFP tarda ~2 s en asentarse y hay que verificarla; si se combina con camera stream, HFP se configura ANTES.
9. **Errores físicos existen**: `.thermalCritical`, `.thermalEmergency`, `.batteryCritical`, `.peakPowerShutdown`, `.datAppOnTheGlassesUpdateRequired` (→ ofrecer `openDATGlassesAppUpdate()`), `StreamError.hingesClosed` al quitarse las gafas.
10. **Setup**: bloque `MWDAT` en Info.plist (AppLinkURLScheme propio, ClientToken, MetaAppID, TeamID del portal Wearables Developer Center) + permisos BT/accesorio + background modes; registro one-time vía `startRegistration()` → Meta AI app → `handleUrl(url)`. Linkear SOLO `MWDATCore` + `MWDATDisplay` (+`MWDATCamera` si se usa); MockDevice jamás en el target de producción.

---

## 3. Arquitectura: `GlassesBody` + la abstracción `Surface`

### 3.1 GlassesBody — el actor que encarna las reglas del §2

```swift
actor GlassesBody {
    enum BodyState: Equatable {
        case absent            // sin registro o sin device
        case incompatible      // linked pero handshake/DAT app no calza (regla 3)
        case dormant           // elegible, sin sesión activa
        case active            // DeviceSession .started + Display .started
        case ailing(GlassesAilment)  // thermal | battery | updateRequired — el cuerpo duele
    }

    private let selector: AutoDeviceSelector      // UNO, vive toda la app (regla 2)
    private var session: DeviceSession?           // UNA, o nil
    private var tokens: ListenerTokenBag          // bag NUEVO por generación (regla 4)

    func ensureActive() async throws              // idempotente: crea/arranca solo si hace falta
    func teardown() async                         // display.stop() + session.stop() SIEMPRE
    var stateStream: AsyncStream<BodyState> { get }
    var statusLine: String { get }                // → system message volátil (B.2)
}
```

- `ensureActive()` es la única vía de arranque; el back físico y `hingesClosed` disparan `teardown()` (regla 7) y el estado vuelve a `.dormant` — la próxima interacción re-crea la sesión con streams y bag nuevos.
- `ailing` alimenta el RealRegister con los `errorClass` del §1 y, si es `updateRequired`, la acción sugerida (`openDATGlassesAppUpdate()`) se le propone al dueño — el agente nunca fuerza updates.

### 3.2 Surface — la tercera dimensión de extensibilidad

El HUD obliga a nombrar algo que en el doc 04 estaba implícito: la mente **habla semántica**, y cada superficie la **proyecta a su idioma**. Se formaliza como protocol — y esto vale para el teléfono también:

```swift
protocol Surface: AnyObject {
    var id: SurfaceID { get }                          // .phoneChat | .glassesHUD | (futuro: .widget, .watch, .animadREST)
    var capabilities: SurfaceCapabilities { get }      // freeText, images, maxButtons, voiceIn, audioOut
    func render(_ content: SurfaceContent) async       // proyección — pura, reemplazable, testeable
    var events: AsyncStream<SurfaceEvent> { get }      // .userText | .voiceTranscript | .buttonTapped(ActionID) | .exited
}

enum SurfaceContent {                                  // semántico, NUNCA visual
    case assistantTurn(text: String, attachments: [Attachment])
    case proposal(Proposal)                            // B.8: card con confirm/dismiss
    case approvalRequest(ApprovalItem)                 // B.4: diff + aprobar/rechazar
    case status(String)                                // "pensando…", errores amables
}
```

- `PhoneChatSurface` (la ChatView del doc 04) y `GlassesHUDSurface` implementan el mismo contrato. El `AgentLoop` no sabe qué superficie está activa — recibe `SurfaceEvent`s y emite `SurfaceContent`s.
- La restricción de input de las gafas queda **codificada en el tipo**: `GlassesHUDSurface` solo emite `.voiceTranscript`, `.buttonTapped` y `.exited` — nunca `.userText` (no hay teclado en la cara).
- **Continuidad entre superficies gratis**: como el transcript es uno (B.3), empezar en las gafas y seguir en el teléfono es abrir la otra ventana de la misma conversación. Patrón *handoff*: contenido largo en el HUD termina en un botón "ver en el teléfono".

### 3.3 Integración con el loop

```
SurfaceEvent (voz transcrita / botón / texto)
   → AgentLoop.run(turn)                    ← idéntico al doc 04 §5.10
   → TurnOutcome
   → SurfaceRouter.render(outcome)          → a TODAS las superficies activas
                                              (el teléfono siempre; el HUD si BodyState == .active)
```

El `SurfaceRouter` es la única pieza nueva en el camino del turno: decide proyección, no contenido. Reglas: respuestas del turno → superficie donde se originó + espejo en el teléfono; propuestas proactivas (B.8) → ver etiqueta de atención (§4.3).

---

## 4. La superficie HUD

### 4.1 Renderer puro + design system mínimo

Regla 5 (send reemplaza todo) → el renderer es una **función pura** `HUDViewModel → FlexBox`, sin estado en las gafas. Se re-invoca ante: nuevo contenido, wake del display, reconexión.

Design system (sobre el vocabulario cerrado del §2.6):

| Componente | Composición DAT | Uso |
|---|---|---|
| `CardView` | FlexBox column + Text .heading + Text .body + ButtonGroup (≤3) | Turno del asistente, respuestas |
| `ProposalCard` | Card + Icon .bell + botones "Dale" / "Ahora no" | Propuestas del deseo (B.8) |
| `ApprovalCard` | Card + Icon .exclamationTriangle + diff resumido + "Aprobar" / "Rechazar" | PendingOtherApproval (B.4) — confirmar con pinch en la cara |
| `ListView` | FlexBox column de FlexBox .card con onTap | Agenda de mañana, recordatorios |
| `TalkView` | Icon .speechBubble + Text .meta "escuchando…" + botón cancelar | Captura de voz activa |

- Iconos: mapear semántica al catálogo cerrado (`bell` propuesta, `checkmarkCircle` ok, `x` cancelar, `calendar`, `exclamationTriangle` alerta). Lo que no exista → glyph más cercano, nunca inventar nombres.
- Texto: cards cortas (el HUD no es para leer ensayos). Presupuesto duro de caracteres por card; overflow → handoff al teléfono.
- Navegación 100% de la app: callback → `send(siguienteVista)`; botón "Atrás" explícito en toda vista no-raíz (regla 7).

### 4.2 Voz — el pipeline manos-libres

Secuencia (respeta la exclusividad HFP/A2DP y el asentamiento de ruta del §2.8):

```
[usuario: botón "hablar" en HUD o abre TalkView]
  → AVAudioSession .playAndRecord + .allowBluetoothHFP → preferir input HFP
  → esperar ~2s y VERIFICAR currentRoute (si no asentó: reintentar 1 vez, luego degradar a mic del teléfono)
  → AVAudioEngine tap → SFSpeechRecognizer on-device (es-CO)   ← igual que la tool `audio` del doc 04
  → transcript → SurfaceEvent.voiceTranscript → AgentLoop
  → desactivar HFP
  → respuesta: card al HUD + TTS corto (AVSpeechSynthesizer) por A2DP
```

- El beamforming del HFP aísla la voz del usuario (feature verificada) — ideal para calle/oficina.
- TTS solo para respuestas cortas (confirmaciones, resúmenes de 1-2 frases); lo largo va como card + handoff. Sin barge-in en v1.
- El audio crudo **nunca sale del teléfono** (STT on-device) — misma regla de privacidad del doc 04 §8.

### 4.3 Etiqueta de atención (el HUD está en la cara)

Más estricto que las notificaciones del teléfono:

1. **Turnos reactivos**: se proyectan al HUD solo si la sesión de gafas está `.active` (el usuario está engaged).
2. **Propuestas del deseo (B.8)**: NO despiertan el display. Camino: notificación local estándar → si el usuario abre desde las gafas, la propuesta está como `ProposalCard`. Excepción única: `escalateToOther` urgente (B.5) puede proyectarse si el display está despierto.
3. **Rate limit de proyección proactiva**: ≤N cards proactivas/hora (configurable), acumulables en una vista "pendientes" — jamás spamear la cara.
4. El pulso del deseo **no cambia** (≤4/día): el HUD es una proyección nueva de las mismas propuestas, no un canal que genera más.

---

## 5. Tools nuevas y permisos (B.6 extendido)

| Tool | Kind | Qué hace | Permiso |
|---|---|---|---|
| `glasses_camera` | af | Captura foto POV ("¿qué estoy viendo?") → JPEG → content block image (mismo camino que `camera` del doc 04) | **ask** si la pide el agente (la confirmación se renderiza como card y se acepta con pinch — consentimiento en el mismo dispositivo); **allow** si la inicia el dueño. El LED de privacidad del hardware siempre aplica. |
| `glasses_show` | ef | Empuja una card al HUD (el agente decide mostrar algo en la cara) | allow con rate limit del §4.3 — es eferente de bajo riesgo (no muta el mundo), pero consume atención |

- La **voz no es tool**: es modalidad de entrada de la `GlassesHUDSurface`.
- ⚠️ La API concreta de captura de foto vive en `MWDATCamera` (`DeviceSession.addCamera(config:).stream`, errores `StreamError.photoCaptureFailed`) — **verificar contra el `.swiftinterface` real al implementar G1**; el doc de Relay verificó el capability model, no el detalle de foto.

**Estabilidad de caché (decisión)**: las tools de gafas se **declaran siempre** en el array `tools` (el prefijo cacheado no muta con el estado del link — regla del doc 04 §5.1). Sin gafas, `execute()` responde `is_error: "gafas no conectadas"` (`errorClass: glasses_unavailable`) y la línea de estado corporal del system message volátil ya le dijo al modelo que no las use. Si el modelo insiste 3× → el RealRegister escala al dueño — comportamiento correcto, no ruido.

---

## 6. Extensibilidad — el framework único (móvil Y gafas)

Requisito de diseño de primera clase: **agregar tools, skills y superficies sin tocar el core**. Las gafas son la prueba de existencia de los tres ejes. Estos contratos aplican a toda la app — el teléfono los usa igual — y son candidatos a promoverse al [spec](03-harness-mind-convergence.md) cuando el código los valide (animad deberá obedecer los mismos).

### 6.1 Eje 1 — Tools (`ToolRegistry`)

```swift
protocol HarnessTool {                          // doc 04 §5.7, extendido
    var spec: ToolSpec { get }
    var kind: ToolKind { get }                  // .afferent | .efferent
    var requiredBody: BodyRequirement { get }   // .phone | .glasses — el guard lo aplica el registro
    var defaultPermission: PermissionDefault { get }
    func execute(_ input: JSONValue) async -> ToolResult
}

final class ToolRegistry {
    func register(_ tool: HarnessTool)          // en el arranque; orden alfabético SIEMPRE (caché)
    var specs: [ToolSpec] { get }               // estable dentro de la sesión
    func execute(_ call: ToolCall) async -> ToolResult   // guards: body presente → permiso → ACI de la tool
}
```

**Receta "nueva tool"** (1 archivo, 0 cambios al core): conformar `HarnessTool` → registrar en el arranque → checklist: nombre corto + descripción rica (es parte del prefijo cacheado), guard de input dentro (ACI), `errorClass` mapeados para el RealRegister, resultado truncado a presupuesto, test contra mock. Cambios al set de tools = release de la app (el set es cache-estable por diseño).

### 6.2 Eje 2 — Skills (portables entre cuerpos y runtimes)

Los skills son **memoria procedimental en markdown** — y eso los hace el único artefacto **portable**: un skill escrito para anima-ios sirve tal cual en animad (Go); las tools son nativas por runtime, los skills viajan.

```
Documents/skills/<nombre>/SKILL.md
---
name: resumen-de-manana
description: Arma el resumen del día siguiente con agenda, pendientes y sueño
requires_tools: [calendar, reminders, phone_context]     # si falta una tool, el skill no matchea
surfaces: [phoneChat, glassesHUD]                        # dónde tiene sentido proyectarlo
---
<instrucciones procedurales en markdown>
```

- **Carga**: `Documents/` es visible en la app Files → Joshua crea/edita skills **sin release**; se indexan (retrieve híbrido del brain) al abrir la app o en el ciclo.
- **El agente también escribe skills**: el Consolidator, al detectar un workflow repetido exitoso (señal: `practice()` del SkillEngine), redacta un borrador en `skills/drafts/`; promoción a activo = 1 tap del dueño en el inbox. Un skill activo que acumula insistencia en el RealRegister → `reviseSkill` (doc 04 §5.6) o des-promoción.
- `surfaces:` le da al skill conciencia de proyección: un skill con `glassesHUD` puede declarar su salida como card corta (p. ej. "resumen-de-manana" en las gafas = `ListView` de 3 ítems, en el teléfono = texto completo).

### 6.3 Eje 3 — Surfaces

Conformar `Surface` (§3.2) es todo lo que exige una superficie nueva. `GlassesHUDSurface` es la prueba; la lista de espera natural: widget de iOS, watchOS, CarPlay, y el caso mayor — **animad exponiendo una `RESTSurface`** (la API REST del server es "una superficie más" de la misma mente, no otra arquitectura).

---

## 7. Fases (track G, aditivo)

Requiere Fase 0–1 del doc 04 (loop + Sensorimotor + WorkingMemory). Corre en paralelo a las fases 2–4 — ningún entregable de gafas bloquea el core, y viceversa.

### G0 — el cuerpo se enciende
**Entregables**: SPM pin 0.9.0 + Info.plist MWDAT + registro en portal Wearables (MetaAppID/ClientToken) · flujo `startRegistration`/`handleUrl` · `GlassesBody` completo (selector único, sesión única, compat listener, teardown, re-suscripción post-`.stopped`, `ailing` → RealRegister) · card estática "anima 👋" renderizada · tests con MockDeviceKit (`pairGlasses(.rayBanMetaOptics)`, captouch simulado; mock NUNCA en target de producción).
**Aceptación**: gafas reales conectan y muestran la card; quitarse las gafas / back físico / apagarlas → teardown limpio y reconexión al volver, **cero sesiones zombie en 20 ciclos seguidos**; `statusLine` aparece en el system message volátil.

### G1 — conversación en la cara
**Entregables**: `Surface` protocol + refactor de ChatView a `PhoneChatSurface` (mismo contrato) · `GlassesHUDSurface` con renderer puro + design system §4.1 + re-send en wake · pipeline de voz §4.2 (HFP→STT→turno→card+TTS A2DP, con verificación de ruta y fallback a mic del teléfono) · tools `glasses_camera` y `glasses_show` en el `ToolRegistry` (verificando la API de MWDATCamera contra el `.swiftinterface`) · handoff "ver en el teléfono".
**Aceptación**: pregunto por voz desde las gafas "¿qué tengo mañana?" y la agenda aparece como card + TTS corto, manos libres, con el teléfono en el bolsillo; "¿qué estoy viendo?" captura POV (con mi confirmación pinch) y el agente lo describe; la conversación continúa intacta al abrir el teléfono.

### G2 — proactividad y extensión
**Entregables**: `ProposalCard`/`ApprovalCard` en el HUD (B.8/B.4 proyectados) con etiqueta de atención §4.3 · `surfaces:` en el formato de skill + 1 skill demo con proyección dual · borradores de skills por el Consolidator (`drafts/` + promoción en inbox) · telemetría de gafas (sesiones, errores físicos, ratio voz/botones).
**Aceptación**: una propuesta del deseo llega como notificación y la confirmo con un pinch desde las gafas sin tocar el teléfono; el eval C.3 #4 (cero Intentions sin Goal) sigue verde con el nuevo camino de confirmación; un skill editado en Files se refleja sin release.

---

## 8. Seguridad y privacidad (delta sobre doc 04 §8)

- **Cámara en la cara**: `glasses_camera` a petición del agente = ask siempre, con consentimiento **en el mismo dispositivo** (card + pinch). El LED de privacidad del hardware es capa física adicional. Sin captura ambiental ni periódica — la regla de captura no-silenciosa del doc 04 aplica reforzada.
- **Voz**: STT on-device; el audio crudo no viaja. El beamforming reduce captura de terceros (aísla la voz del usuario), pero la regla es la misma: capturar solo bajo acción explícita.
- **El HUD muestra datos sensibles en público**: presupuesto de contenido por card + nada de datos de salud en el HUD por default (configurable) — el teléfono es la superficie privada; las gafas, la semi-pública.
- **Blast radius sin cambios**: las gafas no agregan credenciales ni estado propio; comprometer el link BT expone render y callbacks, no el brain ni la key.

## 9. Preguntas abiertas

| # | Pregunta | Default | Cuándo |
|---|---|---|---|
| 1 | **Meta Connect 2026 (sep 23-24)**: sesión dedicada al DAT — ¿0.10, voice invocation, APIs del Neural Band? | Diseñar contra 0.9.0; re-evaluar tras Connect (voice invocation cambiaría el arranque de voz: hoy es botón, podría ser invocación) | Post-Connect, antes de G1 |
| 2 | Vía **web-apps** (HTML/JS) del Display como alternativa/complemento al SDK nativo | Nativo (integración total con el harness); web-apps solo si el review de Meta atasca | G0 |
| 3 | ¿`glasses_show` puede despertar el display para urgencias reales? | No en v1 (etiqueta §4.3); revisar si el SDK lo permite siquiera | G2 |
| 4 | Streaming de cámara continuo ("mira esto conmigo") vs foto puntual | Foto puntual v1 — el streaming multiplica tokens y batería | Post-G2 |
| 5 | TTS: ¿AVSpeechSynthesizer basta o vale una voz mejor (on-device Personal Voice)? | AVSpeech v1 | G1, oyéndolo |
| 6 | ¿Promover `Surface`/`ToolRegistry`/formato skill al spec (doc 03) como contrato para animad? | Sí, tras validar en código G1 | Cierre de G1, PR al doc 03 |

---

*Plan v1 — 2026-09-12. Fuente SDK: doc de transferencia de Relay (hardware-verified, SDK 0.9.0). Cambios de alcance se anotan aquí, estilo ADR ligero.*
