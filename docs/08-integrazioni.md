# 08 — Integrazioni esterne

Ogni integrazione è isolata dietro **tool** (P4): la logica dell'agente non conosce il
fornitore, conosce solo il contratto del tool. Cambiare fornitore = riscrivere il tool,
non l'agente.

## 8.1 WhatsApp (canale d'ingresso primario) — [FASE 1]

**Provider:** WhatsApp Business Cloud API (Meta).

### Ingresso
- Endpoint webhook unico (P1) esposto dall'orchestratore.
- Setup: verifica `verify_token` (challenge GET di Meta), poi ricezione POST.
- Sicurezza: firma `X-Hub-Signature-256` (§07).
- Normalizzazione del payload Meta → `InboundMessage` (vedi `appendici/schema-messaggi.md`).
- Deduplica su `provider_message_id`.

### Uscita — tool `send_whatsapp`
```text
send_whatsapp(to, text | template, context_message_id?) -> { provider_message_id }
```
- Rispetta la **finestra di servizio 24h** di WhatsApp: fuori finestra si usano
  **message template** approvati; dentro finestra, messaggi in testo libero.
- Idempotenza: chiave logica per evitare doppi invii su retry.
- Tipi supportati in v2.0: testo; template; (media/immagini opzionali). Audio/immagini in
  ingresso: gestiti almeno con acknowledgment; trascrizione audio [DA DECIDERE / fase >1].

## 8.2 Calendly + Calendario — [FASE 2]

Usati dall'**agente prenotazioni** (scheda dedicata).

### Tool
```text
get_availability(range, durata) -> [slot]         # legge disponibilità reale
create_booking(slot, invitato, note) -> booking   # crea l'appuntamento
cancel_booking(booking_id, motivo) -> ok          # cancella (categoria B, §06)
reschedule_booking(booking_id, nuovo_slot) -> ok  # riprogramma
```

### Fonti
- **Calendly API** per la prenotazione self-service e i tipi di evento.
- **Google Calendar** (connettore disponibile) come fonte di verità della disponibilità
  reale e per creare/leggere eventi, evitando doppie prenotazioni.
- Coerenza: prima di proporre uno slot, l'agente verifica **entrambe** le fonti se
  attive. La proposta di slot è categoria C; la cancellazione è categoria B (§06).

> [DA DECIDERE] se Calendly è la fonte primaria (link self-service) o se la piattaforma
> gestisce direttamente gli slot su Google Calendar e usa Calendly solo come vetrina.

## 8.3 Canale di controllo (verso Massimo) — [FASE 1]

Il modo in cui la piattaforma chiede approvazioni (categoria B) e invia notifiche.

Opzioni:
- **WhatsApp dedicato** (un numero/chat riservata Massimo↔piattaforma): coerente con il
  canale principale, ma soggetto ai vincoli 24h.
- **Telegram Bot**: nessun vincolo di finestra, pulsanti inline (✅/✏️/❌) nativi, ideale
  per approvazioni rapide.

**Raccomandazione:** Telegram Bot per il canale di controllo (pulsanti + no finestra
24h), mantenendo WhatsApp per gli interlocutori esterni. [DA DECIDERE — conferma di
Massimo.]

### Tool
```text
notify_massimo(livello, testo, azioni?) -> ok
request_approval(decision_id, testo_proposto, azioni) -> pending
```
Le risposte di approvazione rientrano come **eventi autenticati** (solo dal chat_id di
Massimo, §07 V5) e sbloccano/annullano l'azione, registrando la correzione.

## 8.4 Google (Calendar, Gmail, Drive) — [FASE 2–3]

Connettori già disponibili nell'ambiente. Usi previsti:
- **Google Calendar**: disponibilità e gestione eventi (§8.2).
- **Gmail**: [FASE 3] estensione multicanale — trattare le email come ulteriore ingresso
  dallo stesso orchestratore (P1). Non in v2.0 core, ma l'architettura lo consente
  aggiungendo un adattatore di ingresso.
- **Google Drive**: archiviazione di documenti/allegati e knowledge base della farmacia
  (fonte per l'agente farmacia).

## 8.5 Provider LLM — [FASE 1]

**Anthropic Claude API.** Assegnazione modelli (§02.5):
- Router → Haiku 4.5 (leggero, P3).
- Agenti conversazionali → Sonnet 5.
- Trainer → Opus 4.8 (ragionamento profondo, offline).
- Embedding per la memoria semantica → tool `embed()` isolato [DA DECIDERE modello].

Vincoli: structured output/tool forzato per router e `Proposal`; nessun segreto nei
prompt; minimizzazione del contesto (§07).

## 8.6 Contratto generale di un tool

Ogni tool MUST:
1. avere uno **schema** di input/output validato;
2. dichiarare `impatto_esterno` e `reversibile`;
3. essere **idempotente** o protetto da chiave di idempotenza se a effetto esterno;
4. scrivere su `audit_log` quando ha effetto esterno;
5. non contenere logica di dominio dell'agente (solo l'operazione tecnica).

Elenco tool per fase in `appendici/struttura-repo.md`.
