# Privacy Blur 🔒

Sito **in un solo file HTML** che prende in input una foto e **sfoca solo le
persone** (sagoma intera, testa compresa): si apre `index.html` in un browser e
si trascina dentro un'immagine. Tutto avviene **nel browser** — nessun file
viene caricato su un server, nessuna installazione, nessun account.

**Stato (18/09/2026):** repo creata il **18/09/2026** dal file scaricato dal
generatore DeepSeek il 17/09/2026 (`deepseek_html_20260917_04c612.html`), qui
portato come `index.html` **senza modifiche al codice** e documentato con questo
README. La repo GitHub è **pubblica**: `origin` =
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

## File presenti

| File           | Ruolo                                                                  |
|----------------|------------------------------------------------------------------------|
| `index.html`   | il sito: HTML + CSS + JavaScript in un solo file, zero da installare    |
| `README.md`    | questo file: uso, come funziona, limiti e note operative                |
| `.gitignore`   | esclusioni: foto di prova (personali), versioni precedenti, file di sistema |

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
- **Provenienza del file.** `index.html` è **l'ultimo file scaricato** dal
  generatore DeepSeek il 18/09/2026 alle 00:31
  (`~/Downloads/deepseek_html_20260917_04c612.html`, MD5
  `62e006162bf051f56c9c2b08769e72ed`), copiato **senza toccare il codice**.
  In `~/Downloads` ci sono due versioni precedenti dello stesso sito
  (`deepseek_html_20260917_702af5.html` e la copia «(1)»): una usava
  `face-api.js`, l'altra solo BodyPix; quella portata qui è la più completa
  (BodyPix **+** rilevamento volti **+** maschera della testa **+** espansione
  regolabile). Le versioni vecchie non sono in repo (`.gitignore` le esclude con
  `deepseek_html_*.html`).
- **Nessun server e nessun test nel repo.** Il sito è un file HTML unico: non
  c'è un'app da avviare né `requirements.txt`. Per provarlo:
  `python3 -m http.server 8899` dalla cartella e poi
  <http://localhost:8899/index.html> (anche l'apertura diretta con `file://`
  funziona: le librerie arrivano da CDN).
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

