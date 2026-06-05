# AeroPlan — Drone Flight Planner & Simulator
**by pezzaliAPP — © Alessandro Pezzali (PezzaliApp)**

Simulatore 3D, configuratore e pianificatore di volo drone, eseguibile interamente nel browser (nessun backend). Pianifica waypoint, POI/orbite, griglie fotogrammetriche e corridor mapping, simula il volo con telemetria live, ed esporta missioni nei formati dei droni reali. All'avvio mostra un avviso di responsabilità che va accettato prima dell'uso.

## Funzioni
- **Avviso all'avvio**: finestra di disclaimer (con opzione "non mostrare più") che ricorda i limiti d'uso e la responsabilità del pilota.
- **Vista 3D** (Three.js) con tre basi selezionabili: **Terreno** procedurale, **Satellite** (Esri World Imagery) o **Mappa** stradale (Esri/OpenStreetMap, stile D-Flight).
- **Geolocalizzazione** — *📍 Mia posizione*: centra la Home sul GPS del dispositivo (richiede HTTPS + permesso); lascia emergere i prompt nativi del browser.
- **Configuratore**: preset DJI + parametri volo/camera; selettore modello per l'export WPML.
- **Pianificazione**: Waypoint, **POI + Orbita**, **Griglia fotogrammetrica**, **Corridor mapping**.
- **Zone UAS · D-Flight (ED-269)**:
  - importa il file ufficiale (accetta direttamente il `.json.gz`, o `.json`/`.zip`);
  - mostra le zone sulla mappa con **etichette in chiaro** del permesso (🔴 Vietato · 🟠 Autorizzazione · 🟣 Condizionata) e i limiti di quota;
  - **raggio regolabile** (10/20/40/80 km) per vedere anche le zone adiacenti;
  - **controllo dei waypoint**: segnala quelli dentro zone con restrizioni, distinguendo vietato/autorizzazione;
  - **memoria offline** (IndexedDB, opzionale) con indicazione della **data del dato** e avviso se più vecchio di un ciclo AIRAC (~28 giorni).
- **Simulazione**: percorso + RTH, telemetria (velocità, quota, batteria, ETA), camere **Orbit/Chase/FPV**.
- **Export**: **DJI WPML `.kmz`** (conforme al modello), **DJI template-only `.kmz`**, **Litchi `.csv`**, **KML**, **GPX**, **JSON** nativo, **CSV** semplice.
- **Import**: `.json` (round-trip completo), `.kmz` / `.wpml` / `.kml` (ricostruisce i waypoint).
- **PWA**: installabile e funzionante offline tramite service worker con auto-update.

## File del repo
```
index.html        # l'app completa (single file)
sw.js             # service worker (cache + auto-update)
manifest.json     # PWA manifest
icon-192.png
icon-512.png
LICENSE
README.md
```

## Pubblicazione su GitHub Pages
1. Crea il repo e carica i file nella **root** (non in una sottocartella).
2. Repo → **Settings → Pages** → *Source: Deploy from a branch* → branch `main`, cartella `/ (root)`.
3. Dopo qualche minuto l'app è online su `https://<utente>.github.io/<repo>/`.
4. Apri quell'URL (HTTPS): il service worker si registra e l'app diventa **installabile** (PWA) e funziona **offline**.

> Il service worker richiede HTTPS (GitHub Pages lo fornisce) o `localhost`. Aperto come `file://` l'app funziona ma senza SW/offline. Anche la **geolocalizzazione** richiede HTTPS.

## Aggiornamento automatico (sw.js)
Cache **versionata** + strategia *stale-while-revalidate*:
- ad ogni nuovo deploy **incrementa `VERSION`** in `sw.js` (es. `aeroplan-v1.1.5`);
- al caricamento successivo il nuovo SW si installa e l'app mostra il banner **"Nuova versione disponibile"**: toccandolo si applica l'update e la pagina si ricarica;
- le vecchie cache vengono eliminate in `activate`.

## Zone UAS (D-Flight / ED-269)
D-Flight non offre un'API pubblica gratuita, ma pubblica le zone geografiche UAS scaricabili gratis nel formato standard **EUROCAE ED-269** (JSON), come previsto dal Reg. UE 2019/947.
1. Accedi a **d-flight.it** → profilo → **Download UAS Geo Zone** (ottieni un `.json.gz`).
2. In AeroPlan: sezione *Zone UAS · D-Flight* → **🛡 Carica ED-269** (accetta direttamente il `.json.gz`, oppure `.json`/`.zip`).

Con "Ricorda offline" il file resta salvato nel browser (IndexedDB) per le sessioni successive; l'app mostra sempre la **data del dato** e avvisa se è scaduto il ciclo AIRAC. **Dato a solo scopo informativo**: la fonte ufficiale resta d-flight.it.

## Disclaimer
Strumento di **pianificazione e simulazione**. Il WPML usa i valori ufficiali `droneEnumValue`/`payloadEnumValue` DJI, ma è supportato **solo dai droni Enterprise**; verifica sempre la missione in DJI Pilot 2 (RTH, no-fly zone, quota relativa) prima di volare. Le zone UAS possono non essere aggiornate. La responsabilità della pianificazione e del volo è esclusivamente del pilota/operatore, nel rispetto delle norme EASA/ENAC e della privacy.

## Licenza & crediti
**© Alessandro Pezzali — PezzaliApp. Tutti i diritti riservati.**
"AeroPlan" e il marchio **pezzaliAPP** sono di proprietà di Alessandro Pezzali. Vedi il file [`LICENSE`](LICENSE) per i termini d'uso.

Crediti tecnici: [Three.js](https://threejs.org), [JSZip](https://stuk.github.io/jszip/), imagery © Esri / Maxar / Earthstar Geographics, mappa © Esri / HERE / Garmin / OpenStreetMap contributors, zone UAS © d-flight S.p.A. (Gruppo ENAV).

— *Made by pezzaliAPP · [pezzaliapp.com](https://pezzaliapp.com)*
