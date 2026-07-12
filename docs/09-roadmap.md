# 09 — Roadmap e fasi

La roadmap è incrementale: ogni fase produce qualcosa di **utile e sicuro** in
produzione, con autonomia che cresce solo dopo che la fase precedente si è dimostrata
affidabile (P7, P8).

## Fase 1 — Fondamenta: WhatsApp, Router, Agente personale, base

**Obiettivo:** ricevere messaggi WhatsApp, classificarli e far preparare i suggerimenti
dall'agente personale, con Massimo che approva tutto (default categoria B).

**Deliverable**
- [ ] Orchestratore + webhook WhatsApp (ingresso, firma, deduplica, normalizzazione).
- [ ] Memoria condivisa: schema base (`contacts`, `conversations`, `messages`,
      `decisions`, `corrections`, `audit_log`).
- [ ] Router (Haiku) con output strutturato (§03).
- [ ] Agente personale "Massimo" in modalità **B** (suggerisce, non invia).
- [ ] Policy engine con categorie A/B/C (regole baseline §06).
- [ ] Canale di controllo su **WhatsApp** (chat riservata) con pulsanti interattivi
      approva/modifica/rifiuta + template di servizio per la finestra 24h (§08.3).
- [ ] Tool `send_whatsapp`, `notify_massimo`, tool di memoria di base.
- [ ] Osservabilità minima: log strutturati + `audit_log` (§10).

**Criteri di uscita**
- Ogni messaggio in ingresso viene classificato e produce una proposta.
- Nessun invio verso terzi avviene senza approvazione di Massimo.
- Le correzioni di Massimo vengono registrate.

**Rischi/decisioni da chiudere:** **topologia dei numeri** (opzione A/B, §08.1.0 — da
decidere per primo perché condiziona l'onboarding), approvazione dei template di servizio
WhatsApp da parte di Meta (§08.3), retention (§07), hosting/regione (§07).

## Fase 2 — Prenotazioni e memoria

**Obiettivo:** l'agente prenotazioni gestisce appuntamenti; la memoria diventa
semantica.

**Deliverable**
- [ ] Agente prenotazioni con tool Calendly + Google Calendar (§08.2).
- [ ] Proposta di slot su disponibilità reale in **categoria C**; cancellazioni in B.
- [ ] Memoria semantica (`memory` + pgvector), tool `search_memory`/`write_memory`,
      riassunti progressivi (§05).
- [ ] Estensione del router per l'intent `prenotazione` e hand-off (§03).
- [ ] Digest delle azioni autonome verso Massimo (§06.5).

**Criteri di uscita**
- Un appuntamento standard viene proposto/creato senza intervento umano, senza
  doppie prenotazioni.
- Il contesto degli agenti include ricordi semantici pertinenti.

## Fase 3 — Nuovi agenti (recruiting, farmacia, e-commerce)

**Obiettivo:** ampliare i domini riusando l'infrastruttura (registry, §04).

**Deliverable**
- [ ] Agente **farmacia**: info pubbliche (orari, servizi, prodotti) in categoria C;
      knowledge base da Google Drive (§08.4). Vincolo V4 (§07): niente consulenza medica.
- [ ] Agente **recruiting** (B): screening iniziale candidature.
- [ ] Agente **e-commerce** (B/C): stato ordini/spedizioni farmacia online.
- [ ] Eventuale adattatore di ingresso **email (Gmail)** riusando l'orchestratore (P1).

**Criteri di uscita**
- Aggiungere un agente richiede solo registry + prompt + tool, senza toccare
  orchestratore/router core.

## Fase 4 — Ciclo di apprendimento

**Obiettivo:** chiudere il loop: le correzioni migliorano il sistema.

**Deliverable**
- [ ] Agente **trainer** (Opus, offline): analizza `corrections`/`decisions`, individua
      pattern di errore (routing sbagliati, toni non graditi, policy troppo/poco
      prudenti).
- [ ] Tool `propose_prompt_change`: il trainer produce **proposte** di modifica a
      prompt/regole, mai auto-applicate (P8, §07).
- [ ] Cruscotto di revisione: Massimo/amministratore approva o scarta le proposte;
      versionamento dei prompt e delle regole.
- [ ] Metriche di miglioramento: tasso di approvazione B, correzioni per agente,
      accuratezza del router nel tempo (§10).

**Criteri di uscita**
- Esiste evidenza misurabile che il tasso di correzioni cala nel tempo su almeno un
  agente, grazie a modifiche proposte dal trainer e approvate.

## Sintesi visiva

```
F1  Ingresso WA + Router + Personale(B) + Policy + Memoria base + Controllo
      │  (tutto approvato da Massimo)
F2  Prenotazioni(C su slot) + Memoria semantica + Digest
      │  (prima autonomia reale, ristretta e reversibile)
F3  Farmacia + Recruiting + E-commerce (+ email)   → scala per domini
      │
F4  Trainer + proposte di miglioramento + revisione → il sistema impara
```

## Principi trasversali alla roadmap
- **Autonomia guadagnata**: C si concede solo dove l'errore è a basso impatto e
  reversibile, e solo dopo osservazione in B.
- **Ogni fase è spedibile**: niente big-bang; valore in produzione a ogni passo.
- **Le decisioni [DA DECIDERE] hanno un proprietario e una fase**: vanno chiuse prima di
  iniziare la fase che le richiede.
