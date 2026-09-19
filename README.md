# Privacy Blur + OCR → LaTeX → PDF 🔒📄

Sito **in un solo file HTML** con due mestieri, tutti **nel browser** (nessun file
caricato su un server, nessuna installazione, nessun account):

1. **sfocare solo le persone** (sagoma intera, testa compresa) in una foto;
2. **estrarre testo e formule** da un'immagine (OCR) e **compilare il PDF** in
   LaTeX, con il sorgente `.tex` sempre scaricabile;
3. *(opzionale, spento di default)* **correggere il testo con un LLM** — locale
   (LM Studio/Ollama) o online — e farlo trascrivere in LaTeX pulito.

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

## La correzione con un LLM (opzionale — sezione 3 della pagina)

L'OCR sbaglia, soprattutto sulla **scrittura a mano**: un modello linguistico può
correggere il testo o trascriverlo in LaTeX. La sezione è **spenta di default**: se
non la accendi tu, niente esce dal computer.

| Controllo | Cosa fa |
|---|---|
| **Usa l'LLM** | l'interruttore. All'apertura è **sempre spento** e lo stato lo dice: «LLM spento: niente esce da questo computer» |
| **Tipo di servizio** | `OpenAI-compatibile` (LM Studio · Ollama · OpenAI · vLLM…), `Anthropic`, `Google` |
| **URL base** | predefinito `http://localhost:1234/v1` (LM Studio); per OpenAI `https://api.openai.com/v1` |
| **Modello** | es. `qwen2.5-vl-7b-instruct` in LM Studio, oppure `gpt-5` / `claude-…` / `gemini-2.5-flash` online |
| **Chiave API** | solo per i servizi online: si scrive a mano e **non viene salvata** (resta nella scheda); con un URL locale il campo si svuota da sé |
| **manda anche l'immagine** | consigliato con la scrittura a mano: manda la foto ridotta a ~1600 px sul lato lungo (meno token). Spenta, va solo il testo dell'OCR |
| **✍️ Correggi il testo** | corregge gli errori evidenti senza inventare (dove non si legge: `[illeggibile]`) |
| **📐 Trascrivi in LaTeX** | riscrive il contenuto come corpo di documento LaTeX, con le formule fra `$...$` |
| **🧹 Pulisci e riorganizza** | riorganizza gli appunti in paragrafi ed elenchi, senza aggiungere nulla |
| **↻ Rifai il documento LaTeX** | prende il testo dell'LLM e rigenera il `.tex` (e lo compila). Il testo dell'LLM **non** viene «protetto» come quello dell'OCR (è già LaTeX): così la matematica `$...$` resta viva |

Il risultato finisce in una scheda a parte, **«Testo corretto (LLM)»**: l'originale
dell'OCR resta in «Testo grezzo», così si vede sempre cosa è cambiato.

