# AUDIT.md — AeroPlan v1.2.2 · Fact-checking completo

> Fase 1 della revisione. **Nessuna modifica al codice in questo documento.**
> Analisi di `index.html` (1372 righe, JS inline) confrontata con il *Manuale Utente
> D-Flight v15 (gen 2026)* e con il brief `REVISIONE-AeroPlan.md`.
> Convenzione: i riferimenti `LNNN` indicano la riga in `index.html`.
> Regola rispettata: dove la normativa non è verificabile dal manuale, è **segnalata**
> come "da verificare su fonte ufficiale", non affermata.

## Metodo e fonti
- **Codice**: letto integralmente (HTML + CSS + JS inline).
- **Manuale**: estratto con `pdftotext -layout`. Sezioni usate: 1.4.1 (legenda/quota),
  1.4.3 + 1.4.4 (filtri NFZ/NOTAM **temporali**), 6 (Download UAS Geo Zones).
- **Fatti chiave confermati dal manuale**:
  - Sez. 6: ED-269 scaricabile **solo da Operatori BASE/PRO con abbonamento attivo**;
    il file contiene zone **statiche (AIP, avio-eli-idrosuperfici) e dinamiche (NOTAM/NFZ)**
    con **validità temporale, regole applicabili e riferimenti**; API M2M solo su richiesta.
  - Sez. 1.4.3/1.4.4: NFZ e NOTAM si filtrano **per tempo di attivazione** (date inizio/fine).
  - Sez. 1.4.1: la legenda rappresenta la **"quota massima"** con colori; la ricerca accetta
    **nome città** *oppure* **coordinate WGS84** separate da spazio (`es. 46.1100 11.3083`).

---

## Priorità in sintesi (cosa risolvere prima)
| # | Bug | Categoria | Gravità | Fix |
|---|-----|-----------|---------|-----|
| B1 | Cambio Home non ricentra camera/mappa | Logica/UX | 🔴 Alta | Unificare `setHome()` |
| B8 | Quota waypoint vs `upper` zona **non controllata** | Logica/sicurezza | 🔴 Alta | Estendere `checkZones` |
| B9 | Verdetto sepolto, nessun "posso decollare?" prominente | UX/logica | 🔴 Alta | Banner verdetto |
| B6 | Validità temporale NOTAM/NFZ ignorata | Coerenza manuale | 🔴 Alta | Parse `applicability` |
| B5 | Colore quota: VIETATO non forzato a rosso, unità/datum | Geo/coerenza | 🟠 Media | Fix `altColor`/`parseED269` |
| B2 | Dettaglio sfocato (mosaico a risoluzione fissa) | Geo/render | 🟠 Media | LOD su zoom + z19 |
| B7 | Testi ED-269 (BASE/PRO) + ENAC/AIP | Coerenza manuale | 🟠 Media | Correggere copy |
| B10 | `checkZones` prende la 1ª zona, non la più restrittiva | Logica | 🟠 Media | Ordinare per severità |
| B3 | Drone troppo piccolo nelle viste larghe | UX/render | 🟡 Bassa | Scala drone/marker |
| B4 | Zone "staccate" su satellite obliquo | Geo/render | 🟡 Bassa | Abbassare y + polygonOffset |
| B11 | Ricerca città: solo 13 città hardcoded, no lat-lon | Funzionale | 🟡 Bassa | Parse "lat lon" + unificare |
| B12 | Poligoni con buchi (donut) non gestiti | Geo/edge | 🟡 Bassa | Sottrarre anelli interni |
| B13 | `selectNearby` cap 800 senza ordinamento per distanza | Edge | 🟡 Bassa | Ordinare prima del cap |

---

# A. I 7 bug noti — verifica ed esito

## B1 — Cambio Home non ricentra camera e mappa  🔴
**Dove.**
- GPS "Mia posizione": `L753-759`. In caso di successo aggiorna `home`, chiama
  `relocalizeZones()` e `if(currentBase!=='proc')loadBasemap(currentBase)`.
