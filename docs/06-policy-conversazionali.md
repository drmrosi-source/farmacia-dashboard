# 06 — Policy conversazionali

Le policy definiscono **quanto** la piattaforma può agire da sola. Sono applicate dal
**policy engine**, un componente **deterministico** (P6): il modello propone una
categoria, ma la decisione finale segue regole codificate e verificabili.

## 6.1 Le tre categorie

| Categoria | Nome | Comportamento | Chi decide di inviare |
|-----------|------|---------------|------------------------|
| **A** | Mai intervenire | Nessuna risposta esce. Si registra (e, se utile, si notifica Massimo in silenzio). | Nessuno / Massimo manualmente |
| **B** | Solo suggerimenti | L'agente prepara una risposta/azione; parte **solo** dopo approvazione umana. | Massimo (approva/modifica/rifiuta) |
| **C** | Autonomo | L'agente risponde/agisce da solo entro i confini definiti; si registra e si notifica secondo soglia. | La piattaforma |

Principio guida (P7): **il default è B** finché non c'è motivo esplicito e sicuro per C,
o un motivo esplicito per A.

## 6.2 Criteri di classificazione

Il policy engine combina tre assi. La categoria risultante è la **più conservativa** tra
quelle imposte dai singoli criteri (una condizione A vince su B, che vince su C).

### Asse 1 — Contenuto / azione proposta

| Contenuto o azione | Categoria minima |
|--------------------|------------------|
| Info pubblica e verificabile (orari, indirizzo, servizi farmacia) | C |
| Proposta di slot su disponibilità reale del calendario | C |
| Risposta interlocutoria neutra ("grazie, verifico e ti aggiorno") | C |
| Impegno per conto di Massimo (accettare collaborazione, confermare presenza a evento) | B |
| Comunicazione di networking sostanziale (risposta a proposta professionale) | B |
| Azione **irreversibile** (cancellazione definitiva, invio a molti, impegno vincolante) | B (mai C) |
| Contenuto sensibile: salute personale, denaro, questioni legali/contrattuali | B |
| Spam, promozioni non richieste, catene, contenuti offensivi | A |
| Consulenza medica/farmacologica personalizzata | A (rifiuto + eventuale rinvio a contatto umano) |

### Asse 2 — Contatto

| Contatto | Effetto |
|----------|---------|
| **Sconosciuto** | Alza la prudenza: azioni a impatto esterno non vanno mai in C al primo contatto. |
| **Cliente farmacia** | Info farmacia in C; nulla di personale su Massimo. |
| **Networking noto** | Suggerimenti in B (l'agente personale prepara, Massimo approva). |
| **Familiare / amico / VIP** | Categoria A di default sul contenuto personale (la piattaforma non risponde al posto di Massimo a chi è intimo), salvo azioni operative neutre. |
| **Fornitore** | B per impegni; C per info logistiche neutre. |

### Asse 3 — Urgenza e confidenza

- `urgenza: alta` **non** aumenta l'autonomia: aumenta la **priorità della notifica** a
  Massimo. Un'urgenza non è un permesso ad agire da soli.
- `confidenza` bassa (router o agente < 0.6) → forza almeno **B**.

## 6.3 Eccezioni

1. **Prima interazione con uno sconosciuto**: qualunque azione a impatto esterno →
   almeno B, indipendentemente dal contenuto.
2. **Orari notturni / fuori servizio** [DA DECIDERE finestra]: le azioni in C possono
   essere differite e notificate, non eseguite immediatamente, se riguardano invii
   verso terzi.
3. **Contatto in lista "sempre-umano"**: alcuni contatti (es. familiari stretti,
   avvocato, commercialista) sono marcati e ricadono **sempre** in A/B, mai C.
4. **Parole/temi sentinella**: presenza di temi sensibili (salute, soldi, legale,
   emergenza) → forza B/A anche se l'agente aveva proposto C.
5. **Override esplicito di Massimo**: Massimo può, per un contatto o una conversazione,
   concedere temporaneamente C ("gestisci tu con Tizio le prossime email"), registrato e
   revocabile.

Le eccezioni sono **regole**, non suggerimenti al modello: vivono nel policy engine.

## 6.4 Il flusso decisionale (deterministico)

```text
policy_engine.evaluate(proposal, contact, routing):
    cat = proposal.categoria_suggerita          # punto di partenza dal modello
    cat = max_conservativa(cat, per_contenuto(proposal.actions, proposal.reply_text))
    cat = max_conservativa(cat, per_contatto(contact, proposal.actions))
    if router.confidenza < 0.6 or proposal.confidenza < 0.6:
        cat = max_conservativa(cat, "B")
    for regola in eccezioni_attive(contact, proposal):
        cat = applica(regola, cat)               # può solo alzare la prudenza
    return { category: cat, notify_level: livello_notifica(routing.urgenza, cat) }
```

Regola d'oro: **il policy engine può solo rendere una decisione più prudente**, mai meno.
Un agente non può "sbloccarsi" da solo verso C.

## 6.5 Notifiche a Massimo

| Situazione | Notifica |
|-----------|----------|
| Categoria B (serve approvazione) | Sempre, con testo pronto: ✅ / ✏️ / ❌ |
| Categoria C con urgenza alta | Notifica informativa post-azione |
| Categoria C ordinaria | Riepilogo aggregato (digest), non per singolo messaggio |
| Categoria A rilevante (es. sconosciuto insistente) | Notifica silenziosa/log; digest |
| Spam A | Solo log, nessuna notifica |

Il livello di notifica evita di spostare il rumore da WhatsApp al canale di controllo:
l'obiettivo (§01) è **ridurre** il carico, non moltiplicarlo.

## 6.6 Registrazione ai fini dell'apprendimento
Ogni decisione e ogni esito di approvazione B sono scritti in `decisions`/`corrections`
(§05). Sono la materia prima dell'agente trainer (§ scheda trainer): il sistema deve
poter spiegare **perché** ha scelto A/B/C e imparare quando Massimo lo corregge.
