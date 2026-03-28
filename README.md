# DroneRX V3.4 — Multi-Protocol Drone Detection & Identification

**Auteurs** : AH & EPERRET  
**Hardware** : M5Stack CoreS3 (ESP32-S3)  
**Licence** : MIT

Système embarqué de détection passive et d'émission de signaux d'identification drone sur la bande WiFi 2.4 GHz. Détecte 4 protocoles, émet en FR SGDSN et ODID ASTM F3411, avec interface cartographique temps réel, météo, ADS-B, et prédiction de trajectoire anticollision.

---

## Fonctionnalités

### Réception (RX) — Détection passive 4 protocoles

| Protocole | OUI | Standard |
|-----------|-----|----------|
| **FR SGDSN** | `6A:5C:35` | Réglementation française |
| **ODID** | `FA:0B:BC` + type `0x0D` | ASTM F3411-22a (Open Drone ID) |
| **DJI DroneID** | `26:37:12` | Propriétaire DJI |
| **Parrot** | `90:3A:E6` | Propriétaire Parrot |

Le sniffer WiFi en mode promiscuous scanne les beacons sur 14 canaux et extrait les Vendor Specific IEs par OUI. Chaque trame détectée est décodée, stockée et transmise à l'interface HTML via USB Serial ou WebSocket.

### Émission (TX) — Beacon FR + ODID

Émission alternée (1s FR, 1s ODID, 10 beacons/s) avec :

- **FR** : encodage TLV conforme à la norme SGDSN
- **ODID** : MessagePack 5 messages (BasicID, Location, SelfID, System, OperatorID) via **structs packed GCC** — pas d'encodage byte-par-byte

L'encodeur ODID minimal (< 100 lignes de structs + 7 helpers) est écrit depuis zéro, basé sur les structures de `opendroneid-core-c`. Le compilateur GCC ARM Little-Endian gère les bitfields automatiquement (convention : premier champ déclaré = bits de poids faible).

**Structure trame ODID WiFi** :
```
[24] 802.11 Mgmt header
[12] Beacon fields (timestamp + interval + capability)
[2]  SSID IE hidden (tag + len=0)
[3]  Supported Rates IE (6 Mbps = 0x8C)
[2]  Vendor IE tag + length
[4]  OUI FA:0B:BC + type 0x0D
[1]  message_counter  ← CRITIQUE, souvent oublié
[3]  Pack header (0xF2, SingleMessageSize=25, MsgPackSize=5)
[125] 5 × 25 bytes messages
= 177 bytes total
```

### Interface HTML (drone_page.h)

Page web autonome servie en PROGMEM par le board, accessible aussi en `file://` depuis un navigateur.

- **Carte Leaflet** — drones, homes, trails, labels altitude
- **Météo Open-Meteo** — conditions actuelles, prévision 6h, vent altitude (10/80/120/180m), GO/NO-GO drone
- **ADS-B OpenSky** — avions en temps réel dans un rayon de 55 km, icônes orientées colorées par altitude, flight level
- **Prédiction trajectoire 60s** — extrapolation par télémétrie native (speed/heading du drone), taux de virage, correction cos(lat), cône d'incertitude
- **TX Config** — modal avec selects (type drone, classe EU C0-C6, ID exploitant) envoyé au board via USB JSON
- **Son on/off** — contrôle le speaker du board depuis la page
- **Mobile responsive** — tabs sur petit écran (< 700px)
- **Simulation** — 2 drones FR en orbite pour tester sans matériel

---

## Architecture

```
drone_rx_v3.ino          Firmware ESP32 (Arduino IDE)
├── WiFi sniffer         Mode promiscuous, scan 14 canaux
├── Décodeur 4 OUI       scanAllVendorIEs()
├── TX FR + ODID          buildFRBeacon() / buildODIDBeacon()
├── ODID packed structs  6 structs __attribute__((__packed__))
├── USB Serial            Réception GPS/TX-CFG, émission JSON drone
├── WebSocket             Serveur ws://192.168.4.1:81
├── NVS                  Sauvegarde ID exploitant
├── Écran tactile         Keyboard ID, waterfall SDR, TX screen
└── Son                  beepNewDrone(), soundEnabled

drone_page.h             Page HTML en PROGMEM
├── Leaflet.js           Carte OSM, markers, trails
├── Décodeurs JS          decodeFR() / decodeODID() / decodeDJI() / decodePAR()
├── Trajectoire          updatePrediction() — heading-based, cos(lat)
├── Météo                Open-Meteo API (zéro CORS)
├── ADS-B                OpenSky Network API
├── TX Config             Modal → JSON USB → board
├── Son toggle           AudioContext beep + commande board
└── Mobile tabs           Responsive < 700px
```

---

## Matériel requis

| Composant | Détail |
|-----------|--------|
| M5Stack CoreS3 | ESP32-S3, 16MB Flash, OPI PSRAM, écran tactile IPS 2" |
| Câble USB-C | Connexion PC pour Serial + programmation |

