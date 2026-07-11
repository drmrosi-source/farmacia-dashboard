# Specifica Tecnica — Piattaforma di Agenti AI Personali

**Versione:** 2.0 (specifica tecnica dettagliata)
**Base di partenza:** Specifica Funzionale v1.0 del 10 luglio 2026
**Data:** 11 luglio 2026
**Committente:** Massimo (Dr. Rosi)
**Stato:** Bozza operativa — pronta per lo sviluppo

---

## Scopo di questo documento

Questo è il documento tecnico di riferimento per la costruzione della *Piattaforma di
Agenti AI Personali*. Espande la specifica funzionale v1.0 in una serie di moduli
autoconsistenti che descrivono architettura, dati, agenti, policy, sicurezza,
integrazioni e roadmap con un livello di dettaglio sufficiente a:

1. guidare lo sviluppo tecnico incrementale;
2. essere dato in pasto a strumenti di coding assistito (Claude Code, Codex) come
   contesto operativo, un modulo alla volta;
3. servire da contratto condiviso tra le parti (prodotto, sviluppo, sicurezza).

Ogni modulo è pensato per essere letto in autonomia. I riferimenti incrociati usano il
numero del modulo (es. *vedi §05*).

---

## Struttura dei moduli

| # | Modulo | Contenuto |
|---|--------|-----------|
| 01 | [Visione e principi](01-visione-e-principi.md) | Obiettivi, principi architetturali, glossario |
| 02 | [Architettura](02-architettura.md) | Componenti, flusso di un messaggio, stack tecnologico |
| 03 | [Orchestratore e Router](03-orchestratore-e-router.md) | Punto d'ingresso unico, classificazione, smistamento |
| 04 | [Catalogo agenti](04-catalogo-agenti.md) | Indice degli agenti, contratto comune, ciclo di vita |
| 05 | [Memoria e modello dati](05-memoria-e-dati.md) | Schema Postgres, memoria condivisa, memoria semantica |
| 06 | [Policy conversazionali](06-policy-conversazionali.md) | Categorie A/B/C, criteri, eccezioni, escalation |
| 07 | [Sicurezza e privacy](07-sicurezza-e-privacy.md) | Autorizzazioni, segreti, GDPR, audit |
| 08 | [Integrazioni esterne](08-integrazioni.md) | WhatsApp, Calendly, Google, notifiche umane |
| 09 | [Roadmap e fasi](09-roadmap.md) | Fasi 1–4, deliverable, criteri di uscita |
| 10 | [Osservabilità e operatività](10-osservabilita.md) | Log, metriche, costi, gestione errori |

### Schede agente (cartella `agenti/`)

| File | Agente |
|------|--------|
| [`agenti/router.md`](agenti/router.md) | Router — classificazione e smistamento |
| [`agenti/personale-massimo.md`](agenti/personale-massimo.md) | Agente personale "Massimo" — filtro networking |
| [`agenti/prenotazioni.md`](agenti/prenotazioni.md) | Agente prenotazioni — Calendly/calendario |
| [`agenti/farmacia.md`](agenti/farmacia.md) | Agente informativo farmacia |
| [`agenti/trainer.md`](agenti/trainer.md) | Agente trainer — apprendimento dalle correzioni |

### Appendici (cartella `appendici/`)

| File | Contenuto |
|------|-----------|
| [`appendici/schema-messaggi.md`](appendici/schema-messaggi.md) | Formato JSON dei messaggi tra componenti |
| [`appendici/prompt-template.md`](appendici/prompt-template.md) | Scheletri di prompt di sistema per ogni agente |
| [`appendici/struttura-repo.md`](appendici/struttura-repo.md) | Layout della codebase e convenzioni |

---

## Come usare questo documento con Claude Code / Codex

- **Non** incollare tutto in un'unica sessione. Fornisci il modulo pertinente al task
  (es. per implementare la memoria, dai §05 + `appendici/schema-messaggi.md`).
- Ogni scheda agente contiene già input/output, strumenti e vincoli: è
  autosufficiente per generare l'implementazione di quel singolo agente.
- Le decisioni ancora aperte sono marcate con **[DA DECIDERE]** e vanno chiuse prima
  di avviare la fase relativa.

---

## Convenzioni

- **MUST / SHOULD / MAY**: requisiti obbligatori / consigliati / opzionali.
- **[DA DECIDERE]**: scelta di prodotto o tecnica ancora aperta.
- **[FASE N]**: il requisito appartiene alla fase N della roadmap (§09).
