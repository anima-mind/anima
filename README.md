# Anima

**Un harness de agentes con forma de mente.** Blueprint portable —agnóstico de lenguaje y de proveedor de LLM— para construir asistentes personales modelados sobre cómo el lenguaje forma la mente humana (Chomsky · Lacan · Kandel), con implementaciones en Swift (edge/móvil) y Go (server/daemon).

> Una mente = LLM (dotación) + harness (desarrollo) + historia (experiencia).
> Este repo es el **contrato compartido**: el spec que todos los runtimes obedecen.

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
| [`anima-ios`](https://github.com/anima-mind/anima-ios) | **Edge** — app iOS/Swift. Consolidación al cargar = sueño; el Otro = el dueño del teléfono. Gafas Meta opcionales como segundo cuerpo ([doc 05](docs/05-meta-glasses-plan.md)). | **Creado** — AnimaKit (SPM) con el invariante de plasticidad + CI verde. Fase 0 pendiente |
| [`animad`](https://github.com/anima-mind/animad) | **Server** — daemon Go y/o servicio REST. Brain compartido, capa intersubjetiva (§B.9). | **Creado** — mismo invariante con valores canónicos idénticos + CI verde. Plan server pendiente |

Regla: los cambios al blueprint se hacen aquí (PR a este repo); los runtimes se actualizan contra él. La [matriz de portabilidad](docs/03-harness-mind-convergence.md) (§C.1) define qué es invariante entre runtimes y qué varía por perfil.

## Estado

- [x] Investigación y spec (docs 01–03, verificados adversarialmente)
- [x] Plan de implementación edge/Swift (doc 04, revisado)
- [x] Plan gafas Meta (doc 05 — opcional: la app funciona con o sin gafas)
- [x] Repos de runtimes creados (públicos, CI verde): [anima-ios](https://github.com/anima-mind/anima-ios) · [animad](https://github.com/anima-mind/animad) — ambos arrancan por el invariante de plasticidad (§C.1) con valores canónicos idénticos en Swift y Go
- [ ] `anima-ios` — Fase 0 (esqueleto que conversa)
- [ ] `anima-ios` — track G (gafas, tras Fase 1)
- [ ] `animad` — plan de implementación server (hermano del doc 04) + Fase 0

---

*Un proyecto de [anima-mind](https://github.com/anima-mind) · Joshua Moreno · 2026*
