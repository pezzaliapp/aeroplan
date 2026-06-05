# AeroPlan — Brief di revisione completa (per Claude Code)

> Documento di lavoro per una revisione **d'insieme** dell'app, con la logica di
> un pilota di drone e i fatti dal Manuale Utente D-Flight v15 (gennaio 2026).
> Obiettivo: rendere l'app **professionale, pratica e non confusiva**, mantenendo
> la sua logica e la sua identità. **NON riscrivere da zero.**

## 0. Come usare questo brief
1. Metti il PDF `D-Flight-Manuale_Utente-v15-ITA.pdf` nella cartella del repo: leggilo
   direttamente (sezioni rilevanti: 1.4 Navigazione mappe/legenda, 1.4.3 Filtri NFZ/NOTAM,
   2. Drone Operation Plan, 6. Download UAS Geo Zones).
2. Lavora in **3 fasi**: prima un `AUDIT.md` (solo analisi), poi conferma dall'utente,
   poi implementazione una modifica alla volta.
3. Verifica regola: la normativa NON si inventa. Dove serve, **segnala** cosa va
   verificato su fonte ufficiale (ENAC, AIP) invece di affermarlo.

## 1. Vincoli inderogabili
- L'app resta **un unico file `index.html`** (Three.js r128 + JSZip via CDN, nessun build).
- Nessuna dipendenza nuova oltre ai CDN già presenti.
- Non rompere **export** (DJI WPML/Litchi/KML/GPX/JSON) né **import** (json/kmz/wpml/kml).
- `fromGPS` / `toGPS` sono la **sorgente di verità condivisa** di tutti gli export: NON riproiettare.
- Tutto lo storage (IndexedDB/localStorage) resta in `try/catch` con degrado morbido
  (deve funzionare anche se il browser blocca lo storage).
- A **ogni** modifica: incrementa `VERSION` in `sw.js` (ora `aeroplan-v1.2.2`).
- Verifica a ogni step: pagina servita (HTTP 200) e parse OK del JS inline.

## 2. Logica da pilota di drone (requisiti funzionali)
L'app deve far capire a un pilota, in pochi secondi e senza ambiguità:
- **Posso decollare qui?** Stato del punto di decollo (Home): libero / serve autorizzazione / vietato.
- **Fino a che quota?** La quota massima AGL consentita nel punto (vedi §3.1).
- **I miei waypoint sono in regola?** Quali cadono in zone con restrizioni e di che tipo.
- **Dov'è il drone e dove sto guardando?** Orientamento e posizione sempre chiari; mai "perdersi".
Gerarchia dell'informazione: prima **il verdetto** (posso/non posso/quota), poi i dettagli.
Ridurre il rumore visivo: l'utente non deve interpretare, deve leggere.

## 3. Fatti dal Manuale D-Flight v15 (da rispettare)
### 3.1 I colori della mappa = QUOTA MASSIMA AGL (non il permesso)
La legenda ufficiale d-flight è una scala di altitudine:
- rosso = max 0 m (di fatto interdetto)
- arancione = max 25 m
- giallo = max 45 m
- azzurro = max 60 m
- bianco/trasparente = max 120 m
- rosso tratteggiato = "area pericolosa"

In v1.2.2 le zone sono già colorate per `upperLimit` (funzione `altColor`) e il permesso
(VIETATO/AUTORIZZAZIONE/CONDIZIONATA) è testo. **Verifica** coerenza e casi limite
(upper mancante, valori non standard, area pericolosa). Caveat: D-Flight potrebbe derivare
la "quota max mostrata" in modo non identico al raw `upperLimit`; segnala se trovi divergenze.

### 3.2 Zone statiche vs dinamiche (validità temporale)
Il file ED-269 contiene zone **statiche** (AIP, avio/eli/idrosuperfici) **e dinamiche**
(NOTAM, No Fly Zone) con **validità temporale**. Oggi AeroPlan le tratta uguali e ignora
le date. Proponi: distinguere visivamente NFZ/NOTAM dalle statiche e mostrare/filtrare per
periodo di validità (campi `applicability`/validità nel data model ED-269).

