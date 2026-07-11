# 10 — Osservabilità e operatività

Serve a rispondere a tre domande in ogni momento: *funziona?*, *quanto costa?*, *perché
ha fatto così?*.

## 10.1 Logging strutturato

- Ogni tappa del ciclo (§03) emette un log JSON con `request_id` (correlazione),
  `contact_id`, `agente`, `categoria_policy`, latenza e esito.
- I contenuti dei messaggi nei log sono **redatti/minimizzati** (§07): si logga metadato
  e riferimento, non necessariamente il testo integrale.
- Livelli: `info` (flusso normale), `warn` (fallback, bassa confidenza), `error`
  (tool falliti, provider giù).

## 10.2 Tracciabilità delle decisioni

La catena è ricostruibile end-to-end dalla memoria:

```
message → decision (router + proposal + categoria) → [correction] → audit_log (azioni)
```

Questo permette di spiegare **perché** un messaggio ha ricevuto una certa risposta, e di
alimentare il trainer (§ fase 4).

## 10.3 Metriche chiave

| Metrica | Perché |
|---------|--------|
| Messaggi/giorno per intent e agente | Carico e distribuzione |
| % categoria A/B/C | Bilanciamento autonomia vs controllo |
| Tasso di approvazione B (approva/modifica/rifiuta) | Qualità delle proposte; input al trainer |
| Correzioni per agente nel tempo | Apprendimento (deve calare, P8) |
| Accuratezza router (via correzioni/hand-off) | Salute del routing |
| Latenza p50/p95 per tappa | SLA (§02.6) |
| Costo LLM per messaggio (router vs agente) | Verifica P3 (router leggero) |
| Tool error rate per integrazione | Salute di WhatsApp/Calendly/Google |

## 10.4 Gestione degli errori

| Errore | Comportamento |
|--------|---------------|
| Provider LLM non risponde | Retry con backoff; se persiste, categoria B con nota "sistema degradato" e notifica a Massimo. |
| Tool a effetto esterno fallisce | Nessun invio parziale; si registra `audit_log.esito=errore`; si notifica se rilevante. |
| Webhook malformato/non firmato | 4xx, log `warn`, nessuna elaborazione. |
| Router a bassa confidenza | Fallback prudente (`personale`, §03) + flag. |
| Doppio webhook (retry Meta) | Deduplica su `provider_message_id` (§05). |

Principio: **fallire chiuso** — in caso di dubbio o guasto, non si agisce verso terzi;
si degrada verso B/A e si avvisa Massimo (P7, §07 V7).

## 10.5 Controllo dei costi

- Router su Haiku e contesto minimizzato (§03) tengono basso il costo di base.
- Budget/allarme per spesa LLM giornaliera; il superamento alza la soglia verso B
  (meno azioni autonome, più batch) e notifica l'amministratore.
- I job del trainer (Opus) girano in batch fuori dal percorso critico (§08.5).

## 10.6 Operatività

- **Deploy**: infrastruttura come codice; migrazioni DB versionate (§05).
- **Segreti**: gestiti in Vault (§07); rotazione documentata.
- **Backup**: backup regolari di Postgres; l'intero stato è ricostruibile dalla memoria
  (§02.6).
- **Runbook**: procedure per "provider giù", "webhook non arrivano", "approvazioni
  bloccate", "purge dati su richiesta interessato".
- **Kill switch**: interruttore che forza **tutto in categoria B/A** (nessuna azione
  autonoma) in caso di comportamento anomalo, attivabile da Massimo/amministratore.
