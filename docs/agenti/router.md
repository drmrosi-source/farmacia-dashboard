# Scheda agente — Router

> Riepilogo operativo. Dettaglio completo in [§03](../03-orchestratore-e-router.md).

| Campo | Valore |
|-------|--------|
| **Ruolo** | Classificare ogni messaggio e sceglierne l'agente competente |
| **Modello** | Haiku 4.5 (leggero — P3) |
| **Fase** | 1 |
| **Autonomia** | Interna: non parla mai all'esterno |
| **Tool** | Nessuno (solo lettura del contesto passato dall'orchestratore) |

## Input
- Testo/tipo del messaggio corrente.
- Contesto sintetico: categoria contatto, ultimo intent, stato conversazione.

## Output (JSON obbligatorio, tool forzato)
```json
{
  "agente": "personale | prenotazioni | farmacia | nessuno",
  "intent": "networking | collaborazione | prenotazione | info_farmacia | spam | personale_urgente | altro",
  "categoria_contatto": "sconosciuto | networking | cliente_farmacia | fornitore | familiare_amico | vip",
  "urgenza": "bassa | media | alta",
  "lingua": "it | en | ...",
  "confidenza": 0.0,
  "motivazione": "stringa breve per audit"
}
```

## Regole
- Non risponde al contatto; solo instrada.
- `confidenza < 0.5` → fallback `personale` + flag per revisione.
- Sceglie l'intento **dominante**; annota gli altri in `motivazione`.
- `urgenza` non aumenta l'autonomia (alza solo la priorità di notifica, §06).

## Metriche di salute
- Accuratezza stimata via hand-off e correzioni (§10).
- Costo/latenza per invocazione (deve restare la frazione più bassa, P3).
