# Documentazione: Aggregatore Dati Finanziari SEC & FMP

Questo documento descrive il progetto per aggregare informazioni aziendali e dati finanziari da due fonti principali: la **SEC EDGAR API** e la **Financial Modeling Prep (FMP) API**.

## 1. Presentazione dell'MVP (Minimum Viable Product)

### Obiettivo Principale

Lo scopo primario di questo progetto è creare e mantenere un dataset finanziario consolidato e arricchito per l'analisi di società quotate. Il processo combina dati anagrafici, informazioni di mercato e dati finanziari dettagliati provenienti sia da fonti ufficiali (SEC) che da un provider di dati finanziari (FMP). Le fasi principali includono:

1.  **Costruzione Dataset Tickers**:
    *   Recuperare l'elenco completo delle aziende registrate presso la SEC (con CIK).
    *   Integrare dati di mercato da Nasdaq (capitalizzazione, settore, industria, IPO).
    *   Unire i dati SEC e Nasdaq basandosi sul ticker.
    *   Pulire il dataset rimuovendo entità non desiderate (fondi, trust, SPAC) tramite filtri su nomi e descrizioni SIC.
    *   Arricchire i dati recuperando informazioni aggiuntive dall'endpoint `submissions` della SEC API (tipo entità, SIC, stato incorporazione) in modo asincrono e rispettando i rate limit.
    *   Consolidare e aggiornare un dataset di ticker di riferimento, gestendo eventuali discrepanze o aggiunte rispetto a un dataset preesistente (recuperato tramite API interna).
    *   Salvare il dataset arricchito dei ticker (es. su S3 e localmente).
2.  **Estrazione Dati Finanziari SEC (Company Facts)**:
    *   Recuperare dati finanziari dettagliati (XBRL facts) dall'endpoint `companyfacts` della SEC API per ciascun CIK valido.
    *   Gestire le tassonomie `us-gaap` e `ifrs-full`.
    *   Estrarre i singoli punti dato (valore, data, unità, periodo fiscale, form).
    *   Filtrare per mantenere solo i dati rilevanti (es., dal 2017 in poi).
    *   Salvare i dati finanziari estratti in batch (es. file Parquet) per gestirne il volume.
3.  **Estrazione/Aggiornamento Dati Finanziari FMP**:
    *   Identificare le aziende che richiedono un aggiornamento dei dati finanziari tramite FMP (es., quelle con potenziali conflitti nei dati SEC o quelle per cui mancano dati recenti).
    *   Recuperare i bilanci (Income Statement, Balance Sheet, Cash Flow) dall'API FMP per i simboli selezionati, utilizzando una API Key.
    *   Aggiornare file JSON locali (`income.json`, `balance.json`, `cash.json`) che fungono da database per i dati FMP. L'aggiornamento prevede la sostituzione dei dati per un'azienda specifica, integrando eventualmente dati storici non presenti nella nuova risposta API.
    *   Tracciare le aziende processate per evitare chiamate ripetute.

### Risultato Atteso

Il progetto mira a produrre e mantenere aggiornati i seguenti dataset:

1.  **Dataset Tickers Consolidato**: Un file (es. Parquet su S3, JSON locale) contenente un elenco pulito e arricchito di aziende, con informazioni anagrafiche, di mercato (Nasdaq) e dettagli SEC (CIK, SIC, ecc.).
2.  **Dataset SEC Financial Facts**: Una serie di file (es. Parquet) contenenti dati finanziari storici dettagliati estratti dai filing XBRL SEC, strutturati in formato long.
3.  **Dataset FMP Financial Statements**: File JSON locali (`income.json`, `balance.json`, `cash.json`) contenenti i dati dei bilanci annuali recuperati da FMP, strutturati per azienda (CIK).

Questi dataset combinati forniscono una base dati ricca per analisi finanziarie, screening, backtesting e modellazione.

## 2. Documentazione Tecnica

### 2.1 Dipendenze Principali

*   `pandas`: Manipolazione e analisi dati.
*   `requests`: Richieste HTTP sincrone (per API SEC iniziale, API interna, API FMP).
*   `asyncio`, `aiohttp`: Gestione richieste HTTP asincrone (per API SEC `submissions` e `companyfacts`).
*   `tqdm.asyncio`: Barre di progresso per operazioni asincrone.
*   `boto3`, `awswrangler`: Interazione con AWS S3 (opzionale, per storage).
*   `json`: Lavorare con dati JSON.
*   `re`: Espressioni regolari per pulizia dati.
*   `os`, `glob`, `pathlib`: Interazione con file system.
*   `collections.deque`: Utilizzato nel rate limiter SEC.
*   `nest_asyncio`: Per esecuzione `asyncio` in Jupyter Notebook.

### 2.2 Flusso di Esecuzione

Lo script è strutturato per essere eseguito in celle di un Jupyter Notebook, seguendo questi passaggi principali:

1.  **Setup**: Import librerie, definizione headers HTTP (con User-Agent per SEC).
2.  **Fase 1: Gestione Tickers e Dati Anagrafici SEC**
    *   Recupero Tickers SEC (`getSecTickers`).
    *   Caricamento e preprocessing dati Nasdaq da file CSV.
    *   Merge iniziale Nasdaq + SEC tickers.
    *   Pulizia iniziale basata su nomi Nasdaq (`exclusion_patterns`).
    *   Recupero asincrono dati SEC `submissions` (`SECDataFetcher`):
        *   Utilizzo di `aiohttp`, `AsyncRateLimiter` (per 10 req/sec SEC) e `asyncio.Semaphore`.
        *   Gestione errori (404, 429, 5xx, Timeout).
        *   Estrazione campi (`extract_company_info`).
        *   Aggiornamento del DataFrame principale con colonne `sec_*`.
    *   Pulizia finale basata su nomi e `sec_sicDescription`.
    *   Aggiornamento/Join con dataset ticker preesistente (da API interna).
    *   Salvataggio/Upload del dataset ticker finale (es. Parquet su S3, JSON locale).
