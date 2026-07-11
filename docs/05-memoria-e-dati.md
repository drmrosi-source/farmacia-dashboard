# 05 — Memoria e modello dati

La memoria condivisa (P5) è l'unica fonte di verità. Ogni agente legge/scrive qui,
secondo permessi (§07). Implementazione: **Postgres (Supabase)** + estensione
**pgvector** per la memoria semantica.

## 5.1 Tipi di memoria

| Tipo | Cosa contiene | Dove |
|------|---------------|------|
| **Profili contatti** | Chi è l'interlocutore, categoria, note, preferenze | `contacts` |
| **Storico conversazioni** | Ogni messaggio in/out, con metadati | `conversations`, `messages` |
| **Decisioni** | Cosa ha deciso router/agente/policy per ogni messaggio | `decisions` |
| **Correzioni** | Approvazioni/modifiche/rifiuti di Massimo | `corrections` |
| **Memoria semantica** | Fatti durevoli e riassunti, ricercabili per similarità | `memory` (con embedding) |
| **Audit** | Traccia immutabile di azioni a effetto esterno | `audit_log` (§07/§10) |

## 5.2 Schema relazionale (DDL indicativo)

```sql
-- Contatti / profili
create table contacts (
  id            uuid primary key default gen_random_uuid(),
  wa_phone      text unique not null,           -- numero WhatsApp normalizzato E.164
  display_name  text,
  categoria     text not null default 'sconosciuto',  -- vedi enum router §03
  vip           boolean not null default false,
  note          text,
  preferenze    jsonb not null default '{}',     -- lingua, canale preferito, ecc.
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

-- Conversazioni (finestra logica con un contatto)
create table conversations (
  id            uuid primary key default gen_random_uuid(),
  contact_id    uuid not null references contacts(id),
  stato         text not null default 'aperta',  -- aperta | in_attesa_approvazione | chiusa
  ultimo_intent text,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

-- Messaggi (in ingresso e in uscita)
create table messages (
  id                  uuid primary key default gen_random_uuid(),
  conversation_id     uuid not null references conversations(id),
  contact_id          uuid not null references contacts(id),
  direzione           text not null,            -- inbound | outbound
  canale              text not null default 'whatsapp',
  provider_message_id text,                     -- per deduplica (unique parziale)
  tipo                text not null default 'text', -- text|image|audio|document|...
  contenuto           text,
  media_url           text,
  created_at          timestamptz not null default now()
);
create unique index on messages (provider_message_id)
  where provider_message_id is not null;         -- idempotenza inbound

-- Decisioni prese dal sistema per un messaggio inbound
create table decisions (
  id               uuid primary key default gen_random_uuid(),
  message_id       uuid not null references messages(id),
  contact_id       uuid not null references contacts(id),
  agente           text not null,
  intent           text,
  categoria_policy text not null,               -- A | B | C
  urgenza          text,
  confidenza_router numeric,
  confidenza_agente numeric,
  proposal         jsonb not null,              -- Proposal completo (§04)
  esito            text not null default 'in_attesa', -- inviato|non_inviato|in_attesa|eseguito
  created_at       timestamptz not null default now()
);

-- Correzioni umane (feedback per il trainer)
create table corrections (
  id            uuid primary key default gen_random_uuid(),
  decision_id   uuid not null references decisions(id),
  tipo          text not null,                  -- approva | modifica | rifiuta
  testo_originale text,
  testo_corretto  text,
  motivo        text,
  created_at    timestamptz not null default now()
);

-- Memoria semantica (fatti durevoli, riassunti)
create table memory (
  id            uuid primary key default gen_random_uuid(),
  contact_id    uuid references contacts(id),   -- null = memoria globale
  tipo          text not null,                  -- fatto | riassunto | preferenza | regola
  contenuto     text not null,
  embedding     vector(1536),                   -- dimensione secondo il modello di embedding
  fonte         text,                           -- da quale messaggio/decisione deriva
  scadenza      timestamptz,                    -- opzionale (dati con TTL)
  created_at    timestamptz not null default now()
);
create index on memory using ivfflat (embedding vector_cosine_ops);

-- Audit immutabile delle azioni a effetto esterno
create table audit_log (
  id            uuid primary key default gen_random_uuid(),
  actor         text not null,                  -- agente/tool/umano
  azione        text not null,                  -- create_booking, send_whatsapp, ...
  target        text,
  args_hash     text,                           -- hash degli argomenti (privacy)
  esito         text not null,                  -- ok | errore | rifiutato
  decision_id   uuid references decisions(id),
  created_at    timestamptz not null default now()
);
```

> Le dimensioni dell'`embedding` dipendono dal modello scelto. [DA DECIDERE] fornitore
> di embedding (es. un modello di embedding dedicato) — isolato dietro un tool
> `embed(text)` così da poterlo cambiare senza toccare lo schema logico.

## 5.3 Accesso alla memoria da parte degli agenti

Gli agenti **non** interrogano Postgres liberamente. Accedono tramite tool controllati:

| Tool | Uso | Permesso tipico |
|------|-----|-----------------|
| `get_contact_profile(contact_id)` | Legge profilo + preferenze | tutti |
| `get_recent_messages(conversation_id, n)` | Ultimi n messaggi | tutti |
| `search_memory(query, k)` | Ricerca semantica top-k | tutti |
| `write_memory(fatto)` | Salva un fatto durevole | agenti conversazionali |
| `update_contact(contact_id, patch)` | Aggiorna profilo (categoria, note) | personale, farmacia |

Questo mantiene la **separazione logica/strumenti** (P4) e permette di applicare
minimizzazione e audit in un solo punto.

## 5.4 Politiche di memoria

- **Riassunti progressivi**: conversazioni lunghe vengono riassunte periodicamente in
  `memory` (tipo `riassunto`) per non gonfiare il contesto (§03 `load_context`).
- **TTL**: dati sensibili o effimeri hanno `scadenza`; un job li purga (§07 retention).
- **Provenienza**: ogni `memory.fonte` traccia da dove viene un fatto, per poterlo
  invalidare se la fonte era errata (utile al trainer).
- **Separazione per contatto**: la ricerca semantica è filtrata per `contact_id` quando
  il fatto è personale; i fatti globali (es. orari farmacia) hanno `contact_id = null`.

## 5.5 Coerenza e concorrenza
- Le scritture su `decisions`/`messages` avvengono in transazione con la persistenza del
  messaggio inbound.
- L'idempotenza sui webhook (indice su `provider_message_id`) previene doppie
  elaborazioni in caso di retry di Meta.
- Lo stato conversazione (`conversations.stato`) coordina il flusso di approvazione:
  passa a `in_attesa_approvazione` in categoria B e torna `aperta` alla risoluzione.
