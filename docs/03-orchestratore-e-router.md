# 03 — Orchestratore e Router

## 3.1 Orchestratore

L'orchestratore è il cuore operativo e l'**unico punto d'ingresso** (P1). Non contiene
logica di dominio: coordina.

### Responsabilità
- Esporre il webhook di ingresso e verificarne l'autenticità.
- Normalizzare, deduplicare e persistere i messaggi in ingresso.
- Costruire il **contesto minimo** per router e agenti (P5, minimizzazione dati §07).
- Invocare router → agente → policy engine → esecuzione tool.
- Gestire il flusso di approvazione umana (categoria B).
- Registrare decisioni, azioni e correzioni.

### Pseudocodice del ciclo principale

```text
on_inbound(webhook):
    verify_signature(webhook)                      # §07
    msg = normalize(webhook)                        # → InboundMessage
    if already_processed(msg.provider_message_id):  # deduplica
        return 200
    contact = resolve_contact(msg.from)
    context = load_context(contact)                 # profilo + ultimi N msg + note
    persist_inbound(msg, contact)

    routing = router.classify(msg, context)         # §3.2
    agent   = select_agent(routing.agente)
    proposal = agent.run(msg, context, routing)     # proposta risposta/azione

    decision = policy_engine.evaluate(              # §06
                   proposal, contact, routing)
    switch decision.category:
        case A: register_only(proposal, decision)               # non inviare
        case B: request_human_approval(proposal, decision)      # §2.4
        case C: execute(proposal.actions)                       # autonomo
                maybe_notify(decision.notify_level)
    persist_decision(routing, proposal, decision)
    return 200
```

> L'orchestratore risponde **200 a Meta il prima possibile** e completa il lavoro
> pesante in modo asincrono quando la latenza rischia di superare i timeout del
> webhook. In pratica: accoda ed elabora, oppure elabora inline se sotto soglia.

### Costruzione del contesto (`load_context`)
- Profilo del contatto (categoria, note, storico decisioni rilevanti).
- Ultimi **N = 10** messaggi della conversazione (configurabile).
- Fino a **K = 5** ricordi semantici pertinenti (ricerca vettoriale su `memory`, §05).
- Stato conversazione (es. "in attesa di conferma orario").

Il contesto è **il più piccolo possibile** che permetta una buona risposta: meno dati
personali passano all'LLM, meglio è (§07).

## 3.2 Router

### Obiettivo
Decidere **a chi** va il messaggio e **con quale inquadramento**, spendendo pochissimo
(P3). Non risponde al contatto.

### Input
- Testo del messaggio corrente (e tipo: testo/immagine/audio/…).
- Contesto sintetico: categoria del contatto, oggetto/ultimo intent della
  conversazione, eventuale stato aperto.

### Output (strutturato, obbligatorio)

```json
{
  "agente": "personale | prenotazioni | farmacia | nessuno",
  "intent": "networking | collaborazione | prenotazione | info_farmacia | spam | personale_urgente | altro",
  "categoria_contatto": "sconosciuto | networking | cliente_farmacia | fornitore | familiare_amico | vip",
  "urgenza": "bassa | media | alta",
  "lingua": "it | en | ...",
  "confidenza": 0.0,
  "motivazione": "breve stringa per audit"
}
```

Il router MUST produrre **solo** questo JSON (structured output / tool forzato). Nessun
testo libero verso l'esterno.

### Regole di smistamento (baseline)

| Condizione | Agente |
|-----------|--------|
| Richiesta di appuntamento / disponibilità / conferma orario | `prenotazioni` |
| Domanda su orari farmacia, disponibilità prodotto, servizi, sede | `farmacia` |
| Networking, proposta di collaborazione, contatto professionale | `personale` |
| Spam evidente / promozioni non richieste | `nessuno` (categoria A, §06) |
| Contatto personale/urgente da familiare o VIP | `personale` con `urgenza: alta` |
| Bassa confidenza (< 0.5) | `personale` (fallback prudente) + flag per revisione |

### Gestione dell'incertezza
- Se `confidenza < 0.5`, si sceglie il **fallback prudente** (`personale`) e si marca la
  decisione per il trainer: routing incerti sono candidati primari di miglioramento.
- Se il messaggio contiene più intenti, il router sceglie l'intento **dominante** e
  annota gli altri in `motivazione`; l'agente potrà richiamare un altro agente
  (hand-off) se necessario.

### Hand-off tra agenti
Un agente può restituire un segnale `handoff: <altro_agente>` quando durante
l'elaborazione emerge che la competenza è di un altro (es. l'agente personale scopre
che il contatto vuole in realtà prenotare). L'orchestratore ri-esegue il ciclo con
l'agente indicato, **senza** ripassare dal router, e registra l'hand-off. Limite: max 1
hand-off per messaggio, per evitare loop.

### Perché un modello leggero
Il router gira su **ogni** messaggio: usarlo con Haiku mantiene costo e latenza bassi
(P3). Se un caso è troppo ambiguo per Haiku, il fallback prudente e la revisione umana
coprono il rischio senza alzare il costo di base.

## 3.3 Selezione dell'agente
`select_agent` mappa la stringa `agente` all'implementazione registrata. Gli agenti
sono registrati in un **registry** (nome → handler + tool consentiti + modello). Questo
rende banale aggiungere nuovi agenti in fase 3 (recruiting, e-commerce) senza toccare
l'orchestratore.
