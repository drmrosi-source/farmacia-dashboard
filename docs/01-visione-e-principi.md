# 01 — Visione e principi

## 1.1 Visione

Costruire una piattaforma di **agenti AI specializzati**, coordinati da un
**orchestratore centrale**, capace di:

- gestire la messaggistica in arrivo (a partire da WhatsApp);
- filtrare le richieste separando rumore, opportunità e urgenze;
- prenotare appuntamenti in autonomia quando le regole lo consentono;
- coinvolgere Massimo **solo quando serve davvero**, con un suggerimento pronto
  all'uso invece che con una notifica grezza.

L'obiettivo non è "un chatbot", ma un **assistente operativo** che riduce il carico
cognitivo e il tempo speso in conversazioni a basso valore, mantenendo Massimo sempre
in controllo delle decisioni che contano.

## 1.2 Utenti e ruoli

| Ruolo | Descrizione |
|-------|-------------|
| **Titolare (Massimo)** | Destinatario finale, unico approvatore per la categoria B, proprietario dei dati. |
| **Interlocutori esterni** | Chiunque scriva su WhatsApp: pazienti/clienti farmacia, contatti di networking, fornitori, sconosciuti. |
| **Amministratore tecnico** | Chi gestisce deploy, segreti e configurazione (inizialmente coincide con lo sviluppatore). |

## 1.3 Principi architetturali

Sono i vincoli non negoziabili che orientano ogni decisione tecnica.

### P1 — Un solo punto d'ingresso
Ogni messaggio, da qualunque canale, entra da **un unico webhook/orchestratore**. Non
esistono agenti raggiungibili direttamente dall'esterno. Questo dà un solo punto dove
applicare autenticazione, logging, rate limiting e policy.

### P2 — Agenti specializzati
Ogni agente ha **una responsabilità stretta** e un prompt dedicato. Nessun agente
"tuttofare". La specializzazione migliora qualità, testabilità e sicurezza (ogni
agente vede solo gli strumenti che gli servono — vedi §07).

### P3 — Router leggero
La classificazione iniziale usa il **modello più economico e veloce** possibile
(Haiku). Il router non risolve il problema, lo instrada. Deve essere quasi
istantaneo e a basso costo perché viene eseguito su **ogni** messaggio.

### P4 — Separazione tra logica e strumenti
La logica di ragionamento (prompt + policy) è separata dagli **strumenti** (tool: invio
WhatsApp, query calendario, scrittura DB). Gli strumenti sono funzioni pure,
versionate e testabili; gli agenti li invocano tramite un'interfaccia comune. Cambiare
fornitore (es. Calendly → altro) non tocca la logica dell'agente.

### P5 — Memoria condivisa
Esiste un **layer di memoria unico** (§05) accessibile da tutti gli agenti secondo
permessi: profili dei contatti, storico conversazioni, decisioni prese, correzioni di
Massimo. Nessun agente conserva stato proprio nascosto.

### P6 — Regole di sicurezza esplicite
Cosa un agente può fare in autonomia, cosa richiede approvazione e cosa è vietato è
**codificato e verificabile**, non lasciato al giudizio del modello (§06, §07). Il
modello propone; le regole decidono cosa è eseguibile senza umano.

### P7 — Human-in-the-loop by design
Il default per azioni a impatto esterno è **proporre, non agire**. L'autonomia piena si
concede per categoria e per azione, in modo incrementale e revocabile.

### P8 — Apprendimento tracciabile
Ogni correzione di Massimo è un dato di prima classe: viene registrata e usata
dall'agente trainer (§ scheda trainer) per proporre miglioramenti ai prompt e alle
regole, mai per modificarli in automatico senza revisione.

## 1.4 Non-obiettivi (fuori scope v2.0)

- Non è un CRM completo né un gestionale di farmacia.
- Non gestisce pagamenti o transazioni sanitarie.
- Non fornisce consulenza medica/farmacologica personalizzata (vedi vincoli §07).
- Non prende decisioni irreversibili in autonomia (cancellazioni definitive, impegni
  contrattuali, comunicazioni legali).

## 1.5 Glossario

| Termine | Significato |
|---------|-------------|
| **Orchestratore** | Componente che riceve ogni messaggio, coordina router e agenti, applica le policy e restituisce/instrada la risposta. |
| **Router** | Agente leggero che classifica il messaggio e sceglie l'agente competente. |
| **Agente** | Unità specializzata (prompt + strumenti + policy) che gestisce un dominio. |
| **Strumento (tool)** | Funzione richiamabile da un agente (es. `send_whatsapp`, `create_booking`). |
| **Memoria condivisa** | Store persistente comune (profili, storico, decisioni, correzioni). |
| **Policy A/B/C** | Livelli di autonomia conversazionale (§06). |
| **Correzione** | Modifica/rifiuto da parte di Massimo di un output proposto; alimenta il trainer. |
| **Contatto** | Interlocutore esterno identificato (numero WhatsApp + profilo). |
| **Conversazione** | Sequenza di messaggi con un contatto entro una finestra temporale. |
