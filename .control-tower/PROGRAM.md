# Programma di progetto — Farmacia Dashboard

Stato: ATTIVO
Priorità portfolio: MEDIA
Obiettivo: rendere la dashboard circolari uno strumento quotidiano affidabile per leggere rapidamente, capire cosa fare e recuperare il testo completo/allegato senza ambiguità.

## Regola di esecuzione

Il progetto è una SPA statica in `index.html`; evitare complessità non necessaria. Ogni modifica deve preservare compatibilità con i webhook Railway esistenti e deve essere verificabile con controlli deterministici su HTML/JS e, quando disponibile, smoke test su preview.

## Milestone 1 — Dettaglio circolare completo

1. Formalizzare il contratto dati della circolare: titolo, categoria/classificazione, data/scadenza, riassunto AI, punti chiave, azioni richieste, testo completo, allegato.
2. Verificare e integrare `testo_completo` nella vista dettaglio quando disponibile.
3. Gestire in modo chiaro campi mancanti e fallback, senza inventare contenuti.
4. Migliorare leggibilità del dettaglio e gerarchia delle informazioni operative.

Gate di uscita: una circolare completa e una incompleta sono entrambe leggibili senza errori JS e senza confondere riassunto e testo originale.

## Milestone 2 — Affidabilità dati/API

1. Documentare gli endpoint Railway usati e il payload atteso.
2. Aggiungere fixture o esempi JSON versionati per testare il rendering senza dipendere dal backend live.
3. Gestire timeout/errori/retry e stati vuoti in UI.
4. Verificare download allegato e messaggi di errore.

## Milestone 3 — Ricerca e azione

1. Rafforzare filtri, ricerca e ordinamento per urgenza/scadenza.
2. Evidenziare chiaramente "cosa devo fare" e "entro quando".
3. Se coerente con i dati disponibili, aggiungere stato letto/da fare solo con una fonte persistente definita; niente stato finto solo client-side se può creare ambiguità.

## Milestone 4 — Qualità UI e accessibilità

1. Tastiera, focus, modali, escape, semantic HTML e contrasto.
2. Responsive desktop/tablet.
3. Ridurre regressioni visive con checklist di smoke test.

## Milestone 5 — Consegna stabile

1. Preview verificata.
2. Contratto backend documentato.
3. Deploy/promozione solo con gate esplicito quando cambia il servizio live.

## Coda immediata

- P1: chiudere il supporto `testo_completo` nella vista dettaglio.
- P1: fixture e contratto payload circolare.
- P2: error handling e allegati.
- P2: accessibilità e smoke QA.

## Definition of Done del progetto

La dashboard è completa quando lo staff può trovare una circolare, capirne scadenze e azioni, leggere sia sintesi sia testo completo e scaricare l'allegato, con errori e dati mancanti gestiti esplicitamente.