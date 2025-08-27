# CRAFT Prompt Builder

Un semplice tool web per progettare prompt C.R.A.F.T. in italiano. L'interfaccia statica (`index.html`) guida l'utente in 3 step e dialoga con un endpoint PHP (`api/communication.php`) che chiama l'API OpenAI Responses.

## Requisiti
- PHP 8.0+ con estensione cURL abilitata
- Composer
- Una chiave API OpenAI valida
- Accesso a rete in uscita verso `https://api.openai.com`

## Struttura del progetto
- `index.html`: UI a 3 step (raccolta obiettivo, form dinamico, prompt finale)
- `api/communication.php`: endpoint PHP che:
  - legge input JSON dal client
  - costruisce il system prompt C.R.A.F.T.
  - chiama l'endpoint OpenAI Responses
  - restituisce JSON al client
- `api/logs/openai.log`: log delle richieste/risposte
- `composer.json` / `composer.lock`: gestione dipendenze
- `vendor/`: dipendenze installate (tra cui `vlucas/phpdotenv`)

## Installazione
1) Installazione dipendenze PHP
```bash
composer install --no-interaction --no-progress
```

2) Configurazione variabili ambiente
Creare un file `.env` nella cartella `api/` con il contenuto:
```bash
# /api/.env
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Note:
- Il file viene caricato da `api/communication.php` tramite `vlucas/phpdotenv`.
- Se la variabile manca, l'endpoint risponde con errore 500.

## Avvio locale
È un progetto statico + PHP. Si può usare il server integrato di PHP dalla root del progetto:
```bash
php -S localhost:8080 -t .
```
Poi apri `http://localhost:8080/` nel browser.

In alternativa, è possibile servire la sola API su una porta dedicata:
```bash
php -S localhost:8081 -t api
```
E modificare le chiamate fetch in `index.html` per puntare a `http://localhost:8081/communication.php`.

## Flusso applicativo (UI)
- Step 1: l'utente inserisce l'obiettivo. La UI inizializza la chat e invia una prima richiesta di bootstrap.
- Step 2: la UI richiede all'API un JSON con i campi necessari oppure un `direct_prompt`. Se riceve `fields`, costruisce dinamicamente il form; se riceve `direct_prompt`, salta allo step 3.
- Step 3: quando l'utente invia i valori, la UI richiede all'API il prompt finale e lo mostra in un `<pre>`.

Le chiamate avvengono tramite `fetch` verso `api/communication.php`, passando:
```json
{
  "messages": [
    { "role": "user", "content": "..." }
  ],
  "responseFormat": "text" | "json_object" | "json_schema"
}
```
La risposta dell'API è del tipo:
```json
{ "text": "...", "raw": { /* risposta OpenAI completa */ } }
```

## Endpoint backend
Percorso: `api/communication.php`
- Metodo: POST
- Headers richiesti: `Content-Type: application/json`
- Corpo: vedi payload sopra
- Risposte:
  - 200: `{ "text": string|null, "raw": object }`
  - 4xx/5xx: `{ "error": { ... } }`

Dettagli implementativi chiave:
- Caricamento `.env` nella cartella `api/`:
  ```php
  $dotenv = Dotenv\Dotenv::createImmutable(__DIR__);
  $dotenv->load();
  ```
- Recupero chiave: `$_ENV['OPENAI_API_KEY']`
- Endpoint OpenAI usato: `POST https://api.openai.com/v1/responses`
- Estrazione testo dalla risposta: itera `output[]` e prende il primo blocco `type === 'message'`.
- Logging su file `api/logs/openai.log` (creato se mancante).

## Modello e formato di risposta
Nel payload inviato a OpenAI viene impostato:
- `model`: `gpt-5-2025-08-07` (sostituire con un modello disponibile nella tua organizzazione, es. `gpt-4.1-mini` o quello attuale)
- `text.format` quando `responseFormat === 'json_object'`:
  ```json
  { "text": { "format": { "type": "json_object" } } }
  ```

La UI usa tre modalità:
- `text`: per ottenere il prompt finale testuale
- `json_object`: per ottenere lo schema dei campi da visualizzare
- `json_schema`: predisposto nel codice ma non utilizzato di default

## Sicurezza
- Mantieni la chiave `OPENAI_API_KEY` solo lato server (nel `.env` di `api/`).
- Non committare `.env` nel VCS.
- Valuta rate limiting/log redaction in produzione.

## Troubleshooting
- Errore 500 con messaggio `OPENAI_API_KEY missing in .env`:
  - Verifica il file `/api/.env` e il valore della variabile.
- Timeout o `Curl error`:
  - Verifica connettività verso `api.openai.com` e aumenta `CURLOPT_TIMEOUT` se necessario.
- La UI mostra "JSON non valido" in Step 2:
  - Il modello deve restituire SOLO JSON valido quando richiesto `json_object`. Riduci creatività (temperature) o usa un modello più preciso.
- Nessun testo in risposta:
  - Il parsing cerca `output[].type == 'message'`. Controlla il formato `raw` nel log per adeguare l'estrazione se l'API cambia.

## Personalizzazioni
- Modello: modifica la variabile `$payload['model']` in `api/communication.php`.
- Prompt di sistema: modifica `$systemContent` per adattare regole e stile.
- Logging: cambia percorso/verbosità nella sezione "Log" della API.
- UI: personalizza campi, stile e testi in `index.html`.

## Deploy
- Servire i file statici con un web server (Nginx/Apache) e configurare PHP-FPM o mod_php per la cartella `api/`.
- Impostare variabili ambiente di produzione o il file `/api/.env` in modo sicuro.
- Proteggere `api/logs/` se esposto pubblicamente (o spostare i log fuori dalla root web).

## Licenza
Non specificata. Aggiungi un file `LICENSE` se vuoi distribuire pubblicamente.
