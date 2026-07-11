# 04 — Catalogo agenti

## 4.1 Elenco

| Agente | Dominio | Autonomia tipica | Modello | Fase |
|--------|---------|------------------|---------|------|
| **Router** | Classificazione e smistamento | — (interno) | Haiku 4.5 | 1 |
| **Personale "Massimo"** | Networking, collaborazioni, contatti professionali | B (suggerisce) | Sonnet 5 | 1 |
| **Prenotazioni** | Appuntamenti: Calendly, calendario | C su slot standard, B su eccezioni | Sonnet 5 | 2 |
| **Farmacia** | Info su farmacia: orari, servizi, prodotti | C su info pubbliche | Sonnet 5 | 3 |
| **Trainer** | Analizza correzioni, propone miglioramenti | Solo proposte (mai auto-applica) | Opus 4.8 | 4 |
| *Recruiting* | Selezione personale | B | Sonnet 5 | 3 |
| *E-commerce* | Ordini/spedizioni farmacia online | B/C | Sonnet 5 | 3 |

Le schede dettagliate sono nella cartella [`agenti/`](agenti/). Recruiting ed
e-commerce hanno scheda solo abbozzata (fase 3).

## 4.2 Contratto comune di un agente

Ogni agente MUST rispettare questa interfaccia, così che orchestratore e policy engine
lo trattino in modo uniforme e sia facile aggiungerne di nuovi (P2).

### Firma logica

```text
agent.run(message, context, routing) -> Proposal
```

### `Proposal` (output di ogni agente)

```json
{
  "agente": "prenotazioni",
  "reply_text": "Testo proposto per l'interlocutore (può essere null se nessuna risposta)",
  "actions": [
    {
      "tool": "create_booking",
      "args": { "...": "..." },
      "reversibile": true,
      "impatto_esterno": true
    }
  ],
  "categoria_suggerita": "C",
  "handoff": null,
  "note_interne": "Ragionamento sintetico per audit (non inviato all'esterno)",
  "confidenza": 0.0
}
```

- `categoria_suggerita` è **un suggerimento** dell'agente; la parola finale è del
  **policy engine** (§06), che è deterministico e non si fida ciecamente del modello
  (P6).
- `actions` è vuoto per risposte puramente informative.
- Ogni azione dichiara `reversibile` e `impatto_esterno`: input chiave per la policy.

### Vincoli comuni (tutti gli agenti)
1. MUST usare **solo** i tool a loro assegnati nel registry (§07, least privilege).
2. MUST NOT inventare fatti su farmacia, disponibilità o impegni: se un dato non è in
   memoria o via tool, l'agente lo dichiara ignoto e/o propone escalation.
3. MUST NOT fornire consulenza medica/farmacologica personalizzata (§07 vincolo V4).
4. MUST scrivere `reply_text` nella **lingua del contatto** (da `routing.lingua`).
5. MUST rispettare tono e stile definiti nel prompt (vedi `appendici/prompt-template.md`).
6. SHOULD restituire `confidenza` calibrata; bassa confidenza → tende verso categoria B.

## 4.3 Ciclo di vita e registrazione

Gli agenti sono dichiarati in un **registry** centrale:

```json
{
  "prenotazioni": {
    "handler": "agents/booking.ts",
    "model": "claude-sonnet-5",
    "tools": ["get_availability", "create_booking", "cancel_booking", "send_whatsapp"],
    "default_policy": "C",
    "prompt": "prompts/booking.system.md",
    "enabled_from_phase": 2
  }
}
```

Aggiungere un agente = aggiungere una voce al registry + prompt + eventuali tool. **Non**
richiede modifiche all'orchestratore né al router (che apprende il nuovo `agente` come
valore di enum aggiornando il proprio schema di output).

## 4.4 Principi di progettazione dei prompt
- **Ruolo stretto**: il system prompt definisce cosa l'agente è e cosa non è.
- **Regole prima degli esempi**: le regole di sicurezza e policy in testa, non in coda.
- **Output strutturato**: gli agenti producono `Proposal` via tool forzato, non prosa
  da parsare.
- **Nessuna promessa non mantenibile**: l'agente non promette azioni che i suoi tool non
  possono compiere.

Vedi `appendici/prompt-template.md` per gli scheletri.
