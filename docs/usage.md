# Guide d'utilisation

Pilotage du **Siglent SDS1204X-E** en SCPI sur LAN. Ce guide couvre
l'installation, la mise en service de la liaison réseau, puis chaque sous-commande
de la CLI. Une section **Dépannage** reprend les pièges vérifiés sur le matériel.

## 1. Installation

### NixOS

Tout l'environnement Python (PyVISA, `pyvisa-py`, NumPy, h5py, scipy, matplotlib,
pyqtgraph, PyQt5, pytest) est fourni par `flake.nix` / `shell.nix` :

```bash
nix develop          # avec flakes (flake.nix)
# ou
nix-shell            # sans flakes (shell.nix)
```

Le shell affiche `Env scope prêt : python -m scope.cli idn <ip> | pytest`.

### Windows

```bat
py -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Le backend `pyvisa-py` (Python pur) suffit : **aucune installation de NI-VISA**
n'est requise. NI-VISA reste utilisable s'il est déjà présent.

### Vérifier les tests

La suite complète (373 tests au 2026-10-06) tourne sans matériel (scope simulé,
y compris la vraie fenêtre GUI hors écran ; PyQt5 requis, donc le `.venv` d'`install.ps1`) :

```bash
nix-shell --run "pytest -q"     # NixOS
pytest -q                       # venv déjà activé
```

## 2. Connexion au scope

### a. Régler / relever l'IP du scope

Sur l'oscilloscope : **Utility → I/O → LAN**. Relever l'IP, ou la fixer si le
scope est en DHCP sans serveur.

### b. Côté PC : assurer une route IP

Le `ping` doit répondre :

```bash
ping <ip>
```

**Connexion Ethernet directe** (scope câblé en direct, pas de routeur/DHCP) :
l'interface peut ne pas avoir d'IPv4. Il faut lui en attribuer une dans le même
sous-réseau que le scope. Exemple réel (scope à `10.11.13.220`, interface
`enp0s31f6`) :

```bash
sudo ip addr add 10.11.13.10/24 dev enp0s31f6
ping 10.11.13.220
```

Cette adresse est **temporaire** (perdue au reboot). Pour la rendre persistante
sous NixOS, voir [`nixos-static-ip.md`](nixos-static-ip.md).

### c. Valider la chaîne réseau → VISA → SCPI

```bash
python -m scope.cli idn <ip>
# -> Siglent Technologies,SDS1204X-E,<n° série>,<firmware>
```

Si cette commande répond, toute la pile (TCP/IP → VXI-11 → PyVISA → SCPI) est
fonctionnelle.

## 3. Options globales

Valables avant la sous-commande. **Le transport par défaut diffère selon la
commande** — mesuré sur matériel, un fetch complet est ~30-50× plus lent en
VXI-11 qu'en socket (~10s contre ~0,2-1,6s, cf. `docs/architecture.md`) :

| Option | Effet | Défaut |
|--------|-------|--------|
| `--socket` | force le transport socket brut (`TCPIP::<ip>::5025::SOCKET`) pour `idn`/`scpi`/`set`/`screenshot` | off (ces commandes restent en VXI-11 par défaut) |
| `--vxi11` | force VXI-11 pour `capture`/`series`/`live`/`gui` (au lieu du socket, leur défaut) | off (ces commandes sont en socket par défaut) |
| `--timeout <ms>` | timeout VISA en millisecondes | `10000` |

```bash
python -m scope.cli --socket --timeout 20000 idn <ip>   # idn, forcer socket
python -m scope.cli --vxi11 gui <ip>                     # gui, forcer VXI-11
```

`capture`/`series`/`live`/`gui` sont en socket par défaut car elles fetchent
toujours la mémoire native complète (cf. §4) — pour `live`/`gui`, à chaque
frame d'une boucle ; pour `capture`, en une fois mais avec le même coût par
requête ; pour `series`, à chaque tick, où la latence VXI-11 ferait décrocher
la cadence demandée (audit du 2026-07-14, cf. § Dépannage « cadence non
tenue »). `idn`/`scpi`/`set`/`screenshot` ne font qu'un fetch ponctuel léger
(pas de waveform) : VXI-11 y reste le défaut (gère plus proprement les blocs
binaires par construction, flag END plutôt qu'un caractère de terminaison) —
et pour `screenshot` en particulier, c'est **obligatoire** : `screen_dump()`
lit avec `read_raw()` délimité par ce flag END, incompatible avec la
terminaison `\n` du mode socket sur un BMP qui contient des octets `0x0A`.

## 4. Sous-commandes

### `idn` — identité

Lit `*IDN?`. Sert de test de bout en bout de la liaison.

```bash
python -m scope.cli idn 10.11.13.220
```

### `scpi` — console SCPI interactive

REPL : une ligne **finissant par `?`** est traitée comme une *query* (réponse
affichée) ; sinon c'est une *commande* (write). `Ctrl-D` pour quitter.

```bash
python -m scope.cli scpi 10.11.13.220
```
```
scpi> *IDN?
Siglent Technologies,SDS1204X-E,...
scpi> C1:VDIV?
C1:VDIV 1.00E+00V
scpi> TDIV 1MS
scpi> SARA?
SARA 1.00E+09Sa/s
```

### `capture` — récupérer des waveforms

Lit une ou plusieurs voies. **Le CSV est un seul fichier multi-colonnes**
(`<horodatage>.csv` : `time_s, <nom1>_<unité1>, <nom2>_<unité2>, ...`, une
colonne par voie cochée) — pas un CSV par voie. `<nom>` reprend le **label**
configuré (`channels_config.json`, même fichier que la GUI, ex. « Vbat »),
sinon le nom de voie brut (`C1`). Les autres formats (**`.npy`** par défaut,
`npz`/`hdf5`/`mat` via `--format`) restent **un fichier par voie**
(`<horodatage>_<voie>.<ext>`). Avec `--plot`, génère aussi un PNG.

| Option | Défaut | Effet |
|--------|--------|-------|
| `--channels C1 [C2 …]` | `C1` | voies à capturer |
| `--outdir <dir>` | `captures` | dossier de sortie |
| `--format {csv,npy,npz,hdf5,mat} […]` | `csv npy` | formats d'export (combinables) |
| `--plot` | off | génère aussi un PNG (matplotlib, décimé) |
| `--no-show` | off | n'ouvre pas la fenêtre du plot (export seul, backend Agg) |
| `--single` | off | arme un trigger unique (`TRMD SINGLE`) et attend le déclenchement avant de capturer (cf. § capture déclenchée) |
| `--trigger-timeout <s>` | `10` | délai max d'attente du déclenchement, avec `--single` |

```bash
# une voie
python -m scope.cli capture 10.11.13.220 --channels C1

