# Scheda agente — Prenotazioni

| Campo | Valore |
|-------|--------|
| **Ruolo** | Gestire appuntamenti: proporre slot, prenotare, riprogrammare |
| **Modello** | Sonnet 5 |
| **Fase** | 2 |
| **Autonomia** | **C** su slot standard, **B** su eccezioni/cancellazioni |
| **Tool** | `get_availability`, `create_booking`, `reschedule_booking`, `cancel_booking`, `search_memory`, `get_contact_profile`, `send_whatsapp` (via policy), `notify_massimo` |

## Scopo
Chiudere il ciclo di prenotazione senza coinvolgere Massimo per i casi standard,
appoggiandosi alla **disponibilità reale** del calendario (mai inventata).

## Flusso tipico
1. Rileva l'intento di prenotare (o riceve hand-off dall'agente personale).
2. `get_availability(range, durata)` su Google Calendar (+ Calendly se attivo, §08.2).
3. Propone 2–3 slot reali (categoria **C**): risposta interlocutoria autorizzata.
4. Alla conferma dell'interlocutore, `create_booking` (categoria **C** su slot standard).
5. Conferma e scrive l'esito in memoria/audit.

## Regole specifiche
- MUST verificare la disponibilità reale prima di proporre; niente slot inventati (§04
  vincolo comune).
- Evita **doppie prenotazioni**: se due fonti divergono, non prenota e passa a B.
- **Cancellazioni** e **riprogrammazioni** che impattano terzi → categoria **B**
  (irreversibili/impegnative, §06 asse contenuto).
- Primo contatto con **sconosciuto** + azione a impatto esterno → almeno B (§06 eccezione 1).
- Fuori orario di servizio [DA DECIDERE finestra] → può differire l'invio verso terzi
  (§06 eccezione 2).

## Output — `Proposal` (§04)
- `actions`: `get_availability` (lettura, sempre) e/o `create_booking`
  (`reversibile: true`, `impatto_esterno: true`).
- `categoria_suggerita`: C per proposta/creazione standard; B per cancellazioni.

## Esempi
| Situazione | Esito |
|-----------|-------|
| Cliente noto chiede appuntamento, slot liberi | C — propone e prenota autonomamente |
| Sconosciuto chiede appuntamento al primo messaggio | B — propone ma Massimo conferma |
| "Annulla l'appuntamento di domani" | B — cancellazione, serve approvazione |
| Due calendari in conflitto sullo slot | B — non prenota, segnala a Massimo |
