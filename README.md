# Simabo — Volunteer Upload (browser → Drive diretto)

Raccolta file dei volontari senza data-entry: il volontario mette nome/cognome,
carica foto e video, i file vengono rinominati e finiscono in una cartella Drive
di **staging**. I byte del file **non passano da n8n**: n8n apre solo una
*resumable upload session* su Google Drive e il browser carica direttamente su Google.

## Pezzi

| Pezzo | Dove |
|-------|------|
| **Link per i volontari** | https://simabo.org/upload-content-library/ (pagina WP con iframe) |
| Sorgente (GitHub Pages) | https://valeriorossidigital.github.io/simabo-volunteer-upload/ |
| Repo | https://github.com/ValerioRossiDigital/simabo-volunteer-upload (pubblico) |
| Pagina form | `index.html` (questo folder) |
| Embed WordPress | `wordpress-embed.html` — iframe + auto-altezza, per pagina `simabo.org` |
| Portiere sessioni | n8n `Simabo - Drive Upload Session (resumable)` — ID `xYv0Z8bsuBeyr7u6` |

Il repo è pubblico ma **non contiene segreti**: la credenziale Google vive in n8n,
la pagina espone solo l'URL del webhook (già pubblico di suo).

## Flusso

```
[index.html] --POST {fileName, mimeType}--> [n8n webhook] --initiate--> [Drive API]
     |                                            |
     |<-------------- { uploadUrl } --------------|
     |
     └--PUT byte del file--------------------------> [Google Drive]  (diretto)
```

Naming file: `TYPE Animal (First Last) [code].ext`
- `TYPE` uppercase (DOG, CAT, VOLUNTEER, CLINIC, HOSTEL, GENERIC)
- `Animal` solo per dog/cat, se il volontario lo inserisce (opzionale)
- `(First Last)` = chi ha caricato
- `[code]` = codice univoco `sessione-progressivo` (es. `a3f9-001`) anti-collisione
- Niente data nel nome: c'è già `createdTime` su Drive.

Esempi: `DOG Luna (Mario Rossi) [a3f9-001].jpg` · `CLINIC (Mario Rossi) [a3f9-002].jpg`

Le **immagini** vengono ottimizzate lato browser prima dell'upload (lato lungo max
2560px, JPEG q0.85; HEIC convertito in JPEG via `heic2any`). I **video** restano intatti.

## Setup

### 1. n8n (già configurato)
1. Workflow `Simabo - Drive Upload Session (resumable)`, **attivo**.
2. Nodo **Init Resumable Session**: credenziale **Google Drive OAuth2** (scope `drive`)
   assegnata; nel **Body (JSON)** il `parents` punta alla cartella STAGING
   (folder ID `1zZXFmzpiTFAZEt2PkZsuZNPEQrFklPya`).
3. Webhook: `https://n8n.gorillatribe.net/webhook/simabo-drive-session`.
4. **Settings → Error Workflow** = sé stesso (serve per far scattare l'email d'errore).

> Il nodo webhook deve restare **Enabled**: se disabilitato, il form prende 404.

### 2. Pagina (GitHub Pages)
`WEBHOOK_URL` in `index.html` è già impostato. Deploy automatico: ogni push su
`main` aggiorna la pagina live in ~1 min (GitHub Pages da `main`, root).

### 3. Embed su WordPress (`simabo.org`)
Pubblicata su **https://simabo.org/upload-content-library/** — è il link da dare ai
volontari. La pagina live è incorporata via **iframe** in una pagina WordPress:
1. Nuova pagina WP → widget Elementor **HTML** (o blocco Gutenberg HTML personalizzato).
2. Incolla tutto il contenuto di `wordpress-embed.html`.
3. Pubblica. L'iframe si **auto-dimensiona** (la pagina invia la sua altezza via
   `postMessage`, il listener nello snippet regola l'iframe → niente scrollbar interna).
4. Se i dati sembrano vecchi: svuota la cache **LiteSpeed** per quella pagina.

Quando è dentro l'iframe la pagina è "nuda" (niente logo/sfondo/intro, sfondo
trasparente): l'header lo dà WordPress.

## Monitoraggio errori

Il workflow n8n ha un ramo **Error Trigger → Gmail** che manda un'email a
`simabo.marketing@gmail.com` quando un'esecuzione fallisce (es. token Drive scaduto,
folder ID errato). Richiede che **Settings → Error Workflow** del workflow punti a
sé stesso. Nota: un batch di N file falliti = N email.

## Test locale

```bash
cd simabo/volunteer-upload
python3 -m http.server 8080   # apri http://localhost:8080 (non file:// → CORS)
```

## Note tecniche

- **CORS sul PUT (confermato)**: il browser consegna i byte a Google, ma la *lettura*
  della risposta di Google è bloccata da CORS. È **innocuo**: il file è già salvato
  quando il body è stato inviato. La pagina considera l'upload riuscito su
  `xhr.upload.onload` (byte consegnati), ignorando la risposta illeggibile → niente
  falsi errori rossi.
- **Immagini** ottimizzate lato browser (max 2560px lato lungo, JPEG q0.85; HEIC→JPEG
  via `heic2any` caricata solo quando serve). Fail-safe: se la compressione fallisce,
  carica l'originale. **Video** intatti (transcodifica pesante → rimandata alla fase-2).
- Upload **sequenziale** (un file alla volta) — stabile su rete mobile.
- PUT in **un colpo solo**: se cade la rete a metà, quel file riparte da zero. Per veri
  "resume" su file enormi si può passare a chunk da ~8–16 MB con `Content-Range`.
- Nessuna autenticazione sul form: chiunque abbia il link può caricare nella cartella staging.
- `simabo-logo-white.webp` resta nel repo ma non è più referenziato (logo dato da WP).