- "Cerca città": `L599`. Aggiorna `home`, chiama **solo** `relocalizeZones()`.
- Input Lat/Lon manuali: `L596` (`input` → aggiorna `home`), `L597` (`change` → `relocalizeZones`).

**Perché è sbagliato.**
- **Cerca città**: non chiama mai `loadBasemap` né riframe → su Satellite/Mappa i tile
  restano sulla posizione vecchia, le zone si rifiltrano ma la base no, la camera non si
  muove. È il caso più rotto.
- **Lat/Lon manuali**: stesso problema — nessun reload base, nessun riframe.
- **GPS su "Terreno" (proc)**: non chiama `loadBasemap` (corretto, è procedurale) ma
  **non chiama `frameHome`**, quindi se la camera era ruotata/zoomata altrove il drone
  resta fuori inquadratura.
- Conseguenza pilota: dopo aver cambiato Home "il drone non si trova e la mappa non segue".

**Correzione proposta.** Una sola funzione sorgente di verità:
```
function setHome(lat,lon,{reframe=true}={}){
  home.lat=+(+lat).toFixed(6); home.lon=+(+lon).toFixed(6);
  $('homeLat').value=home.lat; $('homeLon').value=home.lon;
  drone.position.set(0,3,0);            // drone = origine = Home
  relocalizeZones();                    // rifiltra/riproietta zone
  if(currentBase!=='proc') loadBasemap(currentBase); // ricarica tile reali sulla nuova Home
  else if(reframe) frameHome(WORLD.size);            // su proc: ricentra comunque
}
```
Far passare GPS, Cerca città e (su `change`) gli input Lat/Lon **tutti** per `setHome`.
`loadBasemap` chiama già `frameHome`/`frameClose` alla fine, quindi su Satellite il riframe
arriva da lì. Vincoli rispettati: `fromGPS/toGPS` invariati (l'origine è Home, quindi i
waypoint locali restano coerenti).

---

## B2 — Vista "Dettaglio" sfocata (mosaico a risoluzione fissa)  🟠
**Dove.** `loadBasemap` `L780-828`; pulsante Dettaglio `L1100`
(`loadBasemap(currentBase,1200,true)`).

**Perché è sbagliato.**
- Il mosaico è **un'unica `CanvasTexture` 5×5 tile** (`k=2`, `span=5`, `L784`) calcolata a
  uno zoom `z` adattivo, poi **stirata** su `worldSize` (`L809-811`). Avvicinandosi oltre la
  risoluzione dei tile → pixel sgranati. È la causa radice, non estetica.
- Mitigazione **parziale già presente**: "Dettaglio" ricarica con `extentOverride=1200` →
  `z` più alto. Ma: (a) `z` è **clampato a 18** (`L788`); (b) si attiva solo col pulsante,
  **non** quando l'utente zooma con la rotella; (c) resta un singolo mosaico statico finché
  non si ripreme.

**Correzione proposta (LOD dinamico).**
1. Alzare il cap zoom a **19** in `L788` (`z=clamp(z,9,19)`): un quadruplicamento della
   risoluzione per il primo piano.
2. **Ricarico su zoom**: nel listener `wheel` (`L546`) o nel `loop`, quando in `orbit` la
   `orbit.dist` scende sotto una soglia e la base è satellite/mappa, ricaricare la base con
   `extentOverride ≈ orbit.dist*2`, **debounced** (es. 400 ms) per non scommerciare richieste.
   Salvare l'ultimo `z` caricato e ricaricare solo se cambia.
3. (Opzionale) `tex.minFilter=THREE.LinearMipmapLinearFilter; tex.generateMipmaps=true` per
   addolcire le viste larghe.

Vincolo: nessuna dipendenza nuova; resta tutto dentro `loadBasemap`.

---