Aucun module externe requis — le WiFi intégré suffit pour la détection et l'émission sur la bande 2.4 GHz.

---

## Installation

### 1. Arduino IDE

- Board : **ESP32S3 Dev Module** (ou M5Stack CoreS3)
- Flash : **16MB (QIO)**
- PSRAM : **OPI**
- USB CDC On Boot : **Enabled**
- Partition Scheme : **app3M_fat9M_16MB**

### 2. Bibliothèques

- `M5Unified` (>= 0.2.13)
- `M5GFX` (>= 0.2.19)
- `WiFi` (ESP32 built-in)

### 3. Compilation

Placer `drone_rx_v3.ino` et `drone_page.h` dans le même dossier, compiler et flasher.

---

## Mode d'emploi

### Démarrage

1. Le board démarre en **mode Sniff** — scanne les canaux WiFi et affiche le waterfall SDR
2. L'écran affiche un waterfall RSSI par canal + les drones détectés

### Connexion à l'interface HTML

**Option A — WiFi AP** :
1. Se connecter au réseau `DroneRX` (mot de passe : `dronesniff`)
2. Ouvrir `http://192.168.4.1` dans Chrome/Edge
3. Cliquer **WiFi WS** dans le panneau Connexion

**Option B — USB Serial (recommandé)** :
1. Sauvegarder la page HTML depuis le board (bouton **Sauver**)
2. Ouvrir le fichier `DroneRX_V3.html` en local dans Chrome/Edge
3. Connecter le board en USB-C
4. Cliquer **USB Serial**, sélectionner le port COM

### Carte et drones

- Les drones détectés apparaissent sur la carte avec une **icône colorée** par protocole
- Un **trail pointillé** suit la trajectoire passée
- Le **label altitude** s'affiche à côté de chaque drone
- Cliquer un drone dans la liste pour centrer la carte et voir ses données

### "Myself" — Votre drone

- Le premier drone détecté proche de votre position GPS (< 50m, vitesse < 1 m/s) est automatiquement marqué **[MY]** en jaune
- Vous pouvez aussi cliquer un drone dans la liste pour le désigner comme "myself"
- La prédiction de trajectoire des autres drones calcule la distance par rapport à "myself"

### Prédiction de trajectoire

- Chaque drone (sauf myself) affiche une **polyline pointillée** de 60 secondes
- La prédiction utilise la **vitesse et le cap natifs** du drone (pas dérivés du GPS)
- Le **taux de virage** est calculé depuis les headings successifs
- Un **cône d'incertitude** coloré apparaît entre deux prédictions successives
- Couleurs : bleu (> 2km) → vert (> 1km) → jaune (> 500m) → orange (> 200m) → rouge (< 100m)

### Météo

