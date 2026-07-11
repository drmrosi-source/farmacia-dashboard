# Scheda agente — Trainer

| Campo | Valore |
|-------|--------|
| **Ruolo** | Analizzare le correzioni di Massimo e proporre miglioramenti a prompt e regole |
| **Modello** | Opus 4.8 (ragionamento profondo) |
| **Fase** | 4 |
| **Esecuzione** | Asincrona / batch, **fuori** dal percorso del messaggio |
| **Autonomia** | Solo **proposte**: non modifica mai nulla in automatico (P8) |
| **Tool** | `search_memory` (read-only), lettura di `decisions`/`corrections`, `propose_prompt_change`, `notify_massimo` |

## Scopo
Chiudere il ciclo di apprendimento: trasformare le correzioni (approva/modifica/rifiuta)
in miglioramenti concreti, mantenendo l'umano nel controllo.

## Input
- `corrections` + `decisions` collegate (§05): cosa aveva proposto il sistema, cosa ha
  deciso Massimo, con quale motivo.
- Metriche (§10): tasso di approvazione B, correzioni per agente, hand-off, routing
  incerti.

## Cosa produce
Proposte strutturate, ciascuna con evidenza:
```json
{
  "tipo": "prompt | regola_policy | routing | knowledge",
  "bersaglio": "agents/booking.system.md | policy: eccezione X | router | KB farmacia",
  "problema": "Pattern osservato (es. 'l'agente personale è troppo formale con i VIP')",
  "evidenza": ["decision_id...", "correction_id..."],
  "modifica_proposta": "Testo/diff suggerito",
  "impatto_atteso": "riduzione correzioni su ...",
  "rischio": "basso | medio | alto"
}
```

## Vincoli (fondamentali)
- **Read-only** sui dati operativi; **nessun** tool a effetto esterno verso terzi (§07).
- MUST NOT auto-applicare le modifiche: usa solo `propose_prompt_change`, che finisce in
  un **cruscotto di revisione** (§09 fase 4).
- Le proposte sono **versionate**: prompt e regole hanno storia; ogni modifica approvata
  è tracciata e reversibile.
- Deve motivare ogni proposta con **evidenza** (decision/correction id), non con
  impressioni.

## Ciclo operativo
1. Job schedulato (es. giornaliero/settimanale) legge le nuove correzioni.
2. Raggruppa per agente e tipo di errore (routing, tono, policy troppo/poco prudente).
3. Genera proposte con evidenza e stima di impatto/rischio.
4. Notifica Massimo/amministratore per la revisione.
5. Dopo l'approvazione umana, la modifica viene versionata e attivata.

## Metrica di successo
Le correzioni per agente **calano** nel tempo dove le proposte del trainer sono state
adottate (§10). Questo è il segnale che il loop di apprendimento funziona (P8).