# deux voies + PNG sans ouvrir de fenêtre
python -m scope.cli capture 10.11.13.220 --channels C1 C2 --plot --no-show --outdir captures

# capture déclenchée : attend un vrai trigger avant de lire (timeout 5s)
python -m scope.cli capture 10.11.13.220 --single --channels C1 --trigger-timeout 5

# export scientifique (NPZ compressé, HDF5, MAT) en plus du CSV/.npy
python -m scope.cli capture 10.11.13.220 --format csv npy npz hdf5 mat
```

**Formats disponibles** (`waveform.VALID_FORMATS`) :

| Format | Extension | Fichier(s) | Dépendance | Contenu |
|--------|-----------|------------|------------|---------|
| `csv` | `.csv` | **un seul, combiné** (`<horodatage>.csv`) | aucune | `time_s` + une colonne `<nom>_<unité>` par voie |
| `npy` | `.npy` | un par voie (`<horodatage>_<voie>.npy`) | numpy | tableau `(n, 2)` : temps + tension |
| `npz` | `.npz` | un par voie | numpy | `time`, `volts`, `vdiv`, `offset`, `interval`, `channel` (compressé) |
| `hdf5` | `.h5` | un par voie | h5py | datasets `time`/`volts` + attributs (vdiv/offset/interval/channel) |
| `mat` | `.mat` | un par voie | scipy | équivalent HDF5, format MATLAB (`scipy.io.loadmat`) |

Sortie type (2 voies, C1 nommée « Vbat » dans `channels_config.json`) :
```
2 voie(s) -> captures/20260629_131500_C1.npy, captures/20260629_131500_C2.npy, captures/20260629_131500.csv
plot -> captures/20260629_131500.png
```
Le CSV combiné (`20260629_131500.csv`) a pour en-tête `time_s,Vbat_V,C2_V`.

Une voie sans acquisition (ex. sans signal) **n'interrompt pas** la capture des
autres voies : l'échec est signalé (stderr), et seules les voies réussies
apparaissent dans le CSV combiné et ont leur `.npy`.

Relire un `.npy` (colonnes `temps_s`, `volt`) :
```python
import numpy as np
data = np.load("captures/20260629_131500_C1.npy")
t, v = data[:, 0], data[:, 1]
```

### `series` — série d'expérience (time-lapse)

Enregistre une **série d'instantanés à intervalle régulier**, sous un ID
d'expérience, jusqu'à une durée max — pour une capture ponctuelle, voir
`capture` ci-dessus. **Une capture de série = un instantané décimé** (mêmes
points que l'affichage, cf. `--points`), **pas** la mémoire brute — ça garde
les fichiers légers pour un enregistrement long (voir § Décimation ci-dessous).

Fichiers rangés sous `{outdir}/{id}/`. Par capture (tick) : **un seul CSV
combiné** `{id}_{index:04d}.csv` (`time_s` + une colonne `<nom>_<unité>` par
voie, nom = label configuré sinon voie brute — même principe que `capture`
ci-dessus), plus un fichier par voie pour les formats binaires
(`{id}_{index:04d}_{voie}.npy`, etc.). Un sidecar `{id}_meta.json` (paramètres
de la série + horodatage réel de chaque capture) clôt la série.

| Option | Défaut | Effet |
|--------|--------|-------|
| `--id <experiment_id>` | — (**requis**) | préfixe des fichiers et du sous-dossier |
| `--rate <n>` | — (**requis**) | nombre de captures par `--per` |
| `--per {second,minute,hour}` | `minute` | unité de cadence pour `--rate` |
| `--duration <durée>` | — (**requis**) | durée max de la série : `30m`, `2h`, `90s`, ou un nombre nu (secondes) |
| `--channels C1 [C2 …]` | `C1` | voies capturées à chaque tick |
| `--outdir <dir>` | `captures` | dossier de sortie |
| `--format {csv,npy,npz,hdf5,mat} […]` | `csv npy` | formats d'export par capture |
| `--points <n>` | `4000` | points par capture (décimation côté client, comme l'affichage) |
| `--start {now,countdown,threshold}` | `now` | mode de démarrage (voir ci-dessous) |
| `--countdown <s>` | `0` | délai avant démarrage, avec `--start countdown` |
| `--threshold-channel <voie>` | `C1` | voie surveillée, avec `--start threshold` |
| `--threshold-level <val>` | `1V` | niveau de déclenchement, ex. `1V` |
| `--threshold-slope {POS,NEG,WINDOW}` | `POS` | front de déclenchement |
| `--threshold-timeout <s>` | `60` | délai max d'attente du seuil |

**Modes de démarrage** — un seul mécanisme, trois variantes :

- `now` : démarre immédiatement.
- `countdown` : attend `--countdown` secondes (le temps de préparer le montage), puis démarre.
- `threshold` : configure le **trigger natif** du scope (`TRSE`/`TRLV`/`TRSL`/`TRMD SINGLE`
  + `ARM`, mêmes commandes que `capture --single`) et attend le déclenchement. C'est
  aussi la façon de **« jeter le rodage »** : rien n'est capturé avant la mise sous
  tension détectée par ce seuil — pas de détection Python séparée, juste le trigger
  matériel. Si le seuil n'est jamais atteint dans `--threshold-timeout`, la série est
  annulée : **aucun fichier écrit, aucun meta**.

Au **début de l'enregistrement**, le scope est remis en acquisition continue : après un
départ `threshold`, le trigger SINGLE (arrêté par la détection) repasse en **`TRMD AUTO`**
puis `ARM` ; dans les autres modes, `ARM` seul (relance un scope resté sur Stop). Sans
cela, chaque capture relisait la même trace figée. AUTO plutôt que NORM : si le
générateur s'arrête, les captures suivantes sont « sans signal » au lieu de répéter la
dernière trace.

**Condition d'arrêt** : la première limite atteinte entre le nombre de tops calculés
(`--rate`/`--per`) et `--duration`. Chaque voie en échec (ex. acquisition vide) est
isolée — n'interrompt pas les autres voies ni la série.

```bash
# 10 captures/minute pendant 30 minutes, démarrage immédiat
python -m scope.cli series 10.11.13.220 --id essai42 --rate 10 --per minute --duration 30m

# attend 5s (montage), puis capture C1+C2 toutes les 6s pendant 10 minutes
python -m scope.cli series 10.11.13.220 --id essai43 --rate 10 --per minute --duration 10m \
    --channels C1 C2 --start countdown --countdown 5

