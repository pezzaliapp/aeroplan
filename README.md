# AeroPlan — Drone Flight Planner & Simulator
**by pezzaliAPP — © Alessandro Pezzali (PezzaliApp)**

Simulatore 3D, configuratore e pianificatore di volo drone, eseguibile interamente nel browser (nessun backend). Pianifica waypoint, POI/orbite, griglie fotogrammetriche e corridor mapping, simula il volo con telemetria live, ed esporta missioni nei formati dei droni reali.

## Funzioni
- **Vista 3D** (Three.js) con terreno procedurale o **mappa satellitare reale** (Esri World Imagery).
- **Geolocalizzazione** — pulsante *📍 Mia posizione*: centra la Home sul GPS del dispositivo (richiede HTTPS + permesso) e carica i tile satellitari attorno a te.
- **Configuratore**: preset DJI + parametri volo/camera; selettore modello per l'export WPML.
- **Pianificazione**: Waypoint, **POI + Orbita**, **Griglia fotogrammetrica**, **Corridor mapping**.
- **Zone UAS · D-Flight (ED-269)**: importa il file ufficiale delle zone geografiche UAS scaricato dal tuo profilo d-flight.it, le visualizza sulla mappa (vietate / autorizzazione / condizionate) e **controlla i waypoint**, segnalando quelli dentro aree con restrizioni.
- **Simulazione**: percorso + RTH, telemetria (velocità, quota, batteria, ETA), camere Orbit/Chase/FPV.
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
Il service worker usa una cache **versionata** + strategia *stale-while-revalidate*:
- ad ogni nuovo deploy **incrementa `VERSION`** in `sw.js` (es. `aeroplan-v1.0.4`);
- al caricamento successivo il nuovo SW si installa, l'app mostra il banner **"Nuova versione disponibile"**: toccandolo si applica l'update e la pagina si ricarica con i file più recenti;
- le vecchie cache vengono eliminate in `activate`.

### Auto-deploy (opzionale, GitHub Actions)
Per bumpare la versione e pubblicare ad ogni push puoi aggiungere un workflow che riscrive la stringa `VERSION` in `sw.js` con l'hash del commit. In breve: un job che fa `sed` di `VERSION` con `${{ github.sha }}` prima del deploy Pages. (Chiedi pure e te lo genero.)

## Zone UAS (D-Flight / ED-269)
D-Flight non offre un'API pubblica gratuita, ma pubblica le zone geografiche UAS scaricabili gratis nel formato standard **EUROCAE ED-269** (JSON), come previsto dal Reg. UE 2019/947.
1. Accedi a **d-flight.it** → profilo → **Download UAS Zone Geo** (ottieni uno zip da scompattare in JSON).
2. In AeroPlan: sezione *Zone UAS · D-Flight* → **🛡 Carica ED-269** (accetta `.json` o `.zip`).

Le zone entro ~15 km dalla Home vengono disegnate e i waypoint controllati. **Dato a solo scopo informativo**: la fonte ufficiale resta d-flight.it, da consultare sempre prima del volo.

## Disclaimer
Strumento di **pianificazione e simulazione**. Il WPML usa i valori ufficiali `droneEnumValue`/`payloadEnumValue` DJI, ma è supportato **solo dai droni Enterprise**; verifica sempre la missione in DJI Pilot 2 (RTH, no-fly zone, quota relativa) prima di volare. La responsabilità del volo è del pilota.

## Licenza & crediti
**© Alessandro Pezzali — PezzaliApp. Tutti i diritti riservati.**
"AeroPlan" e il marchio **pezzaliAPP** sono di proprietà di Alessandro Pezzali. Vedi il file [`LICENSE`](LICENSE) per i termini d'uso.

Crediti tecnici: [Three.js](https://threejs.org), [JSZip](https://stuk.github.io/jszip/), imagery © Esri / Maxar / Earthstar Geographics, zone UAS © d-flight S.p.A. (Gruppo ENAV).

— *Made by pezzaliAPP · [pezzaliapp.com](https://pezzaliapp.com)*
