# Privacy Blur + OCR → LaTeX → PDF 🔒📄

Sito **in un solo file HTML** con due mestieri, tutti **nel browser** (nessun file
caricato su un server, nessuna installazione, nessun account):

1. **sfocare solo le persone** (sagoma intera, testa compresa) in una foto;
2. **estrarre testo e formule** da un'immagine (OCR) e **compilare il PDF** in
   LaTeX, con il sorgente `.tex` sempre scaricabile.

**Stato (18/09/2026):** la repo nasce il 18/09/2026 con la parte *sfocatura* (file
DeepSeek `…_04c612.html`, portato qui com'era); lo stesso giorno è arrivata la
pagina **con l'OCR** (`…_65879b (4).html`) e **qui sono state fatte le tre
modifiche** che la fanno funzionare anche aperta con doppio clic da `file://`:
l'avviso in alto, PaddleOCR caricato **solo** via HTTP, e l'OCR di riserva con
Tesseract.js. La versione **solo-sfocatura** resta nella storia di git (commit
`ef74b55`). La repo GitHub è **pubblica**: `origin` =
`https://github.com/Alessandro1040/privacy-blur.git`, branch `main`.

## Come si usa

1. **Apri `index.html`** (doppio clic, oppure trascinalo in Chrome/Firefox/
   Safari/Edge). La prima volta serve internet: le librerie e i modelli di IA
   arrivano da CDN (TensorFlow.js 3.21, BodyPix 2.2, face-detection 1.0).
   *In alternativa*, con un server locale:
   `python3 -m http.server 8899` → <http://localhost:8899/index.html>.
2. Quando in fondo compare **«✅ Modelli pronti»**, carica la foto (clic sul
   riquadro o trascinamento) e premi **Elabora e sfoca**.
3. Confronta **Originale** e **Risultato** affiancati, poi premi
   **Scarica risultato** (PNG, `immagine_sfocata.png`) o **Reset** per
   ricominciare.
4. Per il **testo**: stessa foto caricata, premi **Estrai testo & formule** (il
   risultato compare nelle schede *LaTeX completo*, *Testo grezzo*, *Formule*),
   poi **Compila PDF** → anteprima nella scheda *PDF compilato* e download di
   `.tex` e `.pdf`.

### ⚠️ Aprire il file: doppio clic o server?

| Come apri la pagina | Cosa funziona |
|---|---|
| **Doppio clic** (`file://`) | sfocatura, OCR **Tesseract.js**, formule, PDF |
| **Server locale** (`python3 -m http.server 8899` → <http://localhost:8899/index.html>) | **tutto**, incluso l'OCR avanzato **PaddleOCR** |
| **Sito pubblicato** (`https://`) | tutto |

Il motivo è tecnico: **PaddleOCR scarica i modelli ONNX con `fetch()`**, e i
browser bloccano quelle richieste quando la pagina è aperta da `file://`
(protocollo senza origine, quindi CORS). Da `file://` la pagina **non prova
nemmeno** a caricarli: mostra l'avviso arancione in alto, usa Tesseract.js e
continua a funzionare. Da `http://` o `https://` parte invece PaddleOCR, con
Tesseract.js che resta pronto come rete di sicurezza.

La manopola **Espansione** (0–40 px, default **18**) allarga la maschera attorno
a persone e teste: più alta = più prudente (copre anche capelli e contorno),
più bassa = più preciso.

## Come funziona (la pipeline del file, con i suoi numeri)

1. l'immagine viene ridimensionata a un massimo di **1280 px** sul lato lungo
   (`MAX_DIM`);
2. **BodyPix** (MobileNetV1, `outputStride 16`, `multiplier 1.0`,
   `quantBytes 2`, `internalResolution: 'full'`, soglia **0,6**) segmenta la
   scena e **tutti i pixel "persona" entrano nella maschera**;
3. **BlazeFace** (`@tensorflow-models/face-detection`, runtime `tfjs`,
   `maxFaces 30`) cerca i volti;
4. per ogni volto trovato la maschera riceve un'**ellisse "testa intera"**:
   ~1,9× la larghezza e ~3,1× l'altezza del riquadro del volto, spostata verso
   l'alto (capelli, orecchie, contorno);
5. la maschera viene **dilatata** del valore di *Espansione* (copie offsettate
   su tre anelli) e i bordi **ammorbiditi** con un blur di 8 px (`FEATHER`);
6. si costruisce lo **strato sfocato** dell'immagine intera (blur **24 px**,
   `BLUR_PX`) e lo si **ritaglia con la maschera** (`destination-in`);
7. **composizione finale**: originale + solo lo strato mascherato → **lo sfondo
   resta identico, pixel per pixel**; in alto compare il riepilogo
   («Volti rilevati: N · area persona ≈ X% · espansione Ypx»).

## La parte OCR → LaTeX → PDF (come funziona)

1. l'immagine caricata va al motore OCR: **PaddleOCR** (PP-OCRv5, WebAssembly) se
   la pagina è servita via **HTTP**, altrimenti **Tesseract.js** (`ita+eng`), che
   funziona anche da `file://`;
2. le parole riconosciute tornano **normalizzate nella stessa forma** — `{text,
   box}`, con la casella del riquadro — qualunque sia il motore, così il resto del
   codice non cambia;
3. le regioni che «sembrano formule» (caratteri matematici nel testo, oppure
   riquadri bassi) vengono ritagliate e passate a **TrOCR-LaTeX**
   (`onnx-community/latex_finetuned-ONNX`, via Transformers.js), con una
   **deduplica per riquadro**: Tesseract restituisce una casella per *ogni parola*
   e senza quel controllo la stessa regione arriverebbe più volte al modello;
4. il testo finisce in un documento LaTeX (`article`, `babel` italiano, `amsmath`,
   `graphicx`, `geometry`) e le formule in un blocco `align*`;
5. la compilazione avviene **nel browser**: **StellarLatex** se i suoi file sono in
   `stellarlatex/` (cartella locale, non versionata), altrimenti **Siglum** dal CDN
   (`@siglum/engine`, ~45 MB la prima volta, può richiedere 1–2 minuti). Il PDF
   compare in un'anteprima con due pulsanti di download; il **`.tex` si scarica
   sempre** e si può compilare altrove (Overleaf).

## File presenti

| File           | Ruolo                                                                  |
|----------------|------------------------------------------------------------------------|
| `index.html`   | il sito: HTML + CSS + JavaScript in un solo file, zero da installare    |
| `README.md`    | questo file: uso, come funziona, limiti e note operative                |
| `.gitignore`   | esclusioni: foto di prova (personali), versioni precedenti, file di sistema |
| `stellarlatex/` | **opzionale, non versionata**: qui dentro i file di StellarLatex (`PdfTeXEngine.js` + i suoi asset) se vuoi compilare il PDF in locale invece di scaricare Siglum dal CDN |

## Limiti noti

- **Serve internet al primo caricamento** (CDN per le librerie e i modelli):
  dopo, il browser li tiene in cache. Nessuna immagine lascia il computer.
- **Non conosce i disegni**: BodyPix riconosce persone *fotografate*; con
  cartoon, manichini o persone molto coperte la maschera può essere parziale.
- **Il riquadro "testa intera" vale solo per i volti riconosciuti**: sulle foto
  di prova (paesaggi 4032×3024 con persone a figura intera, quindi volti piccoli
  dopo il ridimensionamento a 1280) BlazeFace ha rilevato **0 volti** e la
  sfocatura è arrivata dalla sola segmentazione — che copre comunque testa e
  corpo quando la persona è visibile. Per i **primi piani** il riquadro della
  testa fa la differenza.
- **Le immagini grandi diventano più piccole**: una foto 4032×3024 esce a
  1280×960 (nessuna opzione per la dimensione piena).
- **Nessuna anteprima del ritaglio**: l'*Espansione* si giudica solo dal
  risultato (e una volta elaborata c'è solo *Reset*, non un "annulla").
- **Elaborazione in un thread**: durante l'analisi la pagina resta occupata
  (secondi per foto, di più su macchine senza accelerazione grafica).

## Note operative e stato corrente (18/09/2026)

Da tenere presente nelle sessioni di lavoro successive:

- **Cartella di lavoro = clone git.** Si lavora direttamente in
  `/Users/alessandrolocurcio/.cline/data/workspaces/chat/privacy-blur/` (path
  **senza spazi**) e si committa da lì; `origin` =
  `https://github.com/Alessandro1040/privacy-blur.git`, branch `main`, repo
  **pubblica** (creata con `gh`, autenticato come Alessandro1040).
- **Provenienza del file.** La parte *sfocatura* viene dal file DeepSeek
  `…_04c612.html` (portato qui com'era il 18/09/2026); la pagina attuale è la
  versione **con l'OCR** scaricata dal generatore lo stesso giorno
  (`~/Downloads/deepseek_html_20260917_65879b (4).html`, 31,6 KB, ore 01:27),
  copiata qui e poi **modificata** con le tre correzioni per `file://` (vedi
  sotto). In `~/Downloads` restano le iterazioni precedenti dello stesso sito
  (`…_65879b` senza numero, `(1)`, `(2)`, `(3)`, `…_ffdac5*`): alcune usano
  Tesseract.js, altre PaddleOCR, una anche Siglum. **Non sono in repo**
  (`.gitignore` esclude `deepseek_html_*.html`).
- **Nessun server e nessun test nel repo.** Il sito è un file HTML unico: non
  c'è un'app da avviare né `requirements.txt`. Per provarlo:
  `python3 -m http.server 8899` dalla cartella e poi
  <http://localhost:8899/index.html>. **Da `file://` funziona tutto tranne
  l'OCR avanzato**: PaddleOCR ha bisogno di HTTP per scaricare i modelli ONNX,
  quindi da `file://` la pagina usa Tesseract.js e lo dice con l'avviso in alto
  (la tabella è in «Come si usa»).
- **Le foto di prova NON stanno in repo**: `.gitignore` esclude `*.jpg`,
  `*.jpeg`, `*.png`, `*.webp`, `*.heic`, `*.tiff`. Le foto usate per le
  verifiche (`IMG-8246.jpg`, `IMG-8250.jpg`, `IMG_9206.jpg`, 4032×3024, in
  `~/Downloads`) restano solo sul Mac.
- **Verifiche del 18/09/2026** (Chrome headless + Selenium sulla pagina servita
  da `http.server` su 8899): i modelli si caricano dalla CDN in ~40 s («✅
  Modelli pronti»); su due foto 4032×3024 l'elaborazione è passata **9/9
  controlli** — immagini ridimensionate a **960×1280**, pixel cambiati
  **10,6%** e **7,0%** (le persone, sfocate), **delta agli angoli 0,00–4,69**
  (lo sfondo resta intatto), riepilogo a schermo coerente («area persona ≈
  9,7%» e «≈ 8,5%», espansione 18 px). Nessun errore in console (compare solo
  il `favicon.ico` 404, innocuo).
- **Il rilevatore di volti funziona, ma solo se il volto è grande.** Sulle due
  foto di prova (persone a figura intera) ha rilevato **0 volti** e la maschera
  è venuta dalla sola segmentazione BodyPix; su un **primo piano di pubblico
  dominio** scaricato da Wikipedia ha rilevato **1 volto** e sfocato il
  **45,3%** dell'immagine (area persona 48,1%). Quindi il limite è la
  dimensione del volto *dopo* il ridimensionamento a 1280 px, non il codice: per
  farlo funzionare anche sui gruppi lontani bisognerebbe far girare il
  rilevatore su ritagli o su una piramide di scale invece che sulla sola
  immagine ridimensionata (**non implementato**).
- **Difetti noti, non corretti** (il file è stato portato così com'è):
  il riquadro della testa è un'**ellisse larga** (~1,9× la larghezza e ~3,1×
  l'altezza del volto) che sui primi piani può sfocare anche collo e spalle;
  l'*Espansione* non ha anteprima; non si può esportare a piena risoluzione;
  l'elaborazione occupa la pagina (un solo thread).
- **Come si aggiorna il sito**: incollare la nuova versione sopra `index.html`,
  riprovare con la procedura qui sopra, aggiornare la sezione «Come funziona»
  (e le presenti note) se cambiano librerie o soglie, quindi commit + push su
  `main`.
- **Le tre correzioni per `file://` (18/09/2026, fatte in questa repo).** Il file
  arrivato da DeepSeek caricava PaddleOCR **sempre** (tag `onnxruntime` fisso + un
  modulo ESM): aperto con doppio clic, il `fetch()` dei modelli ONNX veniva
  bloccato dal browser per CORS e in console restava un errore poco comprensibile.
  Ora:
  1. uno script in testa legge `location.protocol`: se non è `http(s)` mostra
     l'avviso `#fileProtocolWarning` (stile `.alert`, aggiunto al CSS);
  2. ONNX Runtime e il modulo PaddleOCR vengono **creati solo su http(s)**: da
     `file://` non si scaricano nemmeno, e in console resta un solo avviso chiaro;
  3. Tesseract.js è caricato **sempre** e `initOcr()` lo prova per primo; su
     http(s) aspetta `paddleocr-ready` (max **30 s**) e usa PaddleOCR, altrimenti
     resta su Tesseract.js dicendolo nello stato (`✅ OCR pronto (Tesseract.js)`).
     Il riconoscimento normalizza l'output dei due motori nella stessa forma
     (`{text, box}`) e **deduplica i riquadri** prima del modello delle formule
     (Tesseract restituisce una casella per ogni parola). Sistemato anche lo stato
     che prima scriveva «⚠️ OCR pronto» (un avviso per una cosa che andava bene).
- **Verifiche del 18/09/2026 (dopo le correzioni).** I 5 blocchi `<script>` della
  pagina si compilano (JavaScriptCore, `new Function`); in **Chrome headless**:
  da `file://` → avviso **visibile** (`display: block`), **nessun** tag
  `onnxruntime` nel DOM, Tesseract.js presente; da `http://localhost:8123` →
  avviso **nascosto** (`display:none`), tag `onnxruntime` **presente**,
  Tesseract.js presente. **Non ancora provato end-to-end** il riconoscimento su una
  foto vera: serve un'immagine con testo e i modelli da CDN (il flusso resta quello
  di prima, cambia solo quale motore risponde).

