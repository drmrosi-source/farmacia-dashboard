# Appendice — Schema dei messaggi tra componenti

Formati JSON di riferimento per l'implementazione. Sono contratti interni: cambiarli
richiede aggiornare i componenti che li producono/consumano.

## `InboundMessage` (normalizzato dall'orchestratore)

```json
{
  "provider": "whatsapp",
  "provider_message_id": "wamid.HBg...",
  "from": "+391234567890",
  "to": "+39PHARMACYNUMBER",
  "timestamp": "2026-07-11T09:30:00Z",
  "tipo": "text",
  "contenuto": "Buongiorno, vorrei un appuntamento",
  "media_url": null,
  "raw": { "...": "payload originale Meta, per debug/audit" }
}
```

## `RouterOutput`

```json
{
  "agente": "prenotazioni",
  "intent": "prenotazione",
  "categoria_contatto": "cliente_farmacia",
  "urgenza": "media",
  "lingua": "it",
  "confidenza": 0.82,
  "motivazione": "richiesta esplicita di appuntamento"
}
```

## `Proposal` (output di ogni agente)

```json
{
  "agente": "prenotazioni",
  "reply_text": "Buongiorno! Ho disponibilità martedì 10:00 o mercoledì 15:30. Quale preferisce?",
  "actions": [
    { "tool": "get_availability", "args": { "range": "2026-07-14/2026-07-16", "durata": 30 },
      "reversibile": true, "impatto_esterno": false }
  ],
  "categoria_suggerita": "C",
  "handoff": null,
  "note_interne": "Slot proposti su disponibilità reale; nessun impegno assunto.",
  "confidenza": 0.9
}
```

## `PolicyDecision` (output del policy engine)

```json
{
  "category": "C",
  "notify_level": "digest",
  "motivazione": "Info su disponibilità reale, contatto cliente_farmacia, azione reversibile.",
  "blocca_azioni": false
}
```

## `ApprovalRequest` (categoria B → canale di controllo)

```json
{
  "decision_id": "uuid",
  "contatto": "+391234567890 (Mario Rossi, networking)",
  "testo_proposto": "Grazie per la proposta, ne parliamo volentieri...",
  "azioni": [{ "tool": "send_whatsapp", "descrizione": "Invia risposta al contatto" }],
  "opzioni": ["approva", "modifica", "rifiuta"]
}
```

## `ApprovalResponse` (da Massimo, autenticata — §07 V5)

```json
{
  "decision_id": "uuid",
  "esito": "modifica",
  "testo_corretto": "Grazie! Sono interessato, sentiamoci la prossima settimana.",
  "motivo": "troppo formale"
}
```

## `ToolCallResult`

```json
{
  "tool": "create_booking",
  "ok": true,
  "risultato": { "booking_id": "cal_abc123", "quando": "2026-07-14T10:00:00Z" },
  "idempotency_key": "decision:uuid:create_booking",
  "audit_id": "uuid"
}
```

## Note trasversali
- Tutti i timestamp in **ISO 8601 UTC**.
- I numeri di telefono in formato **E.164**.
- Gli enum (`agente`, `intent`, `categoria_contatto`, `categoria`) sono definiti in §03 e
  §06; router e agenti usano **output strutturato/tool forzato** per garantirli.
- `raw` dell'`InboundMessage` è conservato per audit ma **non** passato agli LLM (§07
  minimizzazione).