**Privacy.** Le impostazioni non segrete (servizio, URL, modello, «manda l'immagine»)
si ricordano in `localStorage`; la **chiave API no**: si riscrive a ogni sessione e
non finisce mai nel file (la repo è **pubblica**). Con un URL locale il traffico non
esce dalla macchina; con un servizio online escono il testo e — se la spia è accesa —
l'immagine ridotta: la pagina lo avverte nello stato mentre aspetta.

**Se compare «Failed to fetch»** il browser non riesce proprio a parlare con il
server. Cause, in ordine di frequenza:

1. **il server locale è acceso ma senza CORS**: il flag si attiva con
   `lms server stop && lms server start --cors` (oppure con l'interruttore
   *Enable CORS* nella pagina *Developer*). Senza CORS l'intestazione
   `Access-Control-Allow-Origin` non arriva e il browser blocca la chiamata — è
   l'errore che ho preso io al primo tentativo;
2. il server **non è acceso**: `lms server status`;
3. **nessun modello caricato** e caricamento automatico spento: LM Studio risponde
   *"No models loaded. Please load a model in the developer page or use the 'lms
   load' command."* — la pagina mostra il messaggio del server;
4. l'URL: meglio `http://127.0.0.1:1234/v1` che `localhost`, perché a volte il
   browser prova l'IPv6 `::1` dove il server non ascolta.

Da terminale, con la CLI di LM Studio (`~/.lmstudio/bin/lms`): `lms ls` per i nomi
dei modelli, `lms load qwen/qwen2.5-vl-7b` per caricare quello vision,
`lms server start --cors` per accendere il server in modo che il browser possa
usarlo.

**Se il PDF non esce** (`Compilazione fallita ... nessun log`): è successo premendo
«↻ Rifai il documento LaTeX» **mentre la pagina stava ancora scaricando i modelli**
(lo stato lo dice: «⏳ modello formule in caricamento»). Il compilatore della pagina
è un pdflatex in WebAssembly e in quel momento può fallire in silenzio, senza
lasciare log. Da qui due rimedi: la pagina **riprova da sola una volta** e scrive
l'esito nello stato della sezione LLM (che i messaggi di caricamento non coprono);
e se anche il secondo tentativo fallisce, aspetta che lo stato dica che tutto è
pronto, oppure scarica il `.tex` e compilalo tu. Il LaTeX prodotto dal modello è
valido: gli stessi documenti compilano con pdflatex 2026 senza errori.

**Un LLM locale (LM Studio è già installato):** apri LM Studio → *Developer* →
*Start server* (ascolta su `http://localhost:1234/v1`, CORS già aperto) e carica un
modello; per la scrittura a mano serve un modello **vision**
(`qwen2.5-vl-7b-instruct`, `llama-3.2-11b-vision`, `minicpm-v`…). Nel campo «Modello»
si scrive l'identificativo che LM Studio mostra.

**Verifiche del 19/09/2026 — con un modello vero.** Con LM Studio
(`lms server start --cors`, poi `lms load qwen/qwen2.5-vl-7b`, 6,04 GB, 21 s di
caricamento) e la pagina in Chrome pilotato da Selenium:

| Prova | Esito |
|---|---|
| Correzione di un testo OCR di prova (solo testo) | **4,1 s** — `unrdiscontinuita`→`un' discontinuità`, `interuallo`→`intervallo`, `e positiva`→`è positiva`, formula `\frac` intatta |
| Trascrizione della **foto a mano** (`IMG_9206.jpg`, ridotta a 1600 px, ~642 KB) | **22,8 s** — 472 caratteri di matematica **coerente** (limiti, asintoti obliqui), contro le 44 righe senza senso dell'OCR locale |
| Preset «📐 Trascrivi in LaTeX» sulla stessa foto | **31,6 s** (poi 37,7 s con il prompt rafforzato) — `align*`, `\lim_{x \to \infty}`, `\frac`, `\sin`, `\mathbb R` |
| Correzione di un testo OCR di prova con la pagina aperta **con doppio clic** (`file://`, origine `null`) | **3,0 s** — funziona anche senza un server locale per la pagina: `Nell interuallo`→`Nell'intervallo`, `e positiva`→`è positiva` |

Il primo tentativo di LaTeX **non compilava**: `! Missing $ inserted`, perché il
modello metteva `x \to \infty` e un `\begin{cases}` **fuori** dalla matematica. Da
qui la regola aggiunta al prompt («TUTTA la matematica fra `$...$`, gli ambienti
matematici solo dentro un ambiente matematico, il documento deve compilare con
pdflatex»): con quella, il documento prodotto dalla foto **compila** con pdflatex
2026 (uscita 0, PDF di 111 KB, nessun errore). Vale lo stesso motore che usa la
pagina via Siglum.

Restano da tenere d'occhio due cose: la trascrizione può avere punti incerti
(`[illeggibile]` non sempre rispettato) e va confrontata con «Testo grezzo»; e un
modello da 7 B ogni tanto sbaglia grammatica da solo (nel test ha scritto «un'
discontinuità» invece di «una discontinuità»).

**Verifiche con il server finto** (senza modello): la sezione è stata provata con un
**server finto OpenAI-compatibile** (Python, porta 8126, CORS) e Chrome pilotato da
Selenium:
all'apertura la sezione è **spenta** e i quattro pulsanti disabilitati; accendendola
con un testo OCR presente i tre pulsanti si abilitano; premendo «Correggi il testo»
la pagina chiama `/v1/chat/completions` con il prompt di correzione e mostra il
testo tornato (badge «3 righe»); «↻ Rifai il documento LaTeX» rigenera il `.tex`
**mantenendo la matematica** (`$y = 3x - 1$`); a spia spenta non parte nessuna
richiesta. Le funzioni pure `richiestaLlm` e `testoRispostaLlm` sono provate in
JavaScriptCore su tutti e tre i provider: OpenAI-compatibile (con e senza immagine,
con l'intestazione `Authorization`), Anthropic (`/v1/messages`, `x-api-key`, parte
`image`), Google (`:generateContent`, `inline_data`), più risposta vuota o malformata
→ stringa vuota.

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
- **Le correzioni fatte qui (18/09/2026).** Il file arrivato da DeepSeek (la pagina
  con l'OCR) è stato corretto in **sette punti**, tutti trovati da prove vere nel
  browser:
  1. **`file://`**: uno script in testa legge `location.protocol` e, se non è
     `http(s)`, mostra l'avviso `#fileProtocolWarning` (stile `.alert` aggiunto al
     CSS) con le istruzioni per il mini-server;
  2. **ONNX Runtime e PaddleOCR creati solo su http(s)**: da `file://` non si
     scaricano nemmeno → niente `fetch` bloccato dal CORS e niente errore criptico;
  3. **Tesseract.js sempre caricato** e `initOcr()` lo prova per primo; su http(s)
     aspetta `paddleocr-ready` (max 30 s) e usa PaddleOCR, altrimenti resta su
     Tesseract dicendolo nello stato. Il riconoscimento normalizza l'output dei due
     motori in `{text, box}`;
  4. **Transformers.js importato come modulo**: il tag `<script>` classico non
     partiva («Cannot use 'import.meta' outside a module») ed è il motivo per cui le
     **formule** restavano a zero;
  5. **il modello delle formule si carica in background**, dopo l'OCR: prima il
     pulsante restava bloccato per minuti aspettando un modello da centinaia di MB
     (lo stato ora dice «⏳ modello formule in caricamento (è grosso: il testo si può
     già estrarre)»);
  6. **lista di modelli LaTeX-OCR** provati in ordine:
     `Youn-Sung/latex-finetuned-onnx` (`dtype: 'fp32'`; è l'unico con
     `encoder_model` + `decoder_model_merged` — provato: rende `f(x) = x² + 3x − 1`
     come `f(x)=x^{2}+3x-1` in 49 s), poi `nougat-latex-base-ONNX` e
     `latex_finetuned-ONNX`, che oggi chiedono file non pubblicati nei loro
     repository (404);
  7. **il ritaglio delle formule funziona con entrambi i motori**: nuova funzione
     `casellaItem` che accetta `box`/`bbox` con `xMin,yMin,xMax,yMax`, oppure
     `x,y,width,height`, oppure i 4 punti (`poly`/`points`), e `null` se non c'è
     nulla. Con PaddleOCR il ciclo non partiva **mai** (cercava solo `item.box`);
     per giunta su una foto da 12 MP poteva fare centinaia di inferenze (1-3 s
     l'una): ora c'è un **tetto di 12 formule**, un filtro che scarta i riquadri
     oltre 1/4 dell'immagine e lo stato mostra l'avanzamento.
  In più: la compilazione PDF con **Siglum era rotta** (`pdfEngine.ready is not a
  function`): l'API di `@siglum/engine@0.1.4` è `init()` e
  `compile(source, { engine })` — la sorgente è il **primo** argomento — con
  risultato `{success, pdf, log}` (ora si mostra anche `error` quando fallisce).
  Sistemato anche lo stato che scriveva «⚠️ OCR pronto» per una cosa che andava bene.
- **Verifiche del 18/09/2026 (dopo le correzioni).** I 5 blocchi `<script>` della
  pagina si compilano (JavaScriptCore, `new Function`) e `casellaItem` è provata su
  6 forme di casella (tutte normalizzate, `null` quando non c'è). In **Chrome
  headless**:
  - da `file://` → avviso **visibile** (`display: block`), **nessun** tag
    `onnxruntime` nel DOM, Tesseract.js presente;
  - da `http://localhost:8123` → avviso **nascosto** (`display:none`), tag
    `onnxruntime` **presente**, Tesseract.js presente.
  Prova end-to-end sulla pagina servita via HTTP, con la foto
  **`IMG_9206.jpg`** (4032×3024, un quaderno di matematica, 1,9 MB) caricata
  dall'input file:
  - modelli della sfocatura pronti in **4 s** (freddo) e **2 s** (cache calda);
    **OCR pronto in 69-73 s** da freddo (scarica Tesseract + PaddleOCR) e **9 s**
    con la cache del browser; **Siglum pronto in 5-75 s**; **modello delle formule
    in 45-51 s** in background, senza bloccare il testo;
  - **testo riconosciuto: 44 righe** (387 caratteri). È un quaderno **scritto a
    mano** e pieno di formule, quindi esce mangiato (`In questocaso`,
    `f&) ha um asintoto dliquo,`): da testo **stampato** — l'uso previsto di
    Tesseract/PaddleOCR — ci si deve aspettare molto di meglio;
  - **LaTeX generato** (821 caratteri: `article`, `babel` italiano, `amsmath`,
    `geometry`) e `.tex` scaricabile;
  - **sfocatura sulla stessa foto**: «Volti rilevati: 0 · area persona ≈ 11.0% ·
    espansione 18px» in 18 s (0 volti è atteso su persone a figura intera, come già
    documentato);
  - il **modello delle formule** è stato provato anche da solo, su una formula
    disegnata in un canvas: `f(x) = x² + 3x − 1` → **`f(x)=x^{2}+3x-1`**.
  **Resta non verificato dentro l'app** il pezzo finale delle formule (il ritaglio
  dei riquadri e il blocco `align*` nel LaTeX): le prove sull'immagine da 12 MP
  restano lente (OCR completo + inferenze + la compilazione Siglum che scarica i
  pacchetti CTAN oltrepassano i minuti). I singoli pezzi sono però provati:
  `casellaItem`, il modello che rende il LaTeX giusto, il tetto di 12 formule e
  l'ordine di caricamento.
  **Consiglio per le prove:** usare un profilo Chrome **persistente**
  (`--user-data-dir=…`): il primo caricamento scarica ~400 MB di modelli e senza
  cache ogni prova li riscarica da capo.