### 3.3 Accesso al file ED-269
Il download "UAS GeoZone" (JSON ED-269) è riservato a **Operatori BASE/PRO con abbonamento
attivo** (non genericamente "registrati"). Esistono API M2M solo su richiesta a d-flight.
**Correggi** i testi in README e nel disclaimer di conseguenza.

### 3.4 Riferimenti ufficiali
Il pannello d-flight rimanda a **ENAC (www.enac.gov.it)** e **AIP Italia ENR 5.1**.
Allinea il disclaimer dell'app a queste fonti.

## 4. Problemi noti dai test reali (da risolvere nell'audit)
1. **Vista "Dettaglio" sfocata.** Il mosaico è pre-renderizzato a risoluzione fissa e poi
   ingrandito dalla camera → pixel sgranati da vicino. Serve tile a zoom più alto (z18–19)
   per il primo piano oppure, meglio, **LOD dinamico** (ricaricare i tile quando ci si
   avvicina). È la causa radice, non un dettaglio estetico.
2. **Cambio Home non ricentra la vista.** Sia con "Mia posizione" (GPS) sia con "Cerca città",
   la Home cambia ma camera e mappa **restano dov'erano**: il drone non si trova e la mappa non
   segue la nuova posizione. Il cambio Home deve sempre ricaricare la base attiva sulla nuova
   posizione e ricentrare la camera sul drone (framing), rendendolo visibile e di dimensione adeguata.
3. **Drone troppo piccolo** nelle viste larghe; manca un indicatore di posizione sempre leggibile.
4. **Zone su satellite obliquo**: verificare che restino appoggiate alla mappa (parallasse già
   ridotta in v1.2.0, ma da confermare a vista).
5. **Coerenza colori** (avviata in v1.2.2): completare e verificare la legenda.
6. **Rumore UI**: rivedere la gerarchia (verdetto prima, dettagli dopo); il pannello zone è denso.
7. **Disclaimer**: un click di accettazione non è tutela legale completa; valutare Termini/Privacy
   veri, soprattutto per la geolocalizzazione.

## 5. Funzioni esistenti da NON rompere (mappa rapida del codice)
- Scena 3D (Three.js): camere `orbit`/`chase`/`fpv`; `frameHome()`, `frameClose()`, `focusZone()`.
- Basi mappa: `loadBasemap(kind, extentOverride, frameCloseOn)`, `removeSatellite()`, `BASEMAPS`.
- Geo: `fromGPS`/`toGPS` (origine = Home). **Non toccare la proiezione.**
- Zone ED-269: `parseED269`, `selectNearby` (raggio `ZONE_MAXDIST`), `buildZones`,
  `checkZones` (controlla Home + waypoint), `restrInfo` (permesso/testo), `altColor` (colore/quota),
  `renderZoneList`, persistenza IndexedDB (`idbPut/idbGet/idbDel`, `tryRestoreZones`), `zoneMeta`+AIRAC.
- Pianificazione: waypoint, POI+orbita, griglia fotogrammetrica, corridor mapping.
- Export/Import: tutti i formati elencati al §1.
- PWA: `sw.js` versionato + banner aggiornamento.

## 6. Fasi della revisione
**FASE 1 — `AUDIT.md` (nessuna modifica al codice):** inventario funzioni/flussi; punti che
confondono un pilota (priorità decollo/zone/quota); coerenza dati verificabili (enum DJI WPML,
struttura ED-269, colori-quota §3.1, validità §3.2); bug/edge-case (§4, + mobile, perdita GPS,
ED-269 vecchio, zone fuori area); lista priorità (sicurezza/chiarezza prima, estetica dopo).

**FASE 2 — STOP.** Mostra `AUDIT.md` all'utente. Nessuna modifica finché non approva.

**FASE 3 — Implementazione:** solo dopo OK, una modifica alla volta, verifica HTTP 200 + parse,
`VERSION` bump a ogni step. Niente normativa inventata: dove serve, lascia un avviso nell'app.

---
*AeroPlan © Alessandro Pezzali — pezzaliAPP. Brief di revisione, giugno 2026.*
