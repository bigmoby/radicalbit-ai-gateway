# Radicalbit AI Gateway Core — proposta architetturale e valutazione di fattibilità

**Stato:** proposta / analisi statica preliminare, non ancora validata end-to-end; riscontri del §4 verificati sul codice (`main` @ `9b901ec`, 22/10/2026), incluso un tentativo reale di `alembic upgrade head` su SQLite  
**Destinatari:** team Radicalbit e manutentori del progetto open source  
**Repository upstream:** https://github.com/radicalbit/radicalbit-ai-gateway  
**Caso d'uso iniziale:** Home Assistant Yellow con Raspberry Pi Compute Module 4, 4 GB RAM e memoria eMMC; la proposta è volutamente generalizzabile ad ambienti edge, ARM64 e sviluppo locale.

## 1. Sintesi esecutiva

L'obiettivo è introdurre nell'upstream una modalità di deployment **Core** di Radicalbit AI Gateway: un gateway LLM autentico, non un proxy riscritto e non un'integrazione dedicata a Home Assistant Assist, capace di funzionare senza lo stack di osservabilità pesante.

La modalità Core **deve mantenere**: API OpenAI-compatible, routing, guardrails, gestione di progetti/route/credenziali/gruppi, **rate limiting**, **token limiting**, **budget limiting** (route, progetto, credenziale), **caching esatto e semantico**, **alerting** e UI di configurazione. Sono invece candidati all'esclusione: ClickHouse, OpenTelemetry Collector, metrics-worker, pipeline di analytics storici e tracing distribuito.

Architettura obiettivo:

```text
Client LLM / applicazioni / Home Assistant
                  |
                  v
       Radicalbit AI Gateway Core
       - API e routing LLM
       - guardrails e credenziali
       - rate/token/budget limiting
       - caching esatto e semantico
       - alerting operativo
       - UI di configurazione
                  |
          +-------+--------+
          |                |
          v                v
       SQLite         Valkey Bundle
   configurazioni    cache, contatori,
   e regole          finestre alert,
                     indici vettoriali
                  |
                  v
           Provider LLM esterni
```