- Cliquer **Meteo** dans le panneau Connexion (ou l'onglet Meteo sur mobile)
- Données Open-Meteo : température, humidité, vent, QNH/QFE, visibilité, nuages
- **GO/NO-GO** automatique basé sur vent (> 10 m/s = NO-GO) et visibilité (< 1000m = NO-GO)
- Prévision 6h avec vent, rafales, direction, visibilité
- Vent en altitude : 10m, 80m, 120m, 180m

### ADS-B — Avions

- Cliquer **ADS-B** dans le header pour activer/désactiver
- Données OpenSky Network : avions en vol dans un rayon de ~55 km
- Icônes avion orientées selon le cap, colorées par altitude :
  - Rouge < 500m, orange < 2000m, vert < 5000m, cyan > 5000m
- Label flight level (FL) à côté de chaque avion
- Tooltip : callsign, altitude, vitesse, cap, taux de montée, distance
- Rafraîchissement toutes les 15 secondes

### Émission (TX)

1. Sur l'écran tactile du board : saisir l'**ID exploitant** (clavier Alpha Tango)
2. Ou depuis la page HTML : cliquer **TX Config**, remplir les champs, cliquer **Envoyer**
3. Le board émet en alternance FR + ODID sur le canal 6

### Configuration TX depuis HTML

Le modal TX Config permet de configurer :

| Champ | Valeurs |
|-------|---------|
| ID Exploitant | Format Alpha Tango (ex: FRAokesq0o9db0q6-cf8) |
| Type de drone | Avion, Multirotor, Gyravion, VTOL, Planeur, etc. |
| Type d'identification | Serial ANSI, Serial CAA, UTM UUID, Spécifique |
| Catégorie EU | Non défini, Open, Specific, Certified |
| Classe EU | C0 à C6 |
| Description | Texte libre (max 23 car.) |

La configuration est envoyée au board via USB Serial en JSON et prise en compte immédiatement.

### Son

- Bouton 🔈 dans le header : toggle on/off
- Contrôle le speaker du board (beep nouveau drone) ET le beep navigateur (alerte proximité)
- Beep simple (800 Hz) quand un drone est prédit à < 200m
- Double beep aigu (1200+1600 Hz) quand < 100m

### Simulation

- Cliquer **Simu** pour lancer 2 drones FR virtuels en orbite autour de Cayenne
- Permet de tester la carte, les prédictions, les cônes, sans matériel
- Cliquer **Stop** pour arrêter

---

## Revue de code

### Firmware (drone_rx_v3.ino — 2258 lignes)

**Points forts** :
- Architecture propre : sniffer → décodeur → queue → dispatch → WebSocket/USB
- Queue circulaire `volatile rx_packet_t` avec flag `ready` — communication ISR→loop sans mutex
- L'encodeur ODID utilise des `__attribute__((__packed__))` structs identiques au format fil — le compilateur fait le travail bit-level, éliminant 100% des erreurs d'offset manuelles
- Le `message_counter` byte (entre OUI+type et MessagePack) est correctement inclus — c'est le bug le plus fréquent des implémentations ODID DIY
- Beacon FR en TLV big-endian, beacon ODID en packed little-endian — les deux coexistent proprement

**Points d'attention** :
- `esp_wifi_set_promiscuous(true)` est requis pour `esp_wifi_80211_tx()` — ne jamais le retirer silencieusement
- Le country code WiFi doit être `JP` (ou `FR`) pour autoriser le canal 13 — l'ESP32 défaut à `US` qui le bloque
- Les clés NVS ne doivent pas changer entre versions sous peine de perdre les données stockées
- `serialInBuf[256]` est suffisant pour le JSON TX Config mais serré — ne pas ajouter de champs longs

### Page HTML (drone_page.h — 1140 lignes)

**Points forts** :
- Page 100% autonome en PROGMEM — fonctionne en `file://`, `http://`, et `https://`
- Météo Open-Meteo (CORS natif) — plus de proxies CORS fragiles
- Décodeur ODID en little-endian avec les bons offsets (SpeedVertical byte 4, Latitude bytes 5-8)
- Skip correct de 4 bytes (1 message_counter + 3 pack header) dans `decodeODID()`
- Prédiction trajectoire utilise la télémétrie native (speed/heading du drone) au lieu de dériver des positions GPS — robuste au bruit GPS
- Correction `cos(lat)` sur la conversion mètres→degrés longitude

**Points d'attention** :
- L'API OpenSky est limitée à ~10 requêtes/minute en anonyme — le timer de 15s respecte cette limite
- Les décodeurs DJI et Parrot sont partiels (reverse-engineered) — certains modèles peuvent avoir des formats différents
- Le `100dvh` CSS gère le viewport mobile nativement — ne pas réintroduire de `fixVH()` JavaScript

### Encodage ODID — Pièges résolus

| Bug | Impact | Résolution |
|-----|--------|------------|
| `message_counter` manquant | Décalage total du MessagePack | Ajouté entre OUI+type et pack |
| `SpeedVertical` oublié à byte 4 | Lat/Lon/Alt décalés d'1 octet | Struct packed inclut le champ |
| `EWDirection` au bit 2 | Directions > 180° corrompues | Bit 1 (pas bit 2) |
| Direction divisée par 2 | Cap à moitié | Stockage direct 0-179 |
| AreaRadius `uint16` au lieu de `uint8` | System message décalé | Corrigé dans le struct |
| ClassificationType = 0 | "Undeclared" au lieu de "EU" | Mis à 1 |
| Décodeur JS en big-endian | Toutes les valeurs ODID fausses | Fonctions `leI32`/`leU16` |

---

## Protocole USB Serial

### Board → HTML (JSON par ligne)

```json
{"p":"FR","h":"0102...","m":"AA:BB:CC:DD:EE:01","r":-55,"c":6}
```
- `p` : protocole (FR, ODID, DJI, PAR)
- `h` : payload hex
- `m` : adresse MAC
- `r` : RSSI (dBm)
- `c` : canal WiFi

### HTML → Board (JSON par ligne)

```json
{"g":[4.9371,-52.3258,30]}
```
Position GPS (lat, lon, alt) envoyée chaque seconde.

```json
{"snd":1}
```
Son on (1) / off (0).

```json
{"tx":{"id":"FRAokesq0o9db0q6-cf8","uaType":2,"idType":1,"euCat":1,"euClass":2,"selfDesc":"Recreational flight"}}
```
Configuration TX émetteur.

---

## Crédits

- **OpenStreetMap** — © OpenStreetMap contributors
- **Leaflet.js** — BSD-2
- **Open-Meteo** — API météo gratuite, CC BY 4.0
- **OpenSky Network** — ADS-B data, usage académique/personnel
- **opendroneid-core-c** — Référence structs ODID (Apache 2.0)
- **ESP-IDF / Arduino ESP32** — Espressif Systems

---

## Licence

MIT — Libre d'utilisation, modification et distribution.
