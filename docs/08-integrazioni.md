# 08 — Integrazioni esterne

Ogni integrazione è isolata dietro **tool** (P4): la logica dell'agente non conosce il
fornitore, conosce solo il contratto del tool. Cambiare fornitore = riscrivere il tool,
non l'agente.

## 8.1 WhatsApp (canale d'ingresso primario) — [FASE 1]

**Provider:** WhatsApp Business Cloud API (Meta).

### 8.1.0 Topologia dei numeri (vincolo fondamentale)

**Vincolo tecnico rigido:** un numero collegato alla Cloud API di Meta **non può più
essere usato nella normale app WhatsApp**. L'uso è esclusivo: o app, o API. Questo
determina *quale* numero mettiamo sulla piattaforma e ha impatto diretto sulla vita
quotidiana del titolare.

Ne deriva una topologia a **due numeri**:

| Numero | Dove vive | A cosa serve |
|--------|-----------|--------------|
| **Numero-piattaforma** | Cloud API (gestito dagli agenti) | Riceve i messaggi degli interlocutori esterni → viene filtrato dagli agenti. Non usabile nell'app del telefono. |
| **Numero personale di Massimo** | App WhatsApp normale sul suo telefono | Vita privata, intatta. Riceve dal bot le richieste di approvazione/notifica (canale di controllo, §8.3). |

Con questa impostazione **gli agenti non leggono mai il numero personale di Massimo**:
vedono solo il traffico del numero-piattaforma. Il personale riceve unicamente le
notifiche generate dal bot.

**[DA DECIDERE — scelta di onboarding, proprietario: Massimo]**

- **Opzione A (consigliata):** il numero-piattaforma è un **numero nuovo** (seconda SIM /
  numero virtuale). Il numero personale resta sul telefono e continua a funzionare
  normalmente. I contatti nuovi scrivono al numero-piattaforma; i vecchi si reindirizzano
  gradualmente. Rischio basso, reversibile.
- **Opzione B:** il **numero storico** di Massimo viene migrato sulla Cloud API (filtra
  esattamente i contatti già esistenti), ma da quel momento **non è più usabile nell'app**
  e Massimo adotta un nuovo numero per la vita privata. Cambio più impegnativo e meno
  reversibile.

> Nota: l'app *WhatsApp Business* (gratuita) gira sul telefono ma **non** offre webhook/
> automazione server-side: non abilita gli agenti. Per la piattaforma serve la Cloud API.

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

**Scelta: WhatsApp.** Il canale di controllo è WhatsApp stesso: il **bot (numero-
piattaforma)** scrive al **numero personale di Massimo** (§8.1.0), che resta un normale
utente dell'app. Massimo riceve così approvazioni e notifiche nella sua chat WhatsApp e
risponde normalmente. Coerente con la preferenza del titolare e con il principio "un solo
punto d'ingresso" (P1).

### Come si gestiscono i due vincoli WhatsApp
- **Pulsanti di approvazione**: si usano i messaggi **interattivi** di WhatsApp (*reply
  buttons* / *list messages*) per offrire ✅ Approva / ✏️ Modifica / ❌ Rifiuta. La
  modifica del testo si fa con un messaggio di risposta libero.
- **Finestra di servizio 24h**: fuori dalle 24h dall'ultimo messaggio di Massimo, Meta
  consente solo **template pre-approvati**. Si registrano quindi 1–2 **message template**
  di servizio (es. `approvazione_in_attesa` con variabili `{contatto}` e `{oggetto}`).
  Il template riapre la finestra; da lì la conversazione prosegue in testo libero con i
  pulsanti interattivi. In pratica, interagendo spesso, la finestra resta quasi sempre
  aperta.

> Requisito operativo [FASE 1]: creare e far approvare da Meta i template del canale di
> controllo prima del go-live. Elenco template in `appendici/struttura-repo.md`.

### Tool
```text
notify_massimo(livello, testo, azioni?, template?) -> ok
    # se la finestra 24h è chiusa, usa automaticamente il template di servizio
request_approval(decision_id, testo_proposto, azioni) -> pending
    # invia un messaggio interattivo con i pulsanti Approva/Modifica/Rifiuta
```
Le risposte di approvazione rientrano dallo stesso webhook WhatsApp come **eventi
autenticati** (solo dal numero di Massimo, §07 V5): l'orchestratore le riconosce come
messaggi di controllo — non come messaggi di un interlocutore esterno — e sbloccano o
annullano l'azione, registrando la correzione.

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