SQLite risiederebbe su volume persistente (es. `/data/radicalbit/gateway.db` nell'App Home Assistant); la collocazione effettiva e le impostazioni di journaling vanno definite e testate. Valkey Bundle è il candidato per unificare storage dei limiter, cache e alert; **la compatibilità ARM64 e la sufficienza di un'unica istanza non sono ancora verificate**.

## 2. Principi della proposta upstream

1. **Full invariato di default**: nessuna regressione per i deployment esistenti.
2. **Core opt-in**: profilo esplicito, per esempio `GATEWAY_PROFILE=core` (nome e interfaccia di configurazione da concordare; non è una variabile upstream esistente).
3. **Un solo codice di business**: non duplicare routing, guardrails, limiter, budget, caching o alerting.
4. **Separazione per responsabilità**: distinguere eventi operativi/alert dalla persistenza degli eventi per analytics.
5. **Nessun codice Home Assistant-specifico nell'upstream**: packaging e notifiche HA rimangono integrazioni separate.
6. **Test come condizione di accettazione**: il fatto che una dipendenza sia opzionale nel codice non dimostra da solo la correttezza runtime.

## 3. Requisiti funzionali non negoziabili

| Capacità | Core | Nota |
|---|---|---|
| API OpenAI-compatible e streaming | Obbligatoria | Test con richieste reali |
| Routing e fallback LLM | Obbligatori | Verificare i percorsi di errore |
| Guardrails | Obbligatori | Inclusi eventi che attivano alert |
| Gestione progetti, route, credenziali e gruppi | Obbligatoria | Persistenza SQLite |
| Rate limiting | Obbligatorio | Valkey; concorrenza e reset finestre |
| Token limiting | Obbligatorio | Input/output e streaming |
| Budget limiting | Obbligatorio | Route, progetto e credenziale; verificare contabilizzazione dei costi |
| Routing budget-aware | Obbligatorio se disponibile in Full | Nessuna regressione |
| Cache esatta | Obbligatoria | Hit/miss, TTL, isolamento |
| Cache semantica | Obbligatoria | Ricerca vettoriale; moduli Valkey Bundle |
| Alert immediati e a finestra | Obbligatori | Non dipendenti dalla pipeline analytics |
| Notifiche email esistenti | Da mantenere | Eventuali webhook HA in PR distinta |
| Storico analytics ClickHouse | Escluso | Endpoint/UI devono gestire esplicitamente l'assenza |
| Tracing OpenTelemetry | Escluso | Nessun Collector necessario |

## 4. Riscontri verificati nel codice upstream

### 4.1 SQLite è già previsto, ma non completamente portabile

- [`db/database.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/db/database.py) costruisce una URL `sqlite:///...` quando `db_type == 'sqlite'`.
- [`utils/app_config.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/utils/app_config.py) dichiara `db_type: str = 'sqlite'` come default della classe di configurazione; il Compose ufficiale imposta invece PostgreSQL.
- [`alembic/env.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/alembic/env.py) distingue PostgreSQL nel percorso di migrazione online, ma non rende automaticamente portabili tutte le revisioni.
- **Prova reale:** `alembic upgrade head` su un file SQLite fallisce già alla revisione [`1a2b1d01d33e_add_groups.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/alembic/versions/1a2b1d01d33e_add_groups.py) (riga 24, `op.drop_constraint`) con `NotImplementedError: No support for ALTER of constraints in SQLite dialect`. Nessuna revisione usa `batch_alter_table` e `env.py` non imposta `render_as_batch`.
- [`d1e2f3a4b5c6_add_project_uuid_to_group_routes.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/alembic/versions/d1e2f3a4b5c6_add_project_uuid_to_group_routes.py) usa `create_foreign_key`, `drop_constraint`, `create_primary_key` e `alter_column`: occorre adattamento per SQLite, tipicamente tramite modalità batch Alembic.
- [`e0f1a2b3c4d5_alert_rule_scopes.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/alembic/versions/e0f1a2b3c4d5_alert_rule_scopes.py) contiene SQL PostgreSQL-specifico (`p."UUID"::text` e `UPDATE ... FROM`) da riscrivere o differenziare per dialetto.
- Altre revisioni legate a PostgreSQL: `b3c4d5e6f7a8_add_project_config_table.py` e `ca3e44546368_rename_user_key_to_key.py` usano `postgresql.ENUM` / `postgresql.TIMESTAMP`; `c4d5e6f7a8b9_partial_unique_project_name.py` e `b3c4d5e6f7a8` definiscono indici unici parziali con `postgresql_where`. **Su SQLite `postgresql_where` viene ignorato senza errori**: l'indice diventa unico su tutta la tabella (es. un progetto cancellato logicamente bloccherebbe il riuso del nome). Serve l'equivalente `sqlite_where`.
- **ORM non inizializzato su SQLite:** in [`server.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/server.py) `database.init_mappings()` è chiamato solo se `db_type == 'postgresql'`. Le tabelle (es. `Group`, `KeyLimit`, `ProjectBudgetLimit`) ereditano da `Reflected` (`DeferredReflection`), che non viene mappata senza `prepare()`: con SQLite l'ORM quasi certamente non funziona anche a migrazioni corrette.
- **Pool non adatto a SQLite:** `Database.connect()` usa `create_engine(..., pool_size=20, max_overflow=40)`, pensato per PostgreSQL; per SQLite su eMMC vanno definiti pool, timeout di lock e WAL.
- **Nessuna copertura di test:** la suite in `gateway/tests` non contiene riferimenti a SQLite.

**Conclusione:** esiste solo la costruzione della URL SQLite; **migrazioni (verificato: falliscono), mappature ORM e configurazione del pool non sono supportate**. Preferire migrazioni portabili alla sola creazione dello schema da modelli, che non risolverebbe gli aggiornamenti futuri.

### 4.2 ClickHouse è cablato nel codice, ma probabilmente non blocca l'avvio

In [`server.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/server.py) vengono costruiti `ClickHouseDatabase`, `EventDAO`, `RequestEventDAO`, `OtelTracesDAO` e `AlertDAO`; il ciclo di vita esegue `ch_database.connect()`. Tuttavia [`ClickHouseDatabase.connect()`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/db/clickhouse_database.py) si limita a `create_engine()`, che non apre connessioni: l'avvio senza ClickHouse con ogni probabilità **riesce**, e i guasti emergerebbero solo alla prima query analytics (non ancora verificato con un avvio reale).

Il refactoring resta necessario, ma per ragioni diverse dallo sblocco dell'avvio: inizializzazione condizionale, dipendenze opzionali e risposta esplicita delle API di analytics non disponibili, invece di errori a runtime.

Il Collector OTLP è già condizionato dalla configurazione `collector_base_url`; ciò non implica però che tutte le altre dipendenze da tracing/telemetria siano assenti.

### 4.3 Rate limiting e budget possono usare Valkey

I moduli [`limiting/rate_limiter.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/limiting/rate_limiter.py), [`limiting/budget_limiting.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/limiting/budget_limiting.py) e [`limiting/project_budget_limiter.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/limiting/project_budget_limiter.py) usano `RedisStorage` quando è configurato Redis/Valkey, con fallback in memoria. I contatori operativi non richiedono ClickHouse.

**Attenzioni:** il controllo e il consumo del limite possono avvenire in passaggi distinti; testare concorrenza e soglie. Il budget è contabilizzato usando costi di token/durata: verificare modelli, streaming, cache hit e richieste interrotte. Non confondere enforcement operativo con analytics storici.

### 4.4 Cache esatta e semantica

[`utils/gateway_route_factory.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/utils/gateway_route_factory.py) crea il client Redis quando è configurata la cache. La cache semantica in [`caching/semantic_caching.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/caching/semantic_caching.py) usa `VectorField`, indici `FT.CREATE`, ricerche `FT.SEARCH`, TTL e filtri per progetto, route e credenziale.

**Conclusione:** per la cache semantica serve supporto alla ricerca vettoriale; una normale istanza Valkey priva dei moduli appropriati non è sufficiente. Verificare Valkey Bundle ARM64, versioni/moduli, inizializzazione degli indici e comportamento dopo riavvio.

### 4.5 Gli alert sono già separabili dalla persistenza analytics

[`events/events_processor.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/events/events_processor.py) invia gli eventi all'`AlertDispatcher` in un executor separato **prima** di aggiungerli al buffer Celery. [`alerting/alert_dispatch.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/alerting/alert_dispatch.py) gestisce eventi immediati e finestre temporali; [`alerting/alert_store.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/alerting/alert_store.py) supporta Redis/Valkey e un fallback in-process per una singola replica.

L'unico punto dell'alerting che legge da ClickHouse è lo storico per regola: `AlertRuleService` → `alert_dao.get_history(...)` ([`services/alert_rule_service.py`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/services/alert_rule_service.py)). Gli alert in tempo reale non ne dipendono; in Core lo storico va dichiarato non disponibile.

Senza broker, il [`CeleryBuffer`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/radicalbit_ai_gateway/events/buffer.py) cattura l'eccezione di `send_task` e registra soltanto l'errore: non rompe il flusso, ma riempirebbe i log a ogni flush. In Core il sink va **disattivato**, non lasciato fallire.

**Conclusione:** conservare dispatcher, regole, scheduler e notifiche, disabilitando soltanto la pubblicazione degli eventi verso la pipeline analytics e gestendo l'assenza dello storico alert.

### 4.6 Dipendenze Python

`psycopg2` (versione sorgente, non `-binary`) è una dipendenza runtime obbligatoria ([`pyproject.toml`](https://github.com/radicalbit/radicalbit-ai-gateway/blob/main/gateway/pyproject.toml)) e va compilato con libpq; in una prova locale la build è fallita con Python 3.14. Anche in Core servirebbe quindi libpq sull'immagine ARM64, a meno di rendere il driver PostgreSQL una dipendenza opzionale.

## 5. Componenti: Full vs Core

| Componente | Full | Core proposto |
|---|---|---|
| Gateway Python + UI | Sì | Sì |
| PostgreSQL | Sì | No, sostituito da SQLite |
| SQLite | Non nel Compose ufficiale | Sì, persistente |
| Valkey | Sì | Sì |
| Valkey Bundle per cache semantica | Servizio separato nel Compose | Candidato a istanza unica, da verificare |
| ClickHouse | Sì | No |
| ClickHouse migrations | Sì | No |
| Metrics worker / Celery consumer | Sì | No |
| OpenTelemetry Collector | Sì | No |
| Adminer, Redis Insight, CH-UI | Strumenti accessori | No |

**Nota:** `valkey` e `valkey-cache` sono servizi distinti nel Compose upstream. Consolidarli in uno solo è un'ipotesi da collaudare, non una compatibilità già provata.

## 6. Matrice di fattibilità

| Area | Valutazione statica | Blocco / test ancora necessario |
|---|---|---|
| SQLite — connessione | Solo URL; pool pensato per PostgreSQL | Pool, lock timeout, WAL; test con DB persistente |
| SQLite — mappature ORM | Non supportate (`init_mappings` solo per PostgreSQL) | Chiamare `prepare()` anche per SQLite; test CRUD |
| SQLite — migrazioni | **Falliscono** (verificato, revisione `1a2b1d01d33e`) | Batch mode, `sqlite_where`, ENUM; `upgrade head` e upgrade incrementali in CI |
| Avvio senza ClickHouse | Probabilmente riesce (engine lazy); errori a runtime | Avvio reale; risposta esplicita delle API analytics |
| Avvio senza Celery worker | Plausibile; il buffer registra errori a ogni flush | Disattivare il sink senza perdere alert |
| Dipendenze Python | `psycopg2` da sorgente obbligatorio | Rendere opzionale il driver PostgreSQL o fornire libpq |
| Rate limiting | Implementazione Redis esistente | Concorrenza, soglie, finestre |
| Token limiting | Implementazione limiter esistente | Streaming e contabilizzazione |
| Budget limiting | Implementazione Redis esistente | Costi, scope, persistenza, reset |
| Routing budget-aware | Implementazione esistente | Test end-to-end |
| Cache esatta | Implementazione esistente | TTL, isolamento, hit/miss |
| Cache semantica | Implementazione vettoriale esistente | Moduli, ARM64, embedding, indice |
| Alerting operativo | Separazione già presente | Email, scheduler, soglie e assenza ClickHouse |
| UI Core | Da progettare | Nascondere/gestire sezioni analytics |
| ARM64 / Home Assistant Yellow | Non verificato | Build, avvio, RAM, CPU, eMMC |

## 7. Piano di lavoro per PR upstream

### PR 1 — Decoupling dell'osservabilità

- Introdurre una configurazione esplicita di profilo o capability (nome da discutere).
- Lasciare Full invariato di default.
- Rendere opzionali connessione ClickHouse, DAO e servizi dipendenti.
- Separare emissione eventi operativi/alert dal sink Celery/analytics, disattivando il `CeleryBuffer` in Core.
- Conservare scheduler e dispatcher degli alert; gestire l'assenza dello storico alert (`get_history`).
- Definire risposta degli endpoint analytics in Core (non disponibili esplicitamente, senza errori di startup).
- Test di regressione Full e test Core senza servizi analytics.

### PR 2 — SQLite realmente supportato

- Inizializzare le mappature ORM (`Reflected.prepare`) anche per SQLite e adattare la configurazione del pool.
- Portare le revisioni Alembic su operazioni compatibili con SQLite (batch dove richiesto, a partire da `1a2b1d01d33e`).
- Riscrivere SQL specifico PostgreSQL o differenziarlo per dialetto; aggiungere `sqlite_where` accanto a `postgresql_where` negli indici parziali; gestire `postgresql.ENUM`.
- Aggiungere in CI un test `alembic upgrade head` su SQLite.
- Valutare se rendere `psycopg2` una dipendenza opzionale.
- Testare schema nuovo, aggiornamenti incrementali, chiavi esterne, integrità e transazioni.
- Testare CRUD di progetti, configurazioni, credenziali, gruppi, budget e regole di alert.
- Valutare WAL, timeout di lock, connessioni, backup e ripristino.

### PR 3 — Deployment Core e test integrati

- Fornire Compose Core documentato, senza stack analytics.
- Testare cache esatta e semantica con Valkey Bundle.
- Testare rate/token/budget limiting, routing budget-aware, alerting e streaming.
- Eseguire smoke test e test di carico su ARM64.
- Documentare capacità presenti/assenti e limiti operativi.

**Strategia di contribuzione:** aprire prima una RFC/GitHub Discussion per concordare perimetro e design, quindi PR piccole e revisionabili. Il packaging dell'App Home Assistant resta in repository separato.

## 8. Scenari di collaudo end-to-end obbligatori

1. **Boot minimale:** solo gateway, SQLite e Valkey Bundle; nessuna connessione a ClickHouse/Collector/worker.
2. **Persistenza:** creare progetto, route, credenziale, gruppo e regola; riavviare; verificare che siano presenti.
3. **LLM:** richiesta chat standard e streaming, routing e fallback.
4. **Rate limit:** superare una soglia, verificare blocco, alert e ripristino alla nuova finestra.
5. **Budget:** costi input/output/durata, soglie route/progetto/credenziale, routing budget-aware, finestre e concorrenza.
6. **Cache esatta:** primo miss, hit successivo, TTL, isolamento fra credenziali e progetti.
7. **Cache semantica:** indice vettoriale creato, ricerca simile, soglia di similarità, TTL e riavvio.
8. **Alert:** guardrail, fallback, limiti, soglie e aggregazioni temporali; email e assenza di duplicazioni.
9. **UI/API:** configurazione e CRUD completi; endpoint analytics disabilitati in modo chiaro.
10. **Affidabilità:** indisponibilità Valkey, riavvio container, carico concorrente, consumo RAM e scritture eMMC.
11. **Regressione Full:** suite esistente invariata, inclusi analytics e tracing.

## 9. Questioni ancora da decidere con il team

- **Profilo unico o feature flags indipendenti?** Un profilo Core semplifica il deployment, ma flag per sink eventi, tracing e analytics possono rendere il sistema più componibile. Evitare combinazioni non supportate.
- **Portabilità SQLite nel lungo periodo:** mantenere due dialetti comporta manutenzione continua delle migrazioni.
- **Budget e cache:** verificare precisamente se e come una risposta servita dalla cache incida su rate limit, token e budget; documentare la semantica desiderata.
- **Persistenza Valkey:** decidere RDB/AOF, TTL, eviction policy e cosa succede ai budget dopo un riavvio o una perdita dei contatori. Il budget deve avere comportamento sicuro e prevedibile.
- **Alert:** quali eventi richiedono esclusivamente stato operativo e quali consultano ClickHouse? Come garantire affidabilità delle notifiche in caso di riavvio?
- **Cache semantica:** costo degli embedding, compatibilità del motore vettoriale e memoria disponibile su 4 GB.
- **Dipendenze Python/ARM64:** build di pacchetti nativi e disponibilità delle immagini Valkey Bundle per l'architettura target.
- **Sicurezza e manutenzione:** credenziali, backup SQLite, aggiornamenti schema, accesso alla UI e compatibilità con i cicli di release upstream.
- **Integrazione Home Assistant:** notifiche tramite webhook/eventi e UI HA da valutare dopo il supporto Core upstream, non nella prima PR.

## 10. Considerazioni finali

**La proposta è architetturalmente promettente, ma non ancora verificata end-to-end.** L'ispezione del repository ha mostrato che rate limiting, budget, caching e alerting hanno già percorsi operativi basati su Redis/Valkey, separabili dalla memorizzazione analytics in ClickHouse. È un buon presupposto per una modalità Core upstream senza duplicare la logica di business.

I due principali interventi sono: **(1) disaccoppiare l'inizializzazione e i servizi ClickHouse/Celery dal percorso LLM e dagli alert; (2) rendere davvero supportato SQLite, che oggi è il lavoro più pesante: migrazioni Alembic (verificato che falliscono), mappature ORM, pool di connessioni e dipendenza da `psycopg2`**. Il disaccoppiamento da ClickHouse appare invece meno invasivo del previsto, perché l'engine è lazy e l'avvio probabilmente non dipende dalla sua raggiungibilità. La cache semantica introduce un ulteriore requisito: supporto vettoriale reale e validato su ARM64.

**Decisione raccomandata:** proporre al team Radicalbit una RFC per una modalità Core opt-in, con Full invariato, e procedere per PR incrementali guidate da test. Non dichiarare la soluzione pronta per Home Assistant Yellow prima di un avvio reale e di prove funzionali e prestazionali.

**Criterio di successo:** un deployment composto da gateway + SQLite + Valkey Bundle, capace di erogare richieste LLM con routing, guardrails, rate/token/budget limiting, cache esatta/semantica e alerting, senza dipendenze runtime da PostgreSQL, ClickHouse, OTel Collector o metrics-worker.
