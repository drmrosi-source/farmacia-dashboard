# Appendice — Struttura della codebase e convenzioni

Proposta di layout coerente con lo stack (§02.5). Adattabile; serve a rendere immediato
per Claude Code / Codex sapere **dove** mettere le cose.

## Layout proposto

```
/
├── docs/                        # questa specifica (fonte di verità funzionale)
├── src/
│   ├── ingress/
│   │   └── whatsapp_webhook.ts  # endpoint unico (P1): firma, deduplica, normalizza
│   ├── orchestrator/
│   │   ├── loop.ts              # ciclo principale (§03)
│   │   ├── context.ts          # load_context: profilo + storico + memoria (minimizzato)
│   │   └── registry.ts         # registry agenti (nome → handler/model/tools/prompt)
│   ├── router/
│   │   └── router.ts           # classificazione (Haiku, output strutturato)
│   ├── agents/
│   │   ├── personal.ts
│   │   ├── booking.ts
│   │   ├── pharmacy.ts
│   │   └── trainer.ts          # job asincrono
│   ├── policy/
│   │   └── engine.ts           # A/B/C deterministico (§06)
│   ├── tools/
│   │   ├── whatsapp.ts         # send_whatsapp
│   │   ├── calendar.ts         # get_availability/create_booking/... (Calendly+Google)
│   │   ├── memory.ts           # get_*/search_memory/write_memory/update_contact
│   │   ├── control.ts          # notify_massimo/request_approval
│   │   └── index.ts            # schema + registrazione tool, guard di autorizzazione
│   ├── memory/
│   │   ├── schema.sql          # DDL (§05)
│   │   └── migrations/
│   ├── prompts/                # system prompt versionati (§ appendice prompt)
│   │   ├── router.system.md
│   │   ├── personal.system.md
│   │   ├── booking.system.md
│   │   ├── pharmacy.system.md
│   │   └── trainer.system.md
│   ├── control-channel/        # bot Telegram/WhatsApp per approvazioni (§08.3)
│   └── lib/
│       ├── llm.ts              # client Claude, structured output, retry/backoff
│       ├── audit.ts            # scrittura audit_log
│       └── config.ts          # lettura segreti (Vault/env)
├── tests/
│   ├── policy/                 # test deterministici A/B/C (priorità alta)
│   ├── router/                 # casi di classificazione
│   └── tools/                  # idempotenza, schema
└── infra/                      # IaC, cron/queue, deploy
```

## Convenzioni

- **Un tool = un file** con schema di input/output esplicito e flag
  `impatto_esterno`/`reversibile` (§08.6).
- **Agenti senza accesso diretto al DB**: solo via `tools/memory.ts` (§05.3, §07).
- **Prompt versionati**: i file in `prompts/` sono la fonte; le modifiche passano dal
  cruscotto di revisione del trainer (fase 4). Ogni modifica ha commit dedicato.
- **Policy testata per prima**: `tests/policy/` è la rete di sicurezza più importante
  (§06); casi A/B/C ed eccezioni coperti prima di abilitare qualsiasi C in produzione.
- **Segreti solo in `config.ts`** da Vault/env, mai altrove (§07).
- **Idempotenza**: ogni tool a effetto esterno usa una `idempotency_key` derivata da
  `decision_id + tool` (§ appendice schema-messaggi).

## Tool per fase

| Fase | Tool da implementare |
|------|----------------------|
| 1 | `send_whatsapp`, `notify_massimo`, `request_approval`, tool memoria base, guard autorizzazioni |
| 2 | `get_availability`, `create_booking`, `reschedule_booking`, `cancel_booking`, `search_memory`, `write_memory`, `embed` |
| 3 | tool KB farmacia (Google Drive), eventuale adattatore ingresso email (Gmail) |
| 4 | `propose_prompt_change`, cruscotto/versionamento prompt e regole |

## Test minimi per fase 1
- Policy engine: ogni regola A/B/C ed eccezione (§06) → esito atteso.
- Router: set di messaggi etichettati → agente/intent attesi.
- Idempotenza webhook: doppio `provider_message_id` → una sola elaborazione.
- Autorizzazione tool: agente che invoca tool fuori perimetro → rifiutato (§07).
