# 02 — Architettura

## 2.1 Vista d'insieme

```
                        ┌──────────────────────────────────────────────┐
   Canali esterni       │                 PIATTAFORMA                    │
                        │                                                │
  WhatsApp ──webhook──► │  ┌────────────┐   ┌──────────┐                 │
  (Meta Cloud API)      │  │ Ingress /  │   │  Router  │  (modello        │
                        │  │ Orchestr.  ├──►│ (Haiku)  │   leggero)       │
                        │  │  (P1)      │   └────┬─────┘                  │
                        │  └─────┬──────┘        │ intent + agente        │
                        │        │               ▼                        │
                        │        │        ┌─────────────────────────┐     │
                        │        │        │   Agente specializzato  │     │
                        │        │        │  (personale / prenot. / │     │
                        │        │        │   farmacia / ...)       │     │
                        │        │        └───────┬─────────────────┘     │
                        │        │                │ usa strumenti          │
                        │   ┌────▼───────┐   ┌─────▼──────┐  ┌───────────┐  │
                        │   │  Policy    │   │  Toolbox   │  │  Memoria   │  │
                        │   │  engine    │   │ (P4)       │  │ condivisa  │  │
                        │   │ (A/B/C)    │   │ tool puri  │  │  (P5)      │  │
                        │   └────┬───────┘   └─────┬──────┘  └─────┬─────┘  │
                        │        │                 │               │        │
                        └────────┼─────────────────┼───────────────┼────────┘
                                 │                 │               │
                    approvazione │        integrazioni│      Postgres+pgvector
                    umana (B)    ▼                 ▼
                          Massimo (WhatsApp/    Calendly, Google
                          Telegram di controllo) Calendar, ...

        ┌──────────────────────────────────────────────────────────┐
        │  Agente Trainer (batch/asincrono) — legge correzioni,     │
        │  propone miglioramenti a prompt/regole → revisione umana  │
        └──────────────────────────────────────────────────────────┘
```

## 2.2 Componenti

| Componente | Responsabilità | Note |
|-----------|----------------|------|
| **Ingress / Orchestratore** | Riceve webhook, normalizza il messaggio, deduplica, carica il contesto dalla memoria, invoca il router, poi l'agente scelto, applica il policy engine, persiste tutto. | Unico punto d'ingresso (P1). Stateless: lo stato è in memoria condivisa. |
| **Router** | Classifica intent + urgenza + categoria contatto, sceglie l'agente. | Modello leggero (P3). Output strutturato (§03). |
| **Agenti** | Generano la risposta/azione nel loro dominio. | Prompt dedicati, tool limitati (P2, P4). |
| **Policy engine** | Decide se l'output proposto è auto-eseguibile (C), da approvare (B) o da non inviare (A). | Deterministico, non affidato al modello (P6). Vedi §06. |
| **Toolbox** | Implementazioni concrete degli strumenti. | Funzioni idempotenti dove possibile; ogni tool ha uno schema. |
| **Memoria condivisa** | Persistenza di contatti, conversazioni, messaggi, decisioni, correzioni, memoria semantica. | Postgres + pgvector (§05). |
| **Canale di controllo** | Il modo in cui la piattaforma parla a Massimo (approvazioni, notifiche). | WhatsApp dedicato o Telegram bot (§08). |
| **Trainer** | Processo asincrono che analizza le correzioni e propone miglioramenti. | Non in linea sul percorso del messaggio. |

## 2.3 Flusso di un messaggio in ingresso

1. **Ricezione** — Meta invia il webhook all'endpoint dell'orchestratore. Verifica
   firma (`X-Hub-Signature-256`) e struttura.
2. **Normalizzazione** — Il payload viene trasformato nel formato interno
   `InboundMessage` (vedi `appendici/schema-messaggi.md`).
3. **Deduplica** — Si scarta il messaggio se il suo `provider_message_id` è già stato
   processato (idempotenza: Meta può ritrasmettere).
4. **Risoluzione contatto** — Si cerca/crea il `contact` a partire dal numero e si
   carica il profilo + gli ultimi N messaggi della conversazione.
