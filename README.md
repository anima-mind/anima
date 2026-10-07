<p align="center">
  <img src="assets/brand/anima-logo.png" alt="Anima" width="140" />
</p>

<h1 align="center">Anima</h1>

<p align="center">
  <b>Un harness de agentes con forma de mente.</b><br/>
  <i>A mind-shaped agent harness.</i>
</p>

<p align="center">
  <code>una mente = LLM (dotación) + harness (desarrollo) + historia (experiencia)</code>
</p>

<p align="center">
  <a href="#estado"><img src="https://img.shields.io/badge/estado-pruebas%20de%20campo-6aa8ff" alt="estado: pruebas de campo"/></a>
  <a href="https://github.com/anima-mind/anima-ios"><img src="https://img.shields.io/badge/runtime-iOS%20%C2%B7%20Swift-1f2a3a" alt="iOS"/></a>
  <a href="https://github.com/anima-mind/animad"><img src="https://img.shields.io/badge/runtime-server%20%C2%B7%20Go-1f2a3a" alt="Go"/></a>
</p>

Blueprint portable —agnóstico de lenguaje y de proveedor de LLM— para construir asistentes personales modelados sobre cómo el lenguaje forma la mente humana (**Chomsky · Lacan · Kandel**), con implementaciones en Swift (edge/móvil) y Go (server/daemon). Este repo es el **contrato compartido**: el spec que todos los runtimes obedecen.

## Así se ve

<p align="center">
  <img src="assets/screenshots/chat.png" alt="Chat" width="22%" />
  <img src="assets/screenshots/propuesta.png" alt="Propuesta proactiva" width="22%" />
  <img src="assets/screenshots/memoria.png" alt="Memoria consolidada" width="22%" />
  <img src="assets/screenshots/mente.png" alt="La mente: plasticidad y noches" width="22%" />
</p>
<p align="center"><sub>Chat · propuesta proactiva ligada a una meta · memoria destilada en la noche · la mente (plasticidad y ciclos de sueño). Capturas del runtime iOS en desarrollo.</sub></p>

## Cómo piensa

| Idea | En Anima |
|---|---|
| **El sueño consolida** (Kandel) | Cada noche, mientras el teléfono carga, un ciclo destila la conversación en memorias durables, reconsolida las viejas y reflexiona. |
| **La plasticidad decae** | `p(n) = 0.05 + 0.95·e^(−n/30)`: la identidad se moldea libre al principio y se estabiliza con las noches; pasada la infancia, cambiar quién es exige tu aprobación. |
| **El deseo del Otro** (Lacan) | Tus metas —declaradas o inferidas— motivan propuestas y seguimientos proactivos, acotados para no volverse ruido. |
| **La gramática como estructura** (Chomsky) | Tools tipadas con permisos explícitos y skills que se aprenden con la práctica: lo que hace es verificable, no solo lo que dice. |

## Los documentos

| # | Documento | Qué es |
|---|---|---|
| 01 | [Agent Harness Manual](docs/01-agent-harness-manual.md) | Anatomía de un harness (loop, contexto, tools, memoria, fallos, seguridad) + comparativa de 8 harnesses reales. Investigación con verificación adversarial. |
| 02 | [Language & Mind Formation](docs/02-language-mind-formation.md) | Cómo el lenguaje forma la mente: Chomsky (gramática generativa), Lacan (el sujeto y el significante), Kandel (neurociencia de la memoria) + convergencias documentadas. |
| 03 | [Harness–Mind Convergence](docs/03-harness-mind-convergence.md) | **El spec.** Manifiesto + arquitectura por capas: 10 subsistemas (B.1–B.10), dos perfiles de runtime (edge/server), matriz de portabilidad, evals falsables, checklist de 12 pasos. |
| 04 | [Swift Implementation Plan](docs/04-swift-implementation-plan.md) | Plan de implementación del perfil **edge** como app iOS: contratos Swift, esquema GRDB, fases 0→4, costos, seguridad. |
| 05 | [Meta Glasses Plan](docs/05-meta-glasses-plan.md) | Las gafas como **segundo cuerpo opcional** de la misma mente (DAT SDK 0.9.0, hardware-verified vía Relay): HUD, voz manos-libres, tools nuevas, track G0–G2, y el **framework de extensibilidad** (tools/skills/surfaces) para móvil y gafas. |

Léelos en orden si llegas de cero; el 03 es la referencia normativa.

## Los runtimes (repos hermanos)

| Repo | Perfil | Estado |
|---|---|---|
| [`anima-ios`](https://github.com/anima-mind/anima-ios) | **Edge** — app iOS/Swift. Consolidación al cargar = sueño; el Otro = el dueño del teléfono. Gafas Meta opcionales como segundo cuerpo ([doc 05](docs/05-meta-glasses-plan.md)). | **En pruebas de campo** (TestFlight interno) — fases 0–4 del plan implementadas: conversación, memoria y sueño, identidad con plasticidad, deseo/metas, recordatorios y seguimientos proactivos, modo 100 % on-device (Apple Foundation Models) y gafas Meta |
| [`animad`](https://github.com/anima-mind/animad) | **Server** — daemon Go y/o servicio REST. Brain compartido, capa intersubjetiva (§B.9). | **Creado** — mismo invariante con valores canónicos idénticos + CI verde. Plan server pendiente |

Regla: los cambios al blueprint se hacen aquí (PR a este repo); los runtimes se actualizan contra él. La [matriz de portabilidad](docs/03-harness-mind-convergence.md) (§C.1) define qué es invariante entre runtimes y qué varía por perfil.

## Estado

- [x] Investigación y spec (docs 01–03, verificados adversarialmente)
- [x] Plan de implementación edge/Swift (doc 04, revisado)
- [x] Plan gafas Meta (doc 05 — opcional: la app funciona con o sin gafas)
- [x] Repos de runtimes creados (públicos, CI verde): [anima-ios](https://github.com/anima-mind/anima-ios) · [animad](https://github.com/anima-mind/animad) — ambos arrancan por el invariante de plasticidad (§C.1) con valores canónicos idénticos en Swift y Go
- [x] `anima-ios` — fases 0–4 (conversa, recuerda, duerme, desea) + capa proactiva + modo on-device
- [x] `anima-ios` — track G (gafas Meta, DAT SDK 1.0)
- [ ] `anima-ios` — widgets, CarPlay y App Store
- [ ] `animad` — plan de implementación server (hermano del doc 04) + Fase 0

---

*Un proyecto de [anima-mind](https://github.com/anima-mind) · Joshua Moreno · 2026*
