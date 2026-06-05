# AeroPlan — Drone Flight Planner & Simulator

Simulatore 3D, configuratore e pianificatore di volo drone, eseguibile interamente nel browser (nessun backend). Pianifica waypoint, POI/orbite, griglie fotogrammetriche e corridor mapping, simula il volo con telemetria live, ed esporta missioni nei formati dei droni reali.

## Funzioni
- **Vista 3D** (Three.js) con terreno procedurale o **mappa satellitare reale** (Esri World Imagery).
- **Configuratore**: preset DJI + parametri volo/camera; selettore modello per l'export.
- **Pianificazione**: Waypoint, **POI + Orbita**, **Griglia fotogrammetrica**, **Corridor mapping**.
- **Simulazione**: percorso + RTH, telemetria (velocità, quota, batteria, ETA), camere Orbit/Chase/FPV.
- **Export**: **DJI WPML `.kmz`** (conforme al modello), **DJI template-only `.kmz`**, **Litchi `.csv`**, **KML**, **GPX**, **JSON** nativo, **CSV** semplice.
- **Import**: `.json` (round-trip completo), `.kmz` / `.wpml` / `.kml` (ricostruisce i waypoint).

## File del repo
```
index.html        # l'app completa (single file)
sw.js             # service worker (cache + auto-update)
manifest.json     # PWA manifest
icon-192.png
icon-512.png
README.md
```

## Pubblicazione su GitHub Pages
1. Crea il repo e carica i file nella **root** (non in una sottocartella).
2. Repo → **Settings → Pages** → *Source: Deploy from a branch* → branch `main`, cartella `/ (root)`.
3. Dopo qualche minuto l'app è online su `https://<utente>.github.io/<repo>/`.
4. Apri quell'URL (HTTPS): il service worker si registra e l'app diventa **installabile** (PWA) e funziona **offline**.

> Il service worker richiede HTTPS (GitHub Pages lo fornisce) o `localhost`. Aperto come `file://` l'app funziona ma senza SW/offline.

## Aggiornamento automatico (sw.js)
Il service worker usa una cache **versionata** + strategia *stale-while-revalidate*:
- ad ogni nuovo deploy **incrementa `VERSION`** in `sw.js` (es. `aeroplan-v1.0.1`);
- al caricamento successivo il nuovo SW si installa, l'app mostra il banner **"Nuova versione disponibile"**: toccandolo si applica l'update e la pagina si ricarica con i file più recenti;
- le vecchie cache vengono eliminate in `activate`.

### Auto-deploy (opzionale, GitHub Actions)
Per bumpare la versione e pubblicare ad ogni push puoi aggiungere un workflow che riscrive la stringa `VERSION` in `sw.js` con l'hash del commit. In breve: un job che fa `sed` di `VERSION` con `${{ github.sha }}` prima del deploy Pages. (Chiedi pure e te lo genero.)

## Disclaimer
Strumento di **pianificazione e simulazione**. Il WPML usa i valori ufficiali `droneEnumValue`/`payloadEnumValue` DJI, ma è supportato **solo dai droni Enterprise**; verifica sempre la missione in DJI Pilot 2 (RTH, no-fly zone, quota relativa) prima di volare. La responsabilità del volo è del pilota.
