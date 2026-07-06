# Simabo — Volunteer Upload (browser → Drive diretto)

Raccolta file dei volontari senza data-entry: il volontario mette nome/cognome,
carica foto e video, i file vengono rinominati e finiscono in una cartella Drive
di **staging**. I byte del file **non passano da n8n**: n8n apre solo una
*resumable upload session* su Google Drive e il browser carica direttamente su Google.

## Pezzi

| Pezzo | Dove |
|-------|------|
| Pagina form | `index.html` (questo folder) |
| Portiere sessioni | n8n `Simabo - Drive Upload Session (resumable)` — ID `xYv0Z8bsuBeyr7u6` |

## Flusso

```
[index.html] --POST {fileName, mimeType}--> [n8n webhook] --initiate--> [Drive API]
     |                                            |
     |<-------------- { uploadUrl } --------------|
     |
     └--PUT byte del file--------------------------> [Google Drive]  (diretto)
```

Naming file: `AAAA-MM-GG_Cognome-Nome_NNN.ext` (accenti/spazi normalizzati,
contatore sequenziale per invio).

## Setup

### 1. n8n
1. Apri il workflow `Simabo - Drive Upload Session (resumable)`.
2. Nodo **Init Resumable Session** → assegna la credenziale **Google Drive OAuth2**.
   - La credenziale deve avere lo scope `drive` (o `drive.file`); se serve, ri-autorizza.
3. Stesso nodo → **Body (JSON)**: sostituisci `PASTE_STAGING_FOLDER_ID` con l'ID
   della cartella STAGING di Drive (lo trovi nell'URL della cartella dopo `/folders/`).
4. **Attiva** il workflow e copia il **Production URL** del webhook.

### 2. Pagina
1. In `index.html`, imposta `WEBHOOK_URL` col Production URL del webhook.
2. Ospita la pagina (una delle due):
   - **GitHub Pages** (come `simabo-animal-cards`): push in un repo, Pages da `main`.
   - **Pagina WordPress**: incolla l'HTML in un blocco Custom HTML / snippet.
3. Dai il link ai volontari.

## Test locale

```bash
cd simabo/volunteer-upload
python3 -m http.server 8080
# apri http://localhost:8080
```

## Caveat da verificare al primo test — CORS sul PUT

Il browser fa il **PUT diretto** all'`uploadUrl` di Google. L'endpoint
`googleapis.com/upload/...` supporta CORS, quindi in teoria funziona da qualsiasi
origin. **Va confermato dal vivo**: se il PUT viene bloccato da CORS, le opzioni sono:

- servire la pagina da un dominio Google-friendly, oppure
- passare a upload **chunked** con header `Content-Range` (stesso session URI), oppure
- fallback: proxy del PUT via n8n (ma reintroduce il peso su n8n — da evitare).

## Limiti attuali / possibili migliorie

- Upload **sequenziale** (un file alla volta) — semplice e stabile su rete mobile.
- PUT in **un colpo solo**: se cade la rete a metà, quel file riparte da zero.
  Per veri "resume" su file enormi si può passare a chunk da ~8–16 MB con `Content-Range`.
- Nessuna autenticazione sul form: chiunque abbia il link può caricare nella cartella staging.