5. **Routing** — Il router riceve messaggio + contesto sintetico e restituisce
   `{intent, agente, categoria_contatto, urgenza, confidenza}`.
6. **Esecuzione agente** — L'agente selezionato produce una **proposta di risposta**
   e/o una **proposta di azione** (tool call), non ancora eseguite.
7. **Applicazione policy** — Il policy engine determina la classe A/B/C in base a
   contenuto, contatto e azione:
   - **A**: nessun invio; si registra e (se previsto) si notifica Massimo in modo
     silenzioso.
   - **B**: si prepara un **suggerimento** e lo si invia a Massimo per approvazione.
     L'azione parte solo dopo l'ok.
   - **C**: azione eseguita in autonomia; si registra e si notifica secondo soglia.
8. **Esecuzione azioni** — I tool approvati vengono eseguiti (invio WhatsApp,
   creazione booking, ...). Tutto è idempotente e loggato.
9. **Persistenza** — Messaggio in ingresso, decisione del router, output dell'agente,
   classe policy, esito azioni e stato approvazione vengono scritti in memoria.
10. **Osservabilità** — Metriche e trace (§10).

## 2.4 Flusso di approvazione (categoria B)

```
Agente → proposta → Policy(B) → notifica a Massimo (canale di controllo)
                                   │
                Massimo: ✅ Approva / ✏️ Modifica / ❌ Rifiuta
                                   │
      ✅ → esegui azione            ✏️ → esegui con testo modificato (registra correzione)
      ❌ → non inviare (registra correzione + motivo se fornito)
```

Ogni esito (approva/modifica/rifiuta) è una **correzione** registrata per il trainer.

## 2.5 Stack tecnologico (proposto)

| Livello | Tecnologia | Motivazione |
|---------|-----------|-------------|
| Ingress + orchestrazione | **Supabase Edge Functions (Deno/TypeScript)** oppure servizio Node dedicato | Vicino al DB, deploy semplice, webhook nativi. [DA DECIDERE] Edge Functions vs servizio Node long-running per i job del trainer. |
| Modelli LLM | **Claude API (Anthropic)** — Haiku 4.5 per il router, Sonnet 5 per gli agenti conversazionali, Opus 4.8 per il trainer | Router economico (P3); agenti bilanciati; trainer con ragionamento profondo. |
| Persistenza | **Postgres (Supabase)** + **pgvector** | Relazionale per profili/decisioni, vettoriale per memoria semantica. |
| Coda/async | Supabase Queues o cron (`pg_cron`) per il trainer e le notifiche differite | Il percorso del messaggio è sincrono; trainer e reminder sono asincroni. |
| Canale in ingresso | **WhatsApp Business Cloud API (Meta)** | Requisito di prodotto. |
| Calendario | **Calendly API** + **Google Calendar** | §08. |
| Canale di controllo | WhatsApp dedicato o **Telegram Bot** | §08 — [DA DECIDERE]. |
| Segreti | Supabase Vault / variabili d'ambiente cifrate | §07. |
| Osservabilità | Log strutturati + tabella `audit_log` + metriche | §10. |

> Nota: lo stack è coerente con i connettori già disponibili nell'ambiente (Supabase,
> Google Calendar, Gmail). Le scelte marcate [DA DECIDERE] vanno chiuse in fase 1.

## 2.6 Requisiti non funzionali

| Requisito | Target |
|-----------|--------|
| Latenza risposta (router→agente→invio) | < 5 s p95 per risposte testuali semplici |
| Costo per messaggio | Router < ~1/10 del costo di un turno agente (P3) |
| Idempotenza | Ogni tool a effetto esterno è idempotente o protetto da chiave |
| Disponibilità | Best-effort; nessun messaggio perso (webhook con retry Meta + deduplica) |
| Recuperabilità | Stato interamente ricostruibile dalla memoria condivisa |
| Privacy | Dati personali trattati secondo §07; minimizzazione del contesto inviato agli LLM |