3.  **Fase 2: Estrazione Dati Finanziari SEC (Company Facts)**
    *   Utilizzo della classe `SECCompanyFactsFetcher`.
    *   Caricamento CIK dal dataset ticker finale.
    *   Recupero asincrono dati SEC `companyfacts` (simile a `SECDataFetcher`, con stesso rate limiter).
    *   Estrazione dettagliata dei "facts" XBRL (`extract_financial_data`) per `us-gaap` e `ifrs-full`.
    *   Filtraggio temporale (es. >= 2017).
    *   Salvataggio dei "facts" estratti in batch (file Parquet locali).
    *   Registrazione CIK falliti in file CSV.
4.  **Fase 3: Estrazione/Aggiornamento Dati Finanziari FMP**
    *   Identificazione Conflitti/Mancanze: Analisi dei dati FMP esistenti (file JSON) per identificare aziende con dati potenzialmente duplicati per anno (`conflicts_df`) o aziende per cui manca l'ultimo anno fiscale.
    *   Selezione Batch: Selezione di un sottoinsieme di aziende da aggiornare (es. quelle in `conflicts_df` o quelle senza dati recenti), limitando il numero per esecuzione.
    *   Recupero Dati FMP:
        *   Iterazione sui simboli selezionati.
        *   Chiamate sincrone (`requests`) agli endpoint FMP per Income Statement, Balance Sheet, Cash Flow, utilizzando una API Key.
        *   Gestione errori HTTP.
    *   Aggiornamento JSON Locali (`bulk_update_json_file`):
        *   Caricamento del file JSON esistente (`income.json`, `balance.json`, `cash.json`).
        *   Per ogni azienda aggiornata, sostituzione dei dati esistenti con quelli nuovi recuperati da FMP.
        *   Logica opzionale per integrare/mantenere anni fiscali più vecchi presenti nel file JSON ma non nella nuova risposta API (se il simbolo corrisponde).
        *   Salvataggio del file JSON aggiornato.
    *   Tracciamento: Aggiornamento di un file CSV (`processed.csv`) per registrare i simboli FMP processati ed evitarne il recupero in esecuzioni successive.
5.  **Validazione e Esplorazione**: Celle opzionali per analizzare i dati scaricati (sommari, esplorazione per CIK).

### 2.3 Rate Limiting e Gestione API

*   **SEC API**: Viene implementato un `AsyncRateLimiter` rigoroso per garantire il rispetto del limite di 10 richieste/secondo globali. Questo limiter usa una `deque` e un `asyncio.Lock` per serializzare il controllo del tempo tra le richieste.
*   **FMP API**: Le chiamate sono sincrone (`requests`) e utilizzano una API Key. Lo script non implementa un rate limiter esplicito per FMP, ma processa le aziende in batch (es. 80 per esecuzione) per limitare il numero totale di chiamate per sessione. I limiti FMP sono generalmente più alti (es. chiamate al minuto o giornaliere) rispetto a quelli SEC.
*   **API Interna**: Utilizza un token di autorizzazione statico per recuperare il dataset ticker esistente.

### 2.4 Gestione Errori

*   **SEC API (Async)**: Gestione robusta di 404, 429 (con backoff esponenziale), 5xx (con retry), Timeout, `aiohttp.ClientError` ed eccezioni generiche. I fallimenti definitivi vengono loggati.
*   **FMP API (Sync)**: Gestione di base degli errori HTTP (`response.status_code != 200`) e delle eccezioni `requests.exceptions.RequestException`.
*   **File I/O**: Blocchi `try-except` per la lettura/scrittura di file JSON, CSV, Parquet.

### 2.5 Struttura Dati Output

*   **Tickers**: DataFrame Pandas (salvato come Parquet/JSON) con colonne aggregate da Nasdaq, SEC Submissions e API interna.
*   **SEC Financial Facts**: File Parquet multipli in formato "long" (una riga per fact XBRL).
*   **FMP Financial Statements**: Tre file JSON (`income.json`, `balance.json`, `cash.json`) contenenti liste di dizionari, dove ogni dizionario rappresenta un'azienda (identificata da `cik_str_zfill`) e contiene un array `data` con i bilanci annuali.
*   **File di Supporto**: `conflicts_df.csv` (aziende FMP con potenziali duplicati), `processed.csv` (aziende FMP aggiornate), `failed_ciks_*.csv` (CIK falliti durante fetch SEC).

### 2.6 Note Importanti

*   **User-Agent (SEC)**: È **fondamentale** fornire un User-Agent valido e informativo (con email) nelle richieste alla SEC. Sostituire i placeholder nel codice.
*   **API Key (FMP)**: È necessaria una API Key valida per accedere ai dati FMP. Gestire la chiave in modo sicuro (non hardcoded).
*   **Credenziali/Token**: Credenziali AWS e token per API interne non dovrebbero essere hardcoded. Utilizzare metodi sicuri (ruoli IAM, variabili d'ambiente, gestori di segreti).
*   **Configurazione**: Esternalizzare parametri come rate limit, numero worker, percorsi file, API key/token in file di configurazione per maggiore flessibilità e sicurezza.