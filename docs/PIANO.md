# Piano tecnico — Garmin Descent Mk3i ⇄ Fotocamera iPhone

> Obiettivo dichiarato: realizzare una app per Garmin Descent Mk3i capace di
> avviare la fotocamera su iOS e di intercettarne lo stream video, come fa
> l'app **Camera Remote** di Apple Watch.

Questo documento separa **ciò che è realisticamente fattibile** da **ciò che
non lo è**, spiega il *perché* e propone un'architettura concreta e implementabile.

---

## 1. Conclusione in breve (TL;DR)

| Funzione desiderata | Fattibile? | Note |
|---|---|---|
| Avviare l'app **Fotocamera di sistema** di iOS dall'orologio | ❌ No | Nessuna API pubblica permette a un'app terza di lanciare/controllare `Camera.app`. |
| Intercettare lo **stream della Fotocamera di sistema** sull'orologio | ❌ No | Lo stream del Camera Remote di Apple Watch usa un protocollo privato Apple, non esposto a terzi. |
| Scattare foto / avviare video **da remoto** tramite una **app companion iOS dedicata** | ✅ Sì | L'orologio invia un comando, l'app iOS cattura con AVFoundation. |
| Mostrare un **mirino "live"** sul quadrante Garmin | ⚠️ Solo molto degradato | La banda BLE via Connect IQ permette al massimo thumbnail JPEG piccole a ~0,5–2 fps, non video fluido. |
| Comandi remoti (shutter, timer, zoom, switch fotocamera, REC) + feedback di stato | ✅ Sì | È il caso d'uso solido e consigliato. |

**Riassunto:** replicare *esattamente* il Camera Remote di Apple Watch **con la
Fotocamera di sistema iOS è impossibile** per un dispositivo non-Apple. È invece
fattibile (e consigliato) un sistema a due componenti — **app Connect IQ sul
Descent + app companion iOS con fotocamera propria** — che offre telecomando di
scatto/registrazione e, opzionalmente, un mirino a bassissima frequenza.

---

## 2. Perché il Camera Remote di Apple Watch non è replicabile

Il Camera Remote di Apple Watch funziona perché Apple controlla l'intero stack:

1. **Lancio della Fotocamera di sistema.** L'orologio dice all'iPhone di aprire
   `Camera.app`. iOS **non** offre alcuna API pubblica per avviare o pilotare
   l'app Fotocamera di sistema da un processo terzo o da un accessorio
   Bluetooth. Il sandbox di iOS lo vieta.
2. **Streaming del preview.** L'anteprima che vedi sull'Apple Watch viene
   trasmessa dall'iPhone all'orologio tramite un **protocollo proprietario
   Apple** (sopra il link Watch↔iPhone), non disponibile a sviluppatori terzi.
3. **Anche le app terze che "fanno la stessa cosa" non usano la Fotocamera di
   sistema.** App come FiLMiC Pro mostrano un mirino sull'Apple Watch perché
   implementano una **propria** `AVCaptureSession` dentro la loro app +
   estensione watchOS, usando `WatchConnectivity` tra device Apple. Non
   intercettano lo stream di `Camera.app`.

Un orologio Garmin non è un device Apple e comunica con l'iPhone **solo** tramite
il **Connect IQ Mobile SDK** sopra BLE, mediato da Garmin Connect. Quindi:

- non può lanciare la Fotocamera di sistema;
- non può ricevere lo stream privato Apple;
- ha a disposizione un canale BLE a **banda molto bassa** (vedi §4),
  inadatto al video in tempo reale.

Per questi tre motivi l'obiettivo "1:1 con Apple Watch" non è raggiungibile.
Fonti: Garmin Connect IQ Mobile SDK; documentazione Apple "Use Camera Remote on
Apple Watch".

---

## 2-bis. Verifiche tecniche (aggiornamento 2026-06-13)

Tre punti chiave verificati su documentazione Garmin/Apple e forum sviluppatori.

### A) L'orologio può "avviare" l'app companion iOS? → **NO (limite forte di iOS)**
- Il metodo `openApplication` del Mobile SDK serve nella direzione
  **iOS → orologio** (l'app companion apre/verifica l'app Connect IQ sul
  dispositivo). **Non** esiste l'inverso.
- iOS *può* risvegliare in **background** un'app collegata via Bluetooth quando
  arrivano dati, **ma solo se l'app è già in esecuzione** (almeno in
  background): l'app sull'orologio **non avvia** l'app companion iOS se questa
  non è già attiva. Riferimento: discussioni ufficiali Connect IQ Forums.
- **Conseguenza UX decisiva:** non si può replicare il flusso Apple Watch
  "apro tutto dal polso a freddo". Il flusso reale è: **l'utente apre prima
  l'app companion iOS** (in foreground, fotocamera attiva) → **poi** usa
  l'orologio come telecomando. Al più, con la modalità background
  `bluetooth-central` + CoreBluetooth, l'app resta viva in background, ma una
  cattura video AVFoundation completa **non** è consentita da background/schermo
  bloccato.

### B) Connettività usata tra Garmin e iOS → **BLE, mediato da Garmin Connect**
- Il canale orologio ⇄ iPhone per le app companion è **Bluetooth Low Energy**,
  **relayato dall'app Garmin Connect Mobile (GCM)**, che deve essere installata
  e attiva. Il Companion SDK iOS si appoggia alla connessione BLE di GCM.
- Esiste anche `Toybox.BluetoothLowEnergy` (CIQ 3.1+) con cui l'orologio fa da
  **BLE central** verso periferiche BLE generiche — utile per sensori/hardware,
  **non** è il percorso standard per parlare con un'app iPhone (per quello si usa
  il Mobile SDK + GCM).
- **Niente Wi‑Fi** per questo scambio (il Wi‑Fi di alcuni Garmin serve solo alla
  sincronizzazione con i server Garmin). ANT/ANT+ è per i sensori, non per la
  messaggistica con l'app companion.

### C) Risoluzione reale del display Garmin Descent Mk3i → **AMOLED touch**
- **Mk3i 43 mm:** AMOLED **1,2"**, **390 × 390 px**, touchscreen.
- **Mk3i 51 mm:** AMOLED **1,4"**, **454 × 454 px**, touchscreen.
- Nota: il display è ampio e a buona risoluzione → **non è il collo di
  bottiglia**. Il limite del "mirino live" resta esclusivamente la **banda BLE**
  (§4), non lo schermo.

> **Impatto sul progetto:** confermata l'architettura a due app (§3), ma con un
> vincolo di flusso obbligato: **prima si apre l'app iOS, poi si comanda dal
> polso**. Il punto (A) va comunicato chiaramente nell'onboarding utente.

---

## 3. Architettura consigliata (fattibile)

Sistema a due componenti che comunicano via Connect IQ Mobile SDK:

```
┌─────────────────────────┐        BLE / Connect IQ        ┌──────────────────────────┐
│  Garmin Descent Mk3i     │   (via Garmin Connect)         │   iPhone — App companion   │
│  App Connect IQ (Monkey C)│ <───────────────────────────> │   (Swift, AVFoundation)    │
│                          │   messaggi comando/stato        │                            │
│  - UI telecomando        │                                 │  - AVCaptureSession         │
│  - invio comandi         │                                 │  - scatto/REC/zoom/switch   │
│  - (opz.) mostra thumb    │ <─── thumbnail JPEG (opz.) ──── │  - salvataggio in Foto      │
└─────────────────────────┘                                 └──────────────────────────┘
```

### 3.1 App Connect IQ sull'orologio (Monkey C)
- Tipo: **Watch App** (non widget/data field) per avere input completo.
- UI: pulsante shutter, toggle foto/video, timer (3s/10s), zoom +/- (Digital
  crown / tasti), switch fotocamera anteriore/posteriore, indicatore stato
  (connesso / in registrazione / batteria iPhone).
- Comunicazione: `Communications.transmit()` verso l'app companion; ricezione
  callback per ACK e stato. Niente accesso diretto al BLE raw: si passa sempre
  per il framework Connect IQ.
- Feedback: vibrazione/tono alla conferma di scatto.

### 3.2 App companion iOS (Swift)
- Integra il **Connect IQ Companion App SDK for iOS**
  (`github.com/garmin/connectiq-companion-app-sdk-ios`).
- Gestisce una **propria** `AVCaptureSession` (fotocamera dell'app, non quella
  di sistema): preview a schermo, scatto foto (`AVCapturePhotoOutput`),
  registrazione video (`AVCaptureMovieFileOutput`).
- Riceve i comandi dall'orologio e li esegue; salva i media in `PHPhotoLibrary`.
- Invia all'orologio messaggi di stato compatti (ACK, durata REC, ecc.).
- Permessi richiesti: `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`,
  `NSPhotoLibraryAddUsageDescription`.
- Limite importante: per ricevere comandi in tempo reale l'app companion deve
  essere **in foreground** (con sessione camera attiva). iOS non consente a
  un'app terza di avviare la cattura video da background o da schermo bloccato.

### 3.3 Protocollo applicativo (messaggi)
Messaggi piccoli, serializzati come dictionary Connect IQ:

- `→ {cmd:"shutter"}` / `{cmd:"rec_start"}` / `{cmd:"rec_stop"}`
- `→ {cmd:"set", zoom:2.0, lens:"back", timer:3}`
- `← {ack:"shutter", ok:true}` / `{state:"recording", t:12}` / `{batt:78}`

---

## 4. Vincolo critico: il mirino "live"

La domanda chiave è se si possa mostrare un'anteprima video sull'orologio.

- Il canale Connect IQ ha **banda effettiva molto bassa** (ordine di grandezza:
  pochi KB/s utili, messaggi piccoli, latenza alta). Non è progettato per video.
- Conseguenza realistica: si possono inviare **thumbnail JPEG fortemente
  compresse** (es. 80×80–120×120 px) a **~0,5–2 fps**. Utile come "vedo grosso
  modo cosa inquadro", **non** un mirino fluido.
- Il display **non** è il limite: è AMOLED touch 390×390 (43 mm) / 454×454
  (51 mm), più che sufficiente. Il collo di bottiglia è **solo la banda BLE**.

**Raccomandazione di prodotto:** progettare la v1 come **telecomando di scatto**
(senza mirino) — robusto e davvero utile. Trattare il mirino a basse fps come
funzione **sperimentale opzionale** in una fase successiva, dietro un flag, dopo
aver misurato il throughput reale sul Mk3i.

---

## 5. Approcci alternativi valutati

1. **App iOS standalone con telecomando (consigliato come MVP rapido).**
   Solo app iOS: cattura con AVFoundation + telecomando (es. autoscatto,
   volume-shutter, o un secondo device). Nessuna dipendenza Garmin. Velocissima
   da rilasciare; serve a validare la parte fotocamera prima di aggiungere
   l'integrazione Connect IQ.

2. **Connect IQ + companion iOS (architettura di §3) — soluzione completa.**
   È ciò che più si avvicina all'esperienza Apple Watch *restando nei limiti
   delle API*: telecomando completo e, opzionalmente, mirino degradato.

3. **Integrazione con app fotocamera esistenti di terze parti.**
   Alcune app (es. FiLMiC Pro) supportano telecomandi/Apple Watch ma **non**
   espongono API pubbliche a un orologio Garmin. Non percorribile in modo
   affidabile.

4. **Controllo di action cam invece dell'iPhone.**
   Connect IQ ha già supporto consolidato per controllare action cam (es. linea
   VIRB) dall'orologio. Se l'obiettivo reale è "scattare da lontano", una
   action cam dà mirino e controllo nativi. Da considerare se il vincolo iPhone
   non è rigido.

---

## 6. Roadmap proposta

**Fase 0 — Validazione/Setup**
- Account Garmin Developer + Connect IQ SDK + simulatore.
- Account Apple Developer + progetto Xcode (Swift/SwiftUI).
- Verifica che il Descent Mk3i sia tra i device target supportati dalla app CIQ.

**Fase 1 — MVP telecomando (no mirino)**
- App iOS companion con AVCaptureSession (foto + video) e salvataggio in Foto.
- Integrazione Connect IQ Companion SDK; handshake orologio↔app.
- App CIQ: UI shutter + foto/video + ACK. Test su simulatore e device reale.

**Fase 2 — Controlli avanzati**
- Timer, zoom, switch lente, indicatori di stato (REC, batteria), feedback aptico.

**Fase 3 — Mirino sperimentale (opzionale)**
- Pipeline thumbnail JPEG a basse fps; misurare throughput reale sul Mk3i;
  esporre dietro flag se l'esperienza è accettabile.

**Fase 4 — Rilascio**
- Pubblicazione app iOS su App Store e app su Connect IQ Store; documentazione
  di pairing per l'utente.

---

## 7. Rischi e dipendenze

- **Foreground obbligatorio** dell'app iOS durante l'uso (limite iOS).
- **Latenza/banda BLE**: il mirino live resta il punto più incerto → trattarlo
  come opzionale e misurarlo presto.
- **Supporto device**: confermare API/UX disponibili sul profilo del Mk3i (è un
  computer subacqueo con UI e tasti specifici; in immersione il BLE col telefono
  non è disponibile — l'uso è quindi "in superficie").
- **Processi di review** doppi (Apple App Store + Connect IQ Store).

---

## 8. Riferimenti

- Connect IQ Mobile SDK for iOS — https://developer.garmin.com/connect-iq/core-topics/mobile-sdk-for-ios/
- Companion App SDK (iOS) — https://github.com/garmin/connectiq-companion-app-sdk-ios
- Esempio companion iOS — https://github.com/garmin/connectiq-companion-app-example-ios
- Communicating with Mobile Apps — https://developer.garmin.com/connect-iq/core-topics/communicating-with-mobile-apps/
- Apple — Use Camera Remote on Apple Watch — https://support.apple.com/guide/watch/camera-remote-apda6e61c287/watchos