## B3 — Drone troppo piccolo nelle viste larghe  🟡
**Dove.** `buildDrone` `L470-492` (dimensioni fisse ~3–7 m). Su satellite `worldSize`
può essere decine di km e `frameHome` mette `orbit.dist=ws*0.62` (`L1004`) → il drone è un
puntino. Anche `padRing` (`L466`) è fisso.

**Perché è sbagliato.** Nessuno scaling con la distanza camera: a 20 km il modello da ~6 m
è sub-pixel. Manca un indicatore di posizione sempre leggibile (requisito brief §2: "dov'è
il drone?").

**Correzione proposta.** Nel `loop` (`L1109`), scalare il drone (e/o un marker dedicato) per
mantenere una dimensione apparente costante:
```
const apparent = clamp(orbit.dist*0.025, 1, 60);   // fattore empirico
drone.scale.setScalar(apparent);
```
In alternativa/aggiunta: un **ring/billboard** ancorato alla posizione del drone che scala
con `orbit.dist`, così resta visibile anche quando il modello è minuscolo. Non scalare in
FPV/chase (dove la camera è vicina). Bump puramente visivo.

---

## B4 — Zone "staccate" su satellite obliquo  🟡
**Dove.** `buildZones` `L981-998`. Fill a `y=0.28/0.32`, bordi a `y=0.34/0.46/0.5`; il piano
basemap è a `y=0.05` (`L812`).

**Perché può apparire staccato.** Il dislivello fill↔basemap è ~0.23–0.45 m: trascurabile a
scala km, **visibile** in vista ravvicinata/Dettaglio (worldSize piccolo) e ad angolo obliquo
→ parallasse. Su terreno procedurale (non piatto) le zone piatte a `y≈0.3` possono tagliare i
rilievi. Già ridotto in v1.2.0 (commento `L987-988`), ma da confermare a vista.

**Correzione proposta.**
- Avvicinare i fill al piano: `y≈0.06–0.10` e usare `material.polygonOffset=true;
  polygonOffsetFactor=-1` per evitare z-fighting invece di alzarli fisicamente.
- Tenere i bordi appena sopra i fill (`+0.02`). Mantenere `depthWrite:false, renderOrder=1`.
- Verifica a vista su Satellite a 45° dopo il fix. Gravità bassa: cosmetico.

---

## B5 — Colore = quota max AGL: coerenza con la legenda  🟠
**Dove.** `altColor(u)` `L852-858`; legenda HTML `L229-237`; assegnazione
`color:altColor(upper)` in `parseED269` `L964`; lettura limiti `L956-958`.

**Esito verifica.** Le **fasce colore sono corrette** e combaciano con la legenda dell'app e
con la scala ufficiale (0=rosso, 25=arancio, 45=giallo, 60=azzurro, 120=bianco). **Ma** ci
sono divergenze rispetto a come d-flight deriva la quota mostrata (caveat brief §3.1):

1. **VIETATO non forzato a rosso.** Una zona `PROHIBITED` con `upper=120` nel dato viene
   colorata **bianca** (max 120). d-flight mostra le interdette in **rosso (0 m)**. Il colore
   sulla mappa quindi non comunica il divieto (che resta solo testo nella lista). →
   In `parseED269`/`altColor`: se la restriction è proibitiva, forzare `color=0xff5a6a`.
2. **Unità ignorate.** `parseED269` legge `uomDimensions/uom` (`L958`) ma `altColor` assume
   **metri**. Se `uom='ft'` (frequente in dati AIP) le soglie sono sbagliate (60 ft = 18 m
   verrebbe trattato come 60). → Convertire `upper` in metri prima del bucket, o bucketizzare
   in funzione dell'unità.
3. **Datum verticale.** L'app etichetta tutto "AGL" (lista `L1032`), ma ED-269 ha
   `upperVerticalReference` (AGL/AMSL/WGS84). Se il dato è AMSL, mostrarlo come AGL è
   fuorviante. → Leggere e mostrare il datum; non rietichettare ciecamente "AGL".
4. **`upper==null` → 120 (bianco).** Molte zone AIP senza `upperLimit` diventano "max 120 m"
   bianche: ottimistico. → Trattare il caso ignoto come neutro/segnalato, non come 120.
5. **"Area pericolosa" (rosso tratteggiato).** Non rilevata da `restrInfo`/`altColor`. ED-269
   può marcarle (DANGER/HAZARD). → Aggiungere riconoscimento + stile tratteggiato.

**Correzione proposta.** Far sì che `parseED269` calcoli il colore con una funzione che tenga
conto di restriction (PROHIBITED→rosso), unità e datum; arricchire `restrInfo` con DANGER.

---

## B6 — Validità temporale di NOTAM/NFZ ignorata  🔴
**Dove.** `parseED269` `L936-966`: cattura `name, restriction, lower, upper, uom, geom`. **Non**
legge `applicability` (validità). `checkZones` `L1040-1067` considera ogni zona **sempre attiva**.

**Perché è sbagliato.** Il manuale (Sez. 6) dice che ED-269 contiene zone **dinamiche
(NOTAM/NFZ) con validità temporale**, e (Sez. 1.4.3/1.4.4) che d-flight le filtra **per tempi
di attivazione**. AeroPlan tratta statiche e dinamiche uguali e ignora le date → può segnalare
come attiva una NFZ **scaduta** o **non ancora attiva**, e viceversa nascondere il fatto che
una zona è dinamica.

**Correzione proposta.**
- In `parseED269`, leggere `props.applicability` (array di `{startDateTime,endDateTime}` o
  `permanent:true`) e conservare su `allZones[i].applicability` + un flag `dynamic` quando la
  restriction/categoria è NOTAM/NFZ.
- Aggiungere `isZoneActiveNow(z)` (usa `Date.now()`); in `checkZones`/`renderZoneList`:
  - zone **non attive ora** → grigie/tratteggiate e **escluse** dal conteggio conflitti (o
    incluse con etichetta "non attiva ora");
  - mostrare il periodo di validità nella riga zona (lista `L1028-1036`).
- Distinguere visivamente **dinamiche** (bordo tratteggiato) da **statiche** (bordo pieno) in
  `buildZones` `L991-993`.
- Niente normativa inventata: se `applicability` manca nel file, trattare come permanente e
  **segnalarlo** ("validità non indicata — verifica su d-flight.it").

---

## B7 — Testi ED-269 (BASE/PRO) + riferimenti ENAC/AIP  🟠
**Dove.**
- Hint lista zone `L243`: "Scarica da D-Flight (profilo → **Download UAS Geo Zone**)…" —
  **non** dice che serve un abbonamento.
- Disclaimer `L357`: "fonte ufficiale … **d-flight.it (ENAC/ENAV)**".
- README (`README.md`, file separato) descrive l'accesso ai dati.

**Perché è sbagliato.** Manuale Sez. 6: il download è **esclusivo** per "utenti Operatori UAS
di tipo **BASE e PRO** in possesso di un abbonamento **attivo**" (più API M2M su richiesta).
Dire genericamente "scarica dal profilo" è impreciso. Inoltre il brief §3.4 chiede di allineare
le fonti ufficiali a **ENAC (enac.gov.it)** e **AIP Italia ENR 5.1**.

**Correzione proposta (solo copy in `index.html`).**
- Hint `L243`: "Scaricabile da d-flight solo da **Operatori BASE/PRO con abbonamento attivo**
  (profilo → Download UAS GeoZone). Carica qui il `.json.gz`."
- Disclaimer `L357-358`: citare **d-flight.it** come piattaforma e, come fonti normative,
  **ENAC (www.enac.gov.it)** e **AIP Italia (ENR 5.1)**. ⚠️ La dicitura "ENR 5.1" è richiesta
  dal brief ma **non testualmente confermata** dal manuale: inserirla come riferimento
  ("verifica su AIP Italia, sez. ENR 5.1") senza affermarla come legge.
- (Fuori dal vincolo "un solo file" ma consigliato) allineare anche `README.md`.

---

# B. Bug aggiuntivi emersi dall'analisi

## B8 — La quota dei waypoint NON è confrontata con `upper` della zona  🔴
**Dove.** `checkZones` `L1040-1049`: per ogni waypoint testa solo la **contenenza
planimetrica** (`pointInPoly` su `w.x,w.z`) e la **restriction testuale**. Non usa mai `w.alt`
né `z.upper`.

**Perché è sbagliato.** È il cuore della domanda del pilota ("fino a che quota AGL?"). Un
waypoint a 100 m AGL dentro una zona con `upper=25` è una violazione di quota, ma oggi l'app
lo segnala solo se la restriction è proibitiva/autorizzativa — non per **sforamento quota**.
Una zona "azzurra max 60 m" attraversata a 90 m non genera alcun avviso.

**Correzione proposta.** In `checkZones`, per ogni zona contenente il waypoint, confrontare
`w.alt` con `z.upper` (convertito in metri AGL): se `w.alt > z.upper` → flag specifico
"⚠ quota oltre il limite di zona (max N m)". Stessa logica per la Home (`L1050-1053`):
mostrare la **quota massima consentita al decollo**. Questo abilita anche B9.

---

## B9 — Manca il verdetto prominente "posso decollare? fino a che quota?"  🔴
**Dove.** Il verdetto è in `checkZones` → scritto nel piccolo hint `#zoneConflict` (`L244`,
in fondo alla sezione Zone). HUD e telemetria mostrano dati di volo, non un verdetto.

**Perché è sbagliato.** Brief §2: "prima il **verdetto**, poi i dettagli". Oggi il pilota deve
leggere testo denso in fondo a un pannello laterale per sapere se può decollare. Nessun colpo
d'occhio.

**Correzione proposta.** Un **banner verdetto** in alto nel `#view` (o sopra la lista zone),
a 3 stati colore:
- 🟢 **DECOLLO LIBERO — max N m AGL** (Home fuori zone, quota massima = min upper delle zone
  che la contengono o limite drone);
- 🟠 **SERVE AUTORIZZAZIONE — max N m**;
- 🔴 **DECOLLO VIETATO**.
+ riga sintetica waypoint: "✓ tutti in regola" / "⚠ k oltre quota · j in zona VIETATA".
Riusa i conteggi già calcolati in `checkZones`. Nessuna logica di volo nuova, solo
ri-presentazione. Dipende da B8 per il "max N m".

---

## B10 — `checkZones` prende la prima zona, non la più restrittiva  🟠
**Dove.** Waypoint: `L1044-1047` (`break` al primo `hit`). Home: `L1052-1053` (`break` al primo).

**Perché è sbagliato.** Un punto può cadere in più zone sovrapposte (es. CTR + interdetta).
L'ordine di `dfZones` è quello di parsing, non per gravità: si può riportare la zona **meno**
restrittiva e mascherare una VIETATO.

**Correzione proposta.** Tra tutte le zone che contengono il punto, scegliere la più
restrittiva con un ranking (VIETATO > AUTORIZZAZIONE > CONDIZIONATA > altro) e, a parità, la
`upper` più bassa. Vale per waypoint e Home.

---

## B11 — "Cerca città": solo 13 città hardcoded, niente lat-lon  🟡
**Dove.** `CITIES` `L598`; handler `L599`.

**Perché è sbagliato.** Il manuale (Sez. 1.4.1) prevede ricerca per **nome città** *oppure*
**coordinate WGS84** (`46.1100 11.3083`). L'app accetta solo 13 nomi e per il resto risponde
"Città non in elenco"; non interpreta coordinate digitate. Limita molto la scelta della Home
(e si lega a B1: anche le città note non riframano la vista).

**Correzione proposta.** Nel handler `L599`: se l'input combacia con `/^-?\d+(\.\d+)?[ ,]+-?\d+(\.\d+)?$/`,
trattarlo come `lat lon` e chiamare `setHome(lat,lon)`; altrimenti cercare in `CITIES`.
(Geocoding online è fuori scope: nessuna dipendenza nuova.) Risolve anche la coerenza col
flusso d-flight.

---

## B12 — Poligoni con buchi (donut) non gestiti  🟡
**Dove.** `asPolys` `L920-935`: per `polygon`/`multipolygon` prende solo `coordinates[0]`
(anello esterno), scartando gli anelli interni (`pointInPoly` `L1038` testa un singolo anello).

**Perché è sbagliato.** Zone ad anello (es. esclusione attorno a un'elisuperficie) verrebbero
trattate come piene: un punto **dentro il buco** (in realtà libero) risulterebbe in zona. Edge
case ma con impatto sulla sicurezza percepita.

**Correzione proposta.** Conservare gli anelli interni e, in `pointInPoly`/`checkZones`,
considerare un punto "dentro" la zona solo se è nell'esterno **e non** in un buco. Bassa
priorità (dipende dalla presenza reale di donut nel dataset d-flight).

---

## B13 — `selectNearby`: cap 800 senza ordinamento per distanza  🟡
**Dove.** `selectNearby` `L967-980`: itera `allZones` in ordine di parsing e fa `break` quando
`dfZones.length>=ZONE_CAP` (800).

**Perché è sbagliato.** Se il dataset è grande e supera il cap, le zone selezionate sono le
**prime nel file** entro raggio, non le **più vicine**: una zona molto vicina potrebbe essere
scartata perché appare tardi nell'array.

**Correzione proposta.** Calcolare la distanza per tutte le zone entro raggio, ordinare per
distanza crescente, poi troncare a `ZONE_CAP`. Basso impatto con dataset normali.

---

# C. Coerenza export/import e dati (verificati — OK, da non rompere)
- **WPML**: enum `droneEnumValue/payloadEnumValue` (`L406-416`) usati correttamente per il
  modello selezionato; `followBadArc`, `toPointAndStop…` sono valori DJI validi. **OK.**
- **fromGPS/toGPS** (`L575-576`): unica sorgente per tutti gli export. **Non toccare** (vincolo).
- **Import** JSON/KMZ/WPML/KML (`L1152-1206`): robusto, con fallback di entry. **OK.**
- **Storage**: IndexedDB e `localStorage` già in `try/catch` con degrado morbido
  (`L901-907`, `L1344-1346`). **Conforme al vincolo.** *Nota minore:* `idbPut` di B6 dovrà
  restare nello stesso pattern.
- **Nessuna dipendenza nuova** introdotta da nessuna delle fix proposte (tutto Three r128 +
  JSZip già presenti).

---

# D. Piano di implementazione proposto (Fase 3, una modifica per volta)
Ordine consigliato (sicurezza/chiarezza prima, estetica dopo). **Ogni step**: bump `VERSION`
in `sw.js`, verifica pagina HTTP 200 + parse JS OK.

1. **B1** `setHome()` unificato (GPS / città / lat-lon / input).
2. **B8** controllo quota waypoint/Home vs `upper`.
3. **B9** banner verdetto (dipende da B8).
4. **B6** validità temporale + distinzione dinamiche/statiche.
5. **B5** `altColor` (VIETATO→rosso, unità, datum, DANGER).
6. **B10** zona più restrittiva.
7. **B7** copy ED-269 BASE/PRO + ENAC/AIP.
8. **B2** LOD dinamico tile (z19 + ricarico su zoom).
9. **B11** lat-lon in "Cerca città".
10. **B3** scala drone/marker · **B4** zone glued · **B12**/**B13** edge case.

---

*Fine Fase 1. In attesa di OK prima di toccare il codice (Fase 3).*
*AeroPlan © Alessandro Pezzali — pezzaliAPP.*