# démarre à la mise sous tension (seuil 1V sur C1), 1 capture/s pendant 5 minutes
python -m scope.cli series 10.11.13.220 --id essai44 --rate 1 --per second --duration 5m \
    --start threshold --threshold-channel C1 --threshold-level 1V --threshold-timeout 30
```

Sortie type :
```
3 capture(s) -> captures/essai42/
méta -> captures/essai42/essai42_meta.json
```
Ou, si le seuil n'a jamais été atteint :
```
série annulée : condition de démarrage non atteinte (rien capturé)
```

**Décimation = définition d'une capture, pas une option accessoire.** Une trame
plein écran peut faire ~130-300 Mo (cf. § Mémoire profonde, `docs/architecture.md`) ;
répétée à chaque tick d'une série longue, ça remplit le disque en quelques minutes.
`--points` (défaut `4000`, comme « Points dessinés » dans la GUI) décime
**après** le fetch, avant sauvegarde — un CSV de série pèse alors quelques dizaines
de Ko au lieu de centaines de Mo. **Ça ne réduit pas le temps de transfert depuis le
scope** (toujours la mémoire complète lue) : le plancher de cadence réel reste le
temps d'un fetch (voir § Dépannage « cadence non tenue » plus bas).

Aussi disponible depuis la **GUI**, panneau « Série » — voir § `gui` ci-dessous.

### `measure` — mesures automatiques

Lit des mesures calculées par le scope (SCPI `PAVA?`) sur une ou plusieurs
voies : amplitude crête-à-crête, fréquence, RMS, rise/fall time, etc.

| Option | Défaut | Effet |
|--------|--------|-------|
| `--channels C1 [C2 …]` | `C1` | voies à mesurer |
| `--params PKPK FREQ …` | `PKPK FREQ MEAN RMS` | paramètres PAVA à lire (ignoré avec `--all`) |
| `--all` | off | lit toutes les mesures d'un coup (`PAVA? ALL`) |

```bash
python -m scope.cli measure 10.11.13.220 --channels C1
python -m scope.cli measure 10.11.13.220 --channels C1 C2 --params PKPK FREQ RISE FALL
python -m scope.cli measure 10.11.13.220 --channels C1 --all
```

Sortie type :
```
C1 PKPK: 3.3
C1 FREQ: 1000.0
```

Une mesure indisponible (signal absent, hors écran) s'affiche `--` plutôt que
de faire échouer la commande. Liste complète des paramètres reconnus :
`docs/scpi-reference.md` § Mesures automatiques.

### `discover` — découverte réseau du scope

Scanne un sous-réseau (TCP port 5025, `*IDN?`) et liste les scopes Siglent
trouvés — utile quand l'IP n'est pas connue à l'avance.

| Option | Défaut | Effet |
|--------|--------|-------|
| `--cidr <cidr>` | interfaces locales | sous-réseau à scanner (ex. `10.11.13.0/24`) |

```bash
python -m scope.cli discover                       # déduit le CIDR des interfaces locales
python -m scope.cli discover --cidr 10.11.13.0/24   # sous-réseau explicite
```

Sortie type :
```
10.11.13.220	Siglent Technologies,SDS1204X-E,SDS1EBAQ1R0001,1.3.9R6
```

Sans `--cidr`, le CIDR est déduit des interfaces IPv4 locales (`psutil`, hors
loopback) — pratique sur le montage Ethernet direct (§2.b) une fois l'adresse
attribuée à l'interface.

### `screenshot` — capture d'écran du scope

Dump l'écran du scope (SCPI `SCDP`) en BMP — pratique pour un aperçu visuel
rapide, sans passer par le décodage waveform.

| Option | Défaut | Effet |
|--------|--------|-------|
| `--outdir <dir>` | `captures` | dossier de sortie (si `--out` absent) |
| `--out <path>` | — | chemin exact du fichier `.bmp` (remplace `--outdir`) |

```bash
python -m scope.cli screenshot 10.11.13.220 --out captures/ecran.bmp
```

Le firmware ajoute parfois un octet de terminaison résiduel après les données
BMP : `screen_dump()` tronque automatiquement à la taille déclarée par
l'en-tête BMP (`bfSize`), le fichier produit est un BMP valide.

### `set` — modifier les réglages

Applique un ou plusieurs réglages puis affiche `OK`. Plusieurs options peuvent
être combinées en un appel.

| Option | Exemple | Commande SCPI émise |
|--------|---------|---------------------|
| `--timebase <val>` | `1MS`, `500US`, `2NS` | `TDIV <val>` |
| `--c1-vdiv` … `--c4-vdiv <val>` | `1V`, `50MV` | `C<n>:VDIV <val>` |
| `--coupling CH VAL` | `C1 D1M` | `C1:CPL D1M` |
| `--autoset` | — | `ASET` (auto setup firmware) |
| `--autoscale` | — | autoscale logiciel (volts/div + offset, cf. ci-dessous) |
| `--autoscale-channels C1 [C2 …]` | voies actives | voies ciblées par `--autoscale` |
| `--run` | — | `ARM` (acquisition continue) |
| `--stop` | — | `STOP` |

Couplages valides : `A1M`, `D1M`, `A50`, `D50`, `GND` (casse insensible).

```bash
python -m scope.cli set 10.11.13.220 --timebase 1MS --c1-vdiv 1V
python -m scope.cli set 10.11.13.220 --coupling C1 D1M --c1-vdiv 50MV
python -m scope.cli set 10.11.13.220 --autoset        # si rien ne déclenche
python -m scope.cli set 10.11.13.220 --autoscale      # cale volts/div + offset des voies actives
python -m scope.cli set 10.11.13.220 --autoscale --autoscale-channels C1 C2
```

**`--autoscale` (autoscale logiciel) vs `--autoset` (`ASET` firmware)** :
`--autoset` déclenche l'auto-configuration intégrée du scope (peut aussi
toucher la timebase/trigger). `--autoscale` recalcule uniquement volts/div et
offset **côté client**, à partir des mesures `PAVA? MIN`/`MAX` (2 itérations
par défaut) — utile quand l'auto-setup firmware ne cale pas bien le signal, ou
pour ne pas toucher au reste des réglages. Sans signal détectable, la commande
échoue explicitement plutôt que d'appliquer un réglage aberrant : brancher un
signal (et éventuellement faire un `--autoset` au préalable) puis réessayer.

**Bonnes pratiques (volume de données)** : `capture`/`live`/`gui` récupèrent toujours
**toute la mémoire d'acquisition** (`WFSU NP` n'est plus plafonné par défaut, cf.
`docs/architecture.md` § Mémoire profonde et décimation) — c'est nécessaire pour voir
**toute** la portée temporelle, pas seulement le début du buffer. La réduction du volume
affiché/tracé se fait entièrement **côté client**, après le fetch : `plot.plot_static`
pour les PNG, `gui.decimate`/« Points dessinés » pour le live (voir § `gui`
ci-dessous) — jamais via un plafond matériel (`WFSU NP` seul zoome sur le début du
buffer, `WFSU SP`/`MSIZ` n'ont aucun effet réel sur ce firmware, cf. Dépannage). Ce
fetch complet reste rapide grâce au socket, déjà le défaut pour `capture`/`series`/
`live`/`gui` — ne pas forcer `--vxi11` sur ces commandes sans raison.

### Changer la fréquence d'échantillonnage (`Sa/s`)

Il n'y a pas de commande directe pour régler `Sa/s` : elle est **dérivée** de la base de
temps (`TDIV`) et de la profondeur mémoire (fixe sur ce firmware, cf. Dépannage — `MSIZ`
ne fonctionne pas). Mesuré sur matériel : la fréquence reste à son maximum tant que la
fenêtre totale (`TDIV × 14 divisions`) tient dans la mémoire disponible ; au-delà, le
scope réduit automatiquement l'échantillonnage pour continuer à couvrir tout l'écran.

```bash
python -m scope.cli set 10.11.13.220 --timebase 100US   # rapide : Sa/s au maximum
python -m scope.cli set 10.11.13.220 --timebase 10MS     # lent : Sa/s réduit automatiquement
```

La fréquence résultante est visible dans la barre de statut de la GUI (`Sa/s`) ou via
`scpi> SARA?` après un `ARM`/`STOP` (elle ne se met à jour qu'après une acquisition
réelle — la lire juste après avoir changé `TDIV`, sans ré-armer, renvoie l'ancienne
valeur). Ajuster `TDIV` au fil de l'eau (« Base de temps » dans la GUI, ou `set --timebase`
répété) est la façon normale de faire varier la résolution temporelle selon le signal
observé.

### `live` — affichage temps quasi-réel

Ouvre une fenêtre pyqtgraph et ré-interroge le scope en boucle. Lance d'abord
`ARM` (acquisition continue). Fermer la fenêtre pour quitter. **Transport socket
par défaut** (`--vxi11` pour forcer VXI-11, cf. §3).

| Option | Défaut | Effet |
|--------|--------|-------|
| `--channels C1 [C2 …]` | `C1` | voies affichées |
| `--interval <ms>` | `100` | période de rafraîchissement |

```bash
python -m scope.cli live 10.11.13.220 --channels C1 C2 --interval 100
```

Chaque itération récupère **toute la mémoire d'acquisition**, puis décime
(4000 points/voie) uniquement pour le tracé — l'axe temps affiché couvre donc
toujours toute la fenêtre, jamais juste le début du buffer. En socket, une
itération prend de l'ordre de la seconde (mesuré ~1-1,6s selon la profondeur
mémoire) — c'est plus lent que `--interval` mais montre tout, pas un zoom.

Une lecture ratée (timeout ponctuel) est journalisée mais ne tue pas la boucle.

### `gui` — panneau de contrôle graphique complet

Ouvre une fenêtre PyQt5/pyqtgraph pilotable **entièrement à la souris** : contrairement
à `live` (affichage seul), `gui` regroupe l'affichage temps réel *et* les réglages
(remplace avantageusement `set` + `live` combinés). **Transport socket par défaut**
(`--vxi11` pour forcer VXI-11, cf. §3) — sans ça, l'attente après *Run* était de l'ordre
de 10s (VXI-11), ramenée à ~1-1,6s en socket.

```bash
python -m scope.cli gui 10.11.13.220
```

Charte graphique Materianova (thème clair, bascule **sombre** en haut à droite, mémorisée),
logo embarqué (`scope/assets/`, fonctionne hors ligne). Deux onglets :

- **Mesure** — trois colonnes : **Expérience** (identifiant, fiche d'expérience, boutons
  **Armer la série** / **Arrêter**), **Scope** (réglages, défilable), **live** ;
- **Analyse** — à gauche la **dernière capture de la série** (U axe gauche, I axe droit)
  avec les cœurs de plateaux retenus surlignés (vert = positifs, rouge = négatifs) ; à
  droite **U, I et f en fonction de t**, un point par capture (médiane ± σ des plateaux),
  rempli au fil de la série. En-tête : **Ouvrir une série…**, **Fiche…**, **Re-traiter**,
  et un sélecteur de capture pour parcourir une série ouverte. Voir « Extraction des
  plateaux » et « Fiche d'expérience et traçabilité » ci-dessous.

Onglet Mesure, 3 colonnes : **Expérience** (identifiant, fiche, **Enregistrement**,
boutons Armer/Arrêter) | **Scope** (pilotage de l'oscilloscope seulement) | live. Les listes
affichent des libellés lisibles (`2 mV/div`, `1 ms/div`, `DC 1 MΩ`, `Front montant`…) ; les
codes SCPI (`2MV`, `1MS`, `D1M`, `POS`) restent utilisés en interne et dans
`gui_settings.json`.

Colonne Scope :

| Cadre | Contrôles |
|-------|-----------|
| Acquisition | **Lancer** (`ARM`), **Figer** (`STOP`), **Réglage auto** (`ASET`) |
| Voies | tableau à en-têtes, une ligne par voie : **Afficher** (courbe + `TRA ON/OFF`), **Nom**, **Calibre** (`VDIV`), **Couplage** (`CPL`), **Unité**, **× Facteur** |
| Base de temps et affichage | **Base de temps** (`TDIV`, change aussi `Sa/s`, cf. § `set`) ; **Points dessinés**, défaut `4000` — décimation **purement côté client** (`gui.decimate`) après un fetch qui récupère toujours toute la mémoire : ne change que ce qui est **dessiné**, jamais la portée temporelle ni la capture ponctuelle (données brutes) |
| Déclenchement | **Voie**, **Front**, **Niveau** en volts **à l'entrée du scope** (avant le facteur de la voie ; `1`, `0,5`… validé par Entrée). Aussi utilisé par le départ « Au seuil » d'une série |
| Capture ponctuelle | **avec image PNG**, **Capturer maintenant** (mêmes fichiers que `capture`, dans le dossier des séries) |

Cadre **Enregistrement** (colonne Expérience) — mêmes paramètres que la CLI `series` :
**Cadence** (captures par seconde/minute/heure), **Durée max** (`30m`, `2h`…), **Départ** :
- **Au seuil** (défaut) : l'enregistrement démarre quand le signal franchit le réglage
  *Scope › Déclenchement* (résumé affiché sous le choix) ; **Attente max (s)** au-delà de
  laquelle rien n'est enregistré ;
- **Immédiat** ; **Après un délai** (**Délai (s)**).

Seuls les champs du départ choisi sont affichés ; le choix est mémorisé. **Analyse U / I** :
voies de l'extraction des plateaux (défaut U = C2, I = C3 : montage PEO).

**Connexion** : le scope n'est considéré comme connecté que s'il **répond** à `*IDN?`.
Tant que ce n'est pas le cas, **Armer la série** et **Capturer maintenant** restent grisés
(barre d'état : « Échec de connexion… » ; détail horodaté dans `logs/oscilloscope.log`).
Si le thread d'acquisition s'arrête (liaison perdue), une série armée est remise à zéro et
**Arrêter** fonctionne toujours. Une erreur du scope pendant une série la termine
proprement : les captures déjà faites et le `meta.json` sont conservés.

**Voies d'analyse non cochées** au moment d'armer : une boîte de dialogue propose
**Cocher et armer**, **Armer sans analyse** ou **Annuler** (sans U et I enregistrées,
l'onglet Analyse resterait vide). Installation neuve : C2 et C3 cochées par défaut.

La barre d'état affiche l'état de connexion, le nombre de points **bruts** reçus et le
débit (`Sa/s`) de la dernière trame, et les erreurs (ex. voie sans acquisition — cf.
« acquisition vide » ci-dessous).

**Nom et unité par voie** — sous chaque voie C1–C4 : un champ **nom** (légende/exports,
ex. « Vbat »), un combo **unité** éditable (défaut `V` ; suggestions `mV`/`A`/`mA`/`W`/`mW`)
et un champ **facteur** de conversion `V -> unité` (`valeur affichée = volts_lus × facteur` —
ex. sonde de courant 100 mV/A → facteur `10`). L'oscilloscope mesure toujours des volts ;
ces champs sont **purement côté client**, aucune commande SCPI n'est envoyée (contrairement
à VDIV/couplage). Deux unités actives simultanément → **double axe Y** (gauche = première
unité rencontrée, droite = seconde ; une éventuelle 3ᵉ unité partage l'axe droit). Réglages
mémorisés dans `channels_config.json` (racine du projet, même esprit que
`launcher_config.json`) et propagés à `Capture` (CSV/`.npy`/PNG : en-tête/label = nom + unité,
valeurs converties ; le fichier `.npz`/HDF5/MAT garde en plus les volts bruts et le facteur,
conversion réversible).

**Vue par défaut = toute la mémoire captée, pas l'écran physique du scope** — vérifié
normal sur matériel, cf. README § Pièges. Comme `fetch` récupère toujours toute la
mémoire (voir plus haut), la portée temporelle affichée par défaut peut largement
dépasser `TDIV × 14 divisions` (ex. 7 ms de mémoire contre ~2,8 ms d'écran pour
`TDIV=200US`) : un signal bref semble alors écrasé près de `t=0`. Ce n'est pas un
problème d'échelle/décodage — zoomer à la molette ou clic droit > *View All* (déjà
indiqué dans l'info-bulle du graphe) pour recentrer sur la portion utile.

**Architecture** : un thread d'acquisition dédié possède l'unique connexion au scope (la
ressource VISA n'est pas thread-safe) ; l'interface poste des commandes dans une file et
reçoit les trames par signaux Qt — l'UI ne gèle jamais, même sur une lecture lente
(mémoire profonde).

**Panneau Série (time-lapse)** — mêmes paramètres et même comportement que la
sous-commande `series` (§4 ci-dessus : instantanés décimés, ID d'expérience,
cadence, durée max, modes de démarrage). Cliquer **Armer la série** poste la config
dans la file du worker (même mécanisme que **Capture**) ; **Arrêter** devient
actif et demande l'arrêt à la prochaine occasion — la série en cours écrit son
`meta.json` avec les captures déjà obtenues (arrêt partiel propre, pas de perte).

Pendant qu'une série tourne :
- **le live est automatiquement suspendu** (repris tout seul à la fin si tu
  l'avais laissé en Run) — pas besoin d'y toucher ;
- **Lancer/Figer/Réglage auto, le Déclenchement, la capture ponctuelle et tout le cadre
  Enregistrement sont grisés** : ils entreraient en conflit (le départ au seuil
  reconfigure le trigger natif ; changer VDIV/TDIV pendant un fetch de série risquerait de
  désynchroniser le flux) — réactivés automatiquement à la fin ;
- le départ au seuil utilise le réglage Déclenchement affiché (voie, front, niveau) ; le
  mode de déclenchement du scope, lui, passe en SINGLE pour la détection puis en **AUTO**
  pour l'enregistrement, et y reste après la série (le scope continue d'acquérir).

Fermer la fenêtre pendant une série en cours l'arrête proprement avant de
quitter (le sidecar meta est tout de même écrit) ; le délai d'attente de
fermeture est plus long que d'habitude pour laisser le temps à un fetch en
cours de se terminer.

**Extraction des plateaux (onglet Analyse)** — `scope/plateaux.py`, source unique de
l'algorithme (le script `tools/extract_plateaux.py` l'importe pour l'analyse a posteriori des
dossiers enregistrés) : seuillage à hystérésis, statistiques sur le cœur
des plateaux, fréquence, rapport cyclique). Active si les voies choisies dans
**Analyse U / I** sont **cochées et distinctes** au moment d'**Armer** (sinon la
série tourne sans analyse, message dans la barre d'état) ; au démarrage de
l'enregistrement, l'onglet Analyse s'affiche automatiquement **si la fiche est complète**
(sinon on reste sur Mesure, cf. alerte ci-dessous). Les unités des voies sont converties en V et A avant analyse
(`mA` → A, `kV` → V…) pour que les résultats soient en V et A ; une voie U qui
n'est pas une tension (ou I pas un courant) est signalée à chaque capture. **Aucun seuil
n'est en A ou en V** : signal et plateaux sont jugés par rapport au **bruit de mesure** de
chaque voie (le plus grand du pas de quantification du scope, facteur de sonde compris,
et de l'écart-type du bruit) — même résultat que le courant soit enregistré en mA ou en
A, la sonde déclarée ×10 ou ×100 (seules les valeurs changent d'échelle). Il y a signal
si l'écart entre les niveaux haut et bas de I dépasse 5 fois ce bruit (`MIN_SNR`) ; une
polarité (plateaux + ou −, de I ou de U) n'est découpée que si son niveau dépasse lui aussi
5 fois le bruit. Aux calibres des essais PEO_N_* (pas de 0,2 A et 8 V), cela revient aux
anciens seuils 1 A et 40 V : résultats inchangés sur ces essais. Fichiers
écrits dans le dossier de la série :

| Fichier | Quand | Contenu |
|---------|-------|---------|
| `{id}_{n:04d}_plateaux.png` | à chaque capture (si plateaux) | U + I de la capture, plateaux retenus surlignés — **vérification** du découpage |
| `{id}_plateaux.csv` | fin de série | une ligne par capture (mêmes colonnes que le script) |
| `{id}_UI_vs_t.png` | fin de série | U/I/f vs t avec le fond « mode de pilotage » (courant/tension contrôlé, estimé sur toute la série) |

Le mode de pilotage n'est pas estimé en direct (il demande des captures voisines des
deux côtés) : seulement en fin de série. Critère, sur des fenêtres de captures voisines :
la grandeur pilotée suit sa consigne (constante ou rampe), elle est donc la **moins
rugueuse** (écart à une droite) ; si U et I sont aussi lisses l'un que l'autre, c'est celle
qui **ne dérive pas**. Une capture aberrante isolée est écartée de chaque fenêtre.
« indéterminé » quand U et I sont tous deux constants (rien ne permet de trancher).

**Anomalies** (colonne `flag` du CSV ; toute capture non `ok` est exclue de U(t), I(t) et
du mode de pilotage) : `no_signal` (écart haut-bas de I < 5 × son bruit de mesure : bruit
ou quantification seuls), `U_absent` (I circule, plateau U < 2,5 × le bruit de U — 20 V au
pas de 8 V),
`U_dephase` (U non synchrone de I), `no_plateau`, et — en fin de série, par comparaison
aux captures voisines — `U_saut` (U s'écarte de plus de 20 % de ses voisines alors que I
ne bouge pas : chute ponctuelle). Une **rupture** durable de U à I constant (> 25 % d'une
capture à la suivante, puis nouveau niveau stable) n'exclut rien : elle est signalée dans
la synthèse (`ruptures_U`). Le script `tools/extract_plateaux.py` écrit aussi
`{id}_controle.png` : toutes les captures en liste, une ligne chacune, anomalies sous un
voile rouge avec leur flag en toutes lettres ; chaque ligne porte en marge son numéro
en grand (`0022` = suffixe du fichier `{id}_0022.csv`, cartouche rouge si anomalie) et le
nom du fichier dans son titre. L'analyse tourne sur les instantanés **décimés**
de la série (« Points dessinés », 4000 par défaut) : sur une fenêtre de 28 ms ça fait
~7 µs/point, largement sous la durée mini d'un plateau (0,25 ms) — ne pas descendre trop
bas en points si les impulsions sont courtes. Une erreur d'analyse n'interrompt jamais
la série.

**Fiche d'expérience et traçabilité** (`scope/experiment.py`) — objectif : savoir qui a fait
quel essai, sur quoi et comment, sans cahier de labo. Tout est écrit dans le
`{id}_meta.json` de la série (pas de base centrale : un dossier copié reste autonome).

- **Identifiant** `{préfixe}_{AAAA-MM-JJ}_{NN}` (ex. `PEO_2026-10-06_02`) : préfixe
  modifiable (lettres, chiffres, tirets), numéro suivant le plus haut du jour dans le
  dossier des séries. Proposé en continu, **attribué et figé à l'armement** ; un dossier de
  série existant non vide n'est **jamais écrasé** (la série est refusée).
- **Fiche** : opérateur, objectif ; échantillon (ID, matériau, surface, préparation) ;
  bain (composition, concentration, température, pH, référence/âge) ; consigne (mode
  piloté, valeur, unité, fréquence, rapport cyclique, polarité, durée prévue) ; tags
  libres (autocomplétion sur les tags déjà utilisés dans le dossier) ; notes. Tous les
  champs sauf tags et notes sont obligatoires (`*`).
- **Pré-remplissage** : la fiche reprend l'essai précédent, **sauf l'ID d'échantillon et
  les notes** qui repartent vides (les reprendre attribuerait l'essai au mauvais
  échantillon sans que personne ne le voie).
- **Pas de blocage, une alerte** : l'enregistrement démarre sur le signal de
  l'alimentation, pas sur un clic. Dès l'armement, si la fiche est incomplète : **bandeau
  orange** en haut de la fenêtre et **champs vides en rouge**, jusqu'à ce qu'ils soient
  remplis. La fiche reste **modifiable pendant l'enregistrement** : chaque modification
  est écrite dans le meta (1,5 s après la dernière frappe) et **journalisée**.
- **Meta v2** (rétro-compatible : les clés historiques restent à la racine) : `uuid`,
  `station`, `utilisateur_windows`, `dates` (armée / début = détection / fin, ISO 8601 avec
  fuseau), heure réelle de chaque capture, `acquisition` (nom/unité/facteur des voies +
  VDIV/offset/période d'échantillonnage **lus dans les waveforms**), `fiche`, `resultats`,
  `pieces_jointes`, `analyses`, `historique` (date, utilisateur Windows, champ, avant,
  après), `empreintes` (SHA-256 de chaque fichier brut, calculées en fin de série).
- **Fiche…** (onglet Analyse, série affichée) : fiche modifiable, **résultats post-essai**
  (grandeur / valeur / unité), **pièces jointes** (copiées dans `pieces_jointes/`, jamais
  écrasées, avec leur empreinte), historique, traçabilité (dates, vérification des
  empreintes, liste des analyses).
- **Ouvrir une série…** : n'importe quel dossier de série, ancien format compris
  (PEO_N_22…). À la première ouverture d'une série sans empreintes, elles sont calculées et
  marquées **« a posteriori »** (elles prouvent l'intégrité depuis cette date seulement).
- **Re-traiter** : vérifie les empreintes, puis relance l'extraction des plateaux avec
  l'algorithme actuel et régénère `{id}_plateaux.csv`, `{id}_UI_vs_t.png`,
  `{id}_controle.png` et les `{id}_NNNN_plateaux.png` dans le dossier de la série. Les
  **données brutes ne sont jamais réécrites**. Un fichier brut modifié bloque le
  re-traitement (on peut forcer : l'analyse est alors marquée « FORCÉE ») ; une capture
  retirée volontairement est permise et tracée. Chaque analyse (série ou re-traitement)
  est ajoutée au meta avec la **version de l'algorithme** (empreinte de `plateaux.py`,
  change à toute modification) et ses paramètres.
- **Même re-traitement en script** : `python tools/extract_plateaux.py "D:\Mesures\PEO"`
  (dossier contenant les séries `PEO_N_*`, argument obligatoire) appelle la même fonction
  (`experiment.reprocess`) sur chaque dossier `PEO_N_*`, mais range les CSV et PNG dans
  `resultats/` et ne produit que `{id}_controle.png` (pas de PNG par capture). Dans le
  dossier de série, seul le `meta.json` est mis à jour (empreintes, captures retirées,
  version de l'algorithme, historique), et l'analyse note où sont ses résultats
  (`sorties`). Si une donnée brute a été modifiée, le script affiche `ERROR`, passe à la
  série suivante et se termine avec le code 1, sans jamais forcer.

**Réglages mémorisés** — tout l'onglet Mesure (voies cochées, VDIV/couplage, timebase,
trigger, points, capture, série, voies U/I, préfixe, dernière fiche, thème, onglet actif)
est sauvé dans
`gui_settings.json` (à côté de `channels_config.json`) à la fermeture et au démarrage
d'une série, puis rechargé au lancement suivant. Au rechargement, VDIV/couplage/
timebase/trigger sont **seulement affichés, pas renvoyés au scope** (pas de rafale SCPI,
cf. Dépannage) : modifier un réglage pour l'appliquer. Les voies cochées, elles, sont
réactivées normalement (nécessaire au live).

## 5. Dépannage

### `acquisition vide : aucun échantillon`

`fetch` lève cette erreur quand le descripteur renvoie `count <= 0` : **le scope
n'a pas déclenché**. Cas typique : la source de trigger est une voie **sans
signal** (ex. trigger sur C2 alors qu'on lit C1) ; en mode Normal, le scope
n'acquiert jamais.

Solutions :
- brancher un signal présent puis faire un **Auto Setup** (`set --autoset`) ;
- ou régler le trigger en mode **AUTO** sur la bonne voie ;
- vérifier la source : `scpi> TRSE?` puis `TRSE EDGE,SR,C1`.

### Ne jamais envoyer `MSIZ`, `WFSU TYPE` ni `WFSU SP` seule — abandonnées

**Vu sur matériel**, trois tentatives pour réduire le volume de données transférées ont
échoué avant `WFSU NP` (seul levier dont l'effet réel a été confirmé, cf. `control.set_waveform_points` — mais plus utilisé par défaut dans `live`/`gui`/`capture`, cf. §4) :

- `MSIZ` (profondeur mémoire) envoyée pendant que le scope est **ARMed** et interrogé en
  continu (`WF? DESC`/`DAT2`) l'a fait rester bloqué plusieurs minutes, puis a cassé
  durablement la liaison SCPI/LAN (`VI_ERROR_IO`, y compris sur une **nouvelle**
  connexion). Envoyée à l'arrêt (`STOP` puis `MSIZ 7K`), elle a été **silencieusement
  ignorée** : `MSIZ?` répondait toujours `14M`.
- `WFSU TYPE,0/1` (écran/mémoire) était accepté sans erreur mais **jamais reflété** par
  `WFSU?` (qui ne renvoie que `SP,NP,FP`) et n'a produit aucune réduction observable.
- `WFSU SP,<n>` **seule** (sans toucher `NP`) était bien confirmée par `WFSU?`, mais
  **sans effet réel** : le nombre de points restait inchangé, seul le débit (`Sa/s`)
  affiché changeait — un artefact de calcul côté client, pas une mesure réelle. `NP`
  restant à `0` (« tout envoyer ») semble prendre le pas sur `SP` pour `WF? DAT2`.

Ce firmware n'honore aucune des trois de manière fiable en remote SCPI. **Toutes ont été
retirées du code** (`control.set_memory_size`/`set_waveform_type` n'existent plus, et
`set_waveform_points` règle toujours `SP,1` en même temps que `NP`) — ne pas les
réintroduire ni les utiliser seules en console SCPI (`scpi>`). `WFSU NP` (avec `SP,1`) est
le seul mécanisme dont l'effet réel (moins de points transférés) est confirmé — mais c'est
un **zoom sur le début du buffer**, pas une décimation de toute la fenêtre (cf. §4 `capture`
et `gui` : `live`/`gui`/`capture` ne s'en servent donc plus, ils décimient côté client à la
place).

**Re-testées le 2026-07-08, toujours des culs-de-sac** — deux hypothèses plausibles
envisagées puis réfutées par la mesure : `SP` envoyée avec `NP`/`FP` dans la même commande
(`WFSU SP,4,NP,0,FP,0`) reste sans effet sur le volume transféré ; `MSIZ` renvoyée avec une
valeur de la **même famille** que celle déjà active (`MSIZ?` répondait `7M`, testé `MSIZ
7K`) reste silencieusement ignorée. Détail complet : `docs/scpi-reference.md`. **Ne plus
jamais les retester.**

Si le scope reste bloqué malgré tout (quelle qu'en soit la cause) : vérifier que l'écran
répond ; sinon réinitialiser le LAN (**Utility → I/O → LAN**) ou **redémarrer le scope**.
Si une connexion refuse soudainement (`ConnectionRefusedError`/`error creating link`) alors
qu'elle marchait avant : vérifier qu'aucune autre fenêtre (`gui`/`live`) n'est déjà ouverte
sur ce scope (VXI-11 n'accepte qu'un client à la fois) avant de chercher plus loin.

### Réponses décalées d'un cran (désynchronisation SCPI)

Symptôme : chaque query renvoie la réponse de la **précédente**. Cause : une
lecture binaire interrompue (Ctrl-C, timeout) a laissé des octets en file de
sortie. Le code s'en prémunit de deux façons :

- `flush_input()` purge la file **à chaque ouverture** de connexion ;
- `query_block()` lit les blocs binaires **octet par octet exactement** (les blocs
  WF/DESC contiennent des `\n` internes, donc on ne peut pas se fier à la
  terminaison).

Si le décalage persiste malgré tout, **rouvrir la connexion** (relancer la
commande) suffit en général ; en console, `resync()` répète `*IDN?` jusqu'à
recaler la file.

### `WAVEDESC introuvable dans le descripteur` en `live`/`gui`

Même famille de désynchronisation que ci-dessus, mais côté acquisition continue.
**Cause confirmée sur matériel (2026-07-13)** : ce n'est pas un changement de
réglage isolé qui pose problème (testé sans délai : aucun souci), mais un
**débit trop élevé** de réglages envoyés coup sur coup au scope — typiquement
une molette ou des clics rapides sur un combo (timebase, vdiv, couplage,
trigger). Chaque changement déclenchait immédiatement `write` + `WF? DESC`
suivant, sans aucune pause ; répété assez vite, ça a fini par corrompre les
lectures puis **bloquer entièrement l'interface réseau du scope** (même famille
que l'incident `MSIZ` documenté plus bas — un redémarrage matériel a été
nécessaire pour restaurer la liaison). Le message d'écran « Horizontal scale at
limit! » observé au passage n'est **pas** la cause : `CMR?` (registre d'erreur
SCPI) restait à `0` dans tous les cas testés, y compris ceux qui ont fini par
planter le scope — c'est un simple affichage du scope, sans lien causal.

**Fix appliqué** :
- **Anti-rebond côté GUI** (`MainWindow._debounced_put`/`_flush_debounced`,
  `scope/gui.py`, `DEBOUNCE_MS = 200`) : tous les réglages qui touchent le scope
  (timebase, vdiv, couplage, source/front/niveau de trigger) passent par un
  `QTimer` par réglage (clé ex. `("vdiv", "C1")`). Seul le **dernier** changement
  après 200 ms d'inactivité est réellement envoyé au scope — une rafale de N
  changements sur le même réglage n'en envoie qu'un. Vérifié manuellement (Qt
  offscreen) : 10 changements rapprochés sur la timebase → un seul `write`, la
  dernière valeur ; deux réglages distincts (ex. VDIV de C1 et C2) restent
  indépendants (l'un ne repousse pas l'envoi de l'autre).
- **Auto-guérison en filet de sécurité** : `DescriptorCache.fetch` (`scope/gui.py`,
  utilisé par `live` et `gui`) détecte toute exception de lecture, appelle
  `scope.resync()` (recalage `*IDN?`, cf. ci-dessus) puis retente **une fois**.
  Ça absorbe une désynchronisation ponctuelle du flux, mais **ne suffit pas**
  seul si le scope entre dans l'état dégradé déclenché par une rafale trop
  rapide — d'où l'anti-rebond en prévention, la vraie protection.

### Connexion lente / attente longue après *Run* (VXI-11)

L'ouverture VXI-11 passe par un handshake RPC (portmapper + `create_link`) plus
coûteux qu'un simple `connect()` TCP. `open_resource()` fixe maintenant
explicitement `open_timeout` (au lieu du défaut PyVISA `VI_TMO_IMMEDIATE`, 0 ms,
qui pouvait dégénérer en relances lentes côté RPC).

Plus significatif pour `capture`/`series`/`live`/`gui` : **chaque fetch récupère
toute la mémoire d'acquisition** (cf. §4 — en boucle pour `live`/`gui`/`series`,
en une fois pour `capture`), et ce fetch est ~30-50× plus lent en VXI-11 qu'en
socket (mesuré : ~10s contre ~0,2-1,6s). C'est pourquoi ces quatre commandes
utilisent **socket par défaut** (`live`/`gui` depuis le 2026-07-08,
`capture`/`series` depuis l'audit du 2026-07-14) — si tu as forcé `--vxi11` et
que l'attente est longue, c'est attendu ; retirer `--vxi11` (ou passer
`--socket` pour `idn`/`scpi`/`set`/`screenshot`, qui restent en VXI-11 par
défaut) :

```bash
python -m scope.cli gui 10.11.13.220                             # déjà en socket par défaut
python -m scope.cli capture 10.11.13.220 --channels C1           # déjà en socket par défaut
python -m scope.cli --socket idn 10.11.13.220                    # forcer pour idn
```

### `ping` ne répond pas (Ethernet direct)

L'interface n'a probablement pas d'IPv4 dans le bon sous-réseau. Attribuer une
adresse (voir §2.b), puis re-tester. Vérifier le câble et que le scope est bien en
mode **Static**/**DHCP** cohérent. Persistance NixOS : [`nixos-static-ip.md`](nixos-static-ip.md).

### Timeout VISA

Augmenter `--timeout` pour les très grosses acquisitions (mémoire profonde) ou un
réseau lent :

```bash
python -m scope.cli --timeout 30000 capture 10.11.13.220 --channels C1
```

### Transport : VXI-11 vs socket

Les deux transports sont validés sur le SDS1204X-E ; le défaut diffère par
commande (cf. §3). VXI-11 gère proprement les blocs binaires par construction
(flag END) mais est nettement plus lent au fetch (mesuré ~30-50×, cf.
`docs/architecture.md`) — socket est donc le défaut pour `capture`/`series`/
`live`/`gui` (qui fetchent la mémoire native), VXI-11 le reste pour les
commandes ponctuelles légères (`idn`/`scpi`/`set`/`screenshot`). Forcer l'un
ou l'autre :

```bash
python -m scope.cli --vxi11 capture 10.11.13.220 --channels C1    # forcer VXI-11
python -m scope.cli --socket idn 10.11.13.220                      # forcer socket
```

### `series` : cadence non tenue

Chaque capture de série reste un fetch mémoire complète (`--points` décime
seulement **après** réception, cf. §4). Si l'intervalle demandé
(`--rate`/`--per`) est plus court que le temps réel d'un fetch — souvent < 1s
en socket (défaut depuis l'audit du 2026-07-14), ~10s si `--vxi11` est forcé,
mesuré sur matériel (cf. `docs/architecture.md`) — la série ne saute aucune
capture mais s'exécute **plus lentement que prévu**, sans avertissement dans
la console. Seul signe : l'écart entre les horodatages réels du `meta.json` et
l'intervalle théorique (`per_seconds / rate`). Si la cadence compte, ne pas
forcer `--vxi11` ; sinon desserrer `--rate`/`--per` pour rester au-dessus du
temps de fetch réel observé.

### Fichiers volumineux (mémoire profonde)

Une trame plein écran peut faire **~3,5 millions de points** : CSV ~128 Mo,
`.npy` ~56 Mo. Préférer le `.npy` pour le stockage/relecture. Les plots sont
**décimés** (`max_points`, défaut 20 000 points/voie) et ne reflètent donc pas
chaque échantillon — les fichiers, eux, contiennent la trace complète.

## 6. Aide en ligne

```bash
python -m scope.cli -h
python -m scope.cli capture -h
```
