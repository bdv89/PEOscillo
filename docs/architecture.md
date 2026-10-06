# Architecture

Vue d'ensemble du pilotage SCPI du **Siglent SDS1204X-E** (série SDS1004X-E,
4 voies, 200 MHz) sur LAN. Le code vise deux objectifs : rester **multiplateforme**
(NixOS + Windows, sans NI-VISA) et **robuste** face aux particularités du dialecte
Siglent X-E (réponses binaires avec terminaisons ambiguës).

## Stack technique

| Couche | Choix | Pourquoi |
|--------|-------|----------|
| Transport instrument | **PyVISA** | API VISA standard, abstrait VXI-11 / socket derrière une *resource string* |
| Backend VISA | **`pyvisa-py`** | Python pur : aucune dépendance NI-VISA, identique sous NixOS et Windows |
| Décodage | **NumPy** | conversion vectorisée codes → volts, export `.npy` |
| Plot statique | **matplotlib** | export PNG, décimation pour mémoire profonde |
| Plot live | **pyqtgraph + PyQt5** | rafraîchissement fluide via `QTimer` |

`pyvisa-py` s'appuie sur `psutil` (énumération réseau) et `zeroconf` (découverte
VXI-11, optionnelle). Tous sont déclarés dans `flake.nix` / `shell.nix` /
`requirements.txt`.

## Modules

| Module | Rôle | Testable sans matériel |
|--------|------|------------------------|
| `scope/connection.py` | Ouverture/fermeture de la ressource VISA ; write/query bruts ; **lecture par bloc octet-exact** (`query_block`) ; **purge** de la file (`flush_input`, `_drain_terminator`, `resync`) ; dump d'écran (`screen_dump`, lecture brute) | non (I/O matériel) |
| `scope/waveform.py` | Récupération (`fetch`) et **décodage pur** (`decode`, `parse_descriptor`, `parse_block`) des courbes ; export mono-voie CSV/`.npy` (`save`) et export **multi-voies** (`save_capture` : un CSV combiné + un binaire par voie, réutilisé par `gui.py`/`cli.py`/`series.py`) | oui (cœur pur) |
| `scope/control.py` | Réglages : timebase, vdiv, offset, couplage, trigger (source/niveau/front/mode), run/stop, auto-setup | oui (vérifie les commandes émises) |
| `scope/acquisition.py` | Statut d'acquisition (`SAST?`) et attente de déclenchement par sondage (`INR?`, `wait_for_trigger`) — fondation de la capture conditionnelle | oui (parsers purs + `FakeScope`, horloge injectée) |
| `scope/series.py` | Série d'expérience (time-lapse) : orchestration pure (`run_series`, boucle CLI bloquante) **et** machine à états non-bloquante (`SeriesRunner.step`, pilotée pas à pas par la GUI) partageant les mêmes briques (`arm_threshold`, `resume_acquisition`, `capture_once`, `write_series_meta`). Au début de l'enregistrement, `resume_acquisition` remet le scope en acquisition continue (après un départ au seuil : `TRMD SINGLE` → `TRMD AUTO` + `ARM`). Une erreur pendant `step` termine la série proprement (`_abort` : captures et meta conservés). Capture = instantané **décimé** (`points`, défaut 4000), pas la mémoire brute | oui (cœur pur + `FakeScope`/horloge/`poll` injectés, aucune vraie temporisation) |
| `scope/cli.py` | Point d'entrée ligne de commande (`argparse`), sous-commandes `idn`/`scpi`/`capture` (`--single`)/`series`/`screenshot`/`set`/`live`/`gui` ; choix du transport (`--socket`/`--vxi11`, socket par défaut pour `capture`/`series`/`live`/`gui`, VXI-11 pour les autres) | partiel (`_capture_channels`, `_live_transport`, `cmd_capture`, `cmd_series` testés) |
| `scope/plot.py` | Tracé statique matplotlib, **décimation** (`max_points`) | non (backend graphique) |
| `scope/live.py` | Affichage temps quasi-réel pyqtgraph (socket par défaut) : `fetch` la mémoire complète puis **décime pour l'affichage** (réutilise `gui.decimate`) avant `setData` | non (Qt) |
| `scope/gui.py` | Panneau de contrôle complet (PyQt5/pyqtgraph) : réglages (anti-rebond, cf. § Rabotages), trigger, capture, série time-lapse (cadre « Enregistrement » de la colonne Expérience, pilotée par `series.SeriesRunner` — cf. § Série ci-dessous), et affichage live décimé (« Points dessinés », client uniquement). Libellés lisibles (`2 mV/div`, `DC 1 MΩ`…), codes SCPI en `itemData`. Thread d'acquisition dédié (`AcquisitionWorker`, seul propriétaire du `Scope`) ; « connecté » = `*IDN?` a répondu (Armer/Capturer grisés sinon) ; fin du thread → série remise à zéro ; Arrêter remet toujours l'UI à zéro | oui (`execute_command`, `decimate`, libellés ; fenêtre réelle hors écran avec scope simulé : `tests/test_gui_window.py`) |
| `scope/channel_config.py` | Config par voie (nom, unité, facteur de conversion V→unité) : dataclass `ChannelConfig`, persistance JSON (`channels_config.json`), et `axis_assignment` (répartition gauche/droite du double axe Y selon l'unité) ; réglages GUI mémorisés (`load/save_gui_settings`, `gui_settings.json`) | oui (cœur pur) |
| `scope/experiment.py` | Fiche d'expérience et traçabilité : ID daté (`next_id`), fiche (`FICHE_FIELDS`, `missing_fields`, `prefill`), meta v2 (`load_meta`/`migrate`, `create_meta`, `update_meta` sous verrou + écriture atomique), historique (`apply_fiche`, `set_results`), empreintes SHA-256 (`ensure_checksums`, `verify_checksums`), pièces jointes, version d'algorithme (`algo_version` = empreinte de `plateaux.py`, `algo_parameters`), `reprocess` (re-traitement sans toucher aux données brutes ; sorties dans le dossier de série — GUI — ou dans `out_dir` — script `tools/extract_plateaux.py`, le meta restant toujours à jour ; retourne la synthèse + `integrite`) | oui (cœur pur, horloge injectée) |
| `scope/theme.py` | Charte Materianova : palettes clair/sombre, QSS générée, logo recoloré pour le sombre | oui |
| `scope/gui_experiment.py` | Widgets Qt de la fiche : formulaire (`FicheForm`, champs manquants en rouge) et fenêtre « Fiche » (`FicheDialog` : résultats, pièces jointes, historique, traçabilité) | non (Qt) |
| `scope/plateaux.py` | Extraction des plateaux U/I, **source unique** (`tools/extract_plateaux.py` n'est qu'un lanceur CLI qui l'importe) : `analyse_arrays`/`analyse_waveforms` (conversion V/A via `to_si`), PNG de vérification par capture, `LiveAnalysis` (accumulation pendant une série, appelée par le worker GUI → signal `plateau_ready`), `finalize_series` (sauts de U `flag_jumps` → `U_saut` / ruptures, mode de pilotage, CSV + `UI_vs_t.png`, pandas), `save_control_png` (1 PNG par essai, une ligne par capture, numéro en grand, anomalies sous voile rouge). Seuils **relatifs au bruit de mesure** de chaque voie (`MIN_SNR`, `U_ABSENT_SNR`) : indépendants de l'unité (mA/A) et de la sonde | oui (cœur pur ; échelles A/mA/µA/kA ; non-régression capture par capture sur `samples/PEO_N_22/41/43` contre `samples/reference/`, hors dépôt) |
| `launcher.py` | Lanceur double-clic : ajoute son dossier à `sys.path` (Python embarqué), journal `logs/oscilloscope.log` (horodaté, rotation, exceptions non gérées y compris slots Qt et threads), `--check` (import complet, n'installe jamais rien), bootstrap `.venv` en dev | oui (`tests/test_launcher.py`, dont le dossier Windows avec son propre Python) |
| `install.ps1` | Installation / mise à jour idempotente (Windows) : `git pull`, `.venv` en Python 3.12 (recréé si autre version), dépendances, alignement du dossier autonome, contrôles `--check` | vérifié manuellement (copie temporaire, 3 scénarios) |
| `tools/extract_plateaux.py` | Analyse a posteriori des dossiers `PEO_N_*` via `experiment.reprocess` (sorties dans `resultats/`, seul le `meta.json` des séries est mis à jour) | oui (`tests/test_extract_plateaux.py`, sur copies) |

**Séparation des responsabilités** (prisme design) : le décodage est une fonction
*pure* (`decode` : octets/codes + échelles → `Waveform` NumPy), totalement
découplée de l'I/O VISA. C'est ce qui permet aux tests de figer des descripteurs
et blocs binaires synthétiques et de valider la conversion sans scope.

Les imports lourds sont **paresseux** : `pyvisa` est importé dans le constructeur
de `Scope`, `matplotlib`/`pyqtgraph` dans les fonctions de plot. On peut donc
importer et tester `scope.waveform` / `scope.control` sans ces dépendances.

## Flux de données

De la connexion réseau jusqu'aux fichiers / plot / live :

```
                 ┌──────────────────────────────────────────────┐
                 │  scope/cli.py  (argparse : idn|scpi|capture|  │
                 │                 set|live)                      │
                 └───────────────┬──────────────────────────────┘
                                 │ ip, --socket, --timeout
                                 ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ connection.Scope                                                  │
   │   resource_string(ip)  → "TCPIP::<ip>::INSTR"   (VXI-11, défaut)  │
   │                        → "TCPIP::<ip>::5025::SOCKET" (--socket)   │
   │   open_resource → flush_input()  ← purge tout résidu en file      │
   └───────────────┬──────────────────────────────────────────────────┘
                   │
      write/query  │  query_block (lecture IEEE-488.2 octet-exact)
                   ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ waveform.fetch(scope, "C1")                                       │
   │   1. query_block("C1:WF? DESC")  → WAVEDESC (346 o.)              │
   │        parse_descriptor → {count, vdiv, offset, interval}         │
   │        count <= 0 ?  →  ValueError "acquisition vide"             │
   │   2. query_block("C1:WF? DAT2")  → octets int8 (codes ADC)        │
   │   3. decode(codes, vdiv, offset, interval)                        │
   │        volt = code·(vdiv/25) − offset                             │
   │        t[i] = (i − n/2)·interval                                  │
   │        → Waveform(time[], volts[], …)                             │
   └───────────────┬──────────────────────────────────────────────────┘
                   │
       ┌───────────┼─────────────────────────┐
       ▼           ▼                          ▼
   save(wf)    plot.plot_static          live.run_live
   CSV + .npy  PNG (décimé, max_points)  pyqtgraph QTimer (boucle fetch)
```

## Choix de conception

### Pourquoi PyVISA + `pyvisa-py`

Un même code doit tourner sous NixOS et Windows sans installer le runtime
propriétaire NI-VISA. `pyvisa` fournit l'API standard ; le backend `@py`
(`pyvisa-py`) est une implémentation **Python pur** de VXI-11 et des sockets TCP.
NI-VISA reste utilisable s'il est déjà présent, mais n'est jamais requis.

### Pourquoi VXI-11 par défaut pour les commandes ponctuelles, socket pour le fetch waveform

`resource_string()` construit deux formes :

| Transport | Resource string | Comportement |
|-----------|-----------------|--------------|
| **VXI-11** (défaut) | `TCPIP::<ip>::INSTR` | gère proprement le flag END des blocs binaires → idéal pour lire les waveforms |
| **Socket** (`--socket`) | `TCPIP::<ip>::5025::SOCKET` | terminaison `\n` explicite, plus fragile en binaire |

VXI-11 est choisi par défaut car il délimite les transferts binaires sans se fier
à un caractère de fin (les blocs WF contiennent des `\n` internes). Le mode socket
(port 5025) est offert en repli : il a été validé sur le matériel mais impose
`read_termination = "\n"`.

> **Mesuré sur matériel (`tools/profile_capture.py`, SDS1204X-E, `C1`, 1400 pts)** :
> VXI-11 est **30 à 50× plus lent** que le socket brut pour le **même code**
> (`query_block`) — seul le transport change :
>
> | Transport | `WF? DESC` | `WF? DAT2` |
> |-----------|-----------:|-----------:|
> | VXI-11 (défaut) | ~1,2 s | ~10 s (≈ borne au timeout VISA par défaut) |
> | Socket (`--socket`) | ~0,19 s | ~0,18 s |
>
> Le CSV/`.npy`/décodage sont négligeables dans les deux cas (< 30 ms). Le goulot
> d'un instantané n'est donc ni la taille des données ni leur traitement, mais la
> couche VXI-11 de `pyvisa-py` elle-même (cohérent avec le symptôme déjà noté
> plus haut sur `open_resource`/RPC portmapper).
>
> **Défaut changé le 2026-07-08 pour `live`/`gui` : socket, pas VXI-11.** Depuis que
> ces deux commandes récupèrent toujours toute la mémoire native (cf. plus bas,
> « Mémoire profonde et décimation »), chaque frame coûte un fetch complet — en
> VXI-11 (~10s/frame), l'attente après *Run* devenait très perceptible (rapporté par
> l'utilisateur). `live`/`gui` utilisent donc `--socket` par défaut désormais
> (`cli.py:_live_transport`) ; `--vxi11` force l'ancien transport si besoin (réseau qui
> bloque le port 5025, par exemple).
>
> **Défaut étendu le 2026-07-14 (audit perf) à `capture`/`series` : même transport
> socket, même raison.** `waveform.fetch` (utilisé par `capture` en une fois, et par
> `series.capture_once` à chaque tick) fait exactement le même fetch mémoire native
> complète que `live`/`gui` — rien ne justifiait qu'elles restent sur le transport
> ~30-50× plus lent. Impact concret sur `series` : une cadence demandée plus rapide
> que le temps de fetch (~10s/voie en VXI-11 contre ~0,2-1s en socket, mesuré à
> nouveau sur matériel le 2026-07-14 : `WF? DESC` 2,15s→0,19s, `WF? DAT2`
> 10,71s→0,69s) faisait systématiquement décrocher la série. `cli._live_transport`
> (renommage à prévoir, sert maintenant les quatre commandes) est réutilisée telle
> quelle par `cmd_capture`/`cmd_series`. `idn`/`scpi`/`set`/`screenshot` restent seules
> en VXI-11 par défaut (`--socket` toujours disponible) : ce sont des requêtes
> ponctuelles légères (pas de fetch waveform), où la vitesse importe peu — et pour
> `screenshot` en particulier, VXI-11 est **requis** : `screen_dump()` lit avec
> `read_raw()` délimité par le flag END, incompatible avec la terminaison `\n` du mode
> socket sur un BMP contenant des octets `0x0A` internes.

### Transport USB (`--usb`) et rabotage des surcoûts fixes — 2026-07-08

Objectif « réactivité extrême » : le scope a été branché en USB (`lsusb` :
`f4ec:ee38`, bus USB 2.0 High Speed = 480 Mbit/s, ~4,8× le LAN 100 Mbit). Ajout
d'un troisième transport `USB0::0xF4EC::0xEE38::<serial>::INSTR` (USBTMC via
`pyvisa-py`+`pyusb`, découvert par `connection.find_usb_resource()` — le numéro
de série n'est connu qu'à l'exécution). Sélecteur : `--usb` (prioritaire sur
`--vxi11` pour `live`/`gui`, cf. `cli._live_transport`).

### Mesure socket vs USB — 2026-07-13

**Socket LAN, mesuré (`tools/profile_capture.py --socket`, C1, 3 runs, médiane) :**

| `--points` | `n_points_actual` | `dat2_s` (médiane) | Débit brut (`n/dat2_s`) |
|-----------:|-------------------:|-------------------:|------------------------:|
| 1 400 | 1 400 | ~0,255 s | ~5,5 ko/s (coût fixe par requête, non représentatif) |
| 100 000 | 100 000 | ~0,269 s | ~0,37 Mo/s (coût fixe encore dominant) |
| 0 (mémoire native) | 7 000 000 | ~1,227 s | ~5,7 Mo/s brut, **~7,2 Mo/s net** (coût fixe ~0,25 s déduit) |

Cohérent avec le plafond du LAN 100 Mbit (~12,5 Mo/s théorique, ~7 Mo/s réaliste
avec l'overhead protocolaire) — pas de signe d'incohérence de mesure. Trois runs
consécutifs quasi identiques (écart < 1 %), résultat stable.

**USB, statut : bloqué — 3 bugs logiciels réels corrigés, 1 blocage
matériel/pilote non résolu, décision de ne pas creuser davantage (cf.
« Stop architectural » ci-dessous).**

Bugs corrigés dans `scope/connection.py` (permettent maintenant l'USB pour les
requêtes SCPI simples, ex. `*IDN?`, `VDIV?`) :
1. `find_usb_resource()` cherchait un motif hex (`0xF4EC::0xEE38`) alors que
   `pyvisa-py` restitue les IDs en **décimal** avec un champ interface
   supplémentaire (`USB0::62700::60984::<serial>::0::INSTR`, 6 segments, pas 5)
   — le motif ne matchait donc jamais, même bien branché et autorisé.
   Remplacé par un filtrage manuel sur les entiers VID/PID
   (`_usb_resource_matches`), tolérant décimal et hex.
2. `flush_input()` à l'ouverture levait une `usb.core.USBError` (« Pipe
   error ») non rattrapée sur ce transport (seule `pyvisa.errors.VisaIOError`
   est catchée) — le mécanisme d'abandon-sur-timeout interne à l'USBTMC de
   `pyvisa-py` échoue lui-même sur un flush à vide. Sauté pour `usb=True`
   (pas de résidu à purger sur une liaison USB fraîchement redécouverte, à la
   différence de VXI-11/socket qui restent ouverts entre sessions).
3. `self.inst.chunk_size = CHUNK_SIZE` (32 Mio) était appliqué sans condition
   à tous les transports. Sur USB, `pyvisa-py`/`pyusb`/`libusb` soumettent des
   URB dimensionnées sur ce chunk directement au noyau via `usbfs`, qui
   plafonne la mémoire allouable (`/sys/module/usbcore/parameters/
   usbfs_memory_mb`, mesuré à **16 Mio** sur ce système) → `USBError: [Errno
   12] Insufficient memory` dès la première lecture. Un `USB_CHUNK_SIZE`
   dédié de 1 Mio (largement sous la limite, largement au-dessus d'un
   WAVEDESC de 346 o) est appliqué quand `usb=True`.

Une fois ces trois bugs corrigés, l'USB répond correctement aux requêtes SCPI
courtes (`*IDN?`, `C1:VDIV?`, `TDIV?`) — **mais `WF? DESC`/`WF? DAT2`** (le
bloc binaire de la waveform, celui qu'il fallait justement chronométrer)
**timeout systématiquement**, sans aucune donnée reçue, même avec un timeout
allongé (30 s) et le périphérique fraîchement reseté.

**Deuxième architecture testée (sur suggestion utilisateur) : contourner
`pyvisa-py`/`pyusb` et parler directement au pilote noyau `usbtmc`
(`/dev/usbtmc0`, driver déjà chargé mais pas lié à l'interface — modalias
`icFEisc03ip01` pourtant compatible, lien forcé manuellement via `/sys/bus/
usb/drivers/usbtmc/bind` + permissions `plugdev`).** Résultat initial :
intermittent, ~1 succès sur 3-4 tentatives, y compris pour `*IDN?` seul,
sans corrélation nette avec le délai après reset ni le fait d'utiliser un
process séparé.

**Root cause confirmée le 2026-07-14 par capture `usbmon` (bus USB brut,
`/sys/kernel/debug/usb/usbmon/3u`) : bug du firmware du scope, pas du
logiciel client.** Décodage de la trace d'un `C1:WF? DESC` réussi en
apparence :
- la commande part bien intégralement (transaction Bulk-OUT `017f8000
  0b000000 01000000 43313a57 463f2044 45534300` = en-tête USBTMC
  DEV_DEP_MSG_OUT + `"C1:WF? DESC"`, 11 octets) ;
- l'hôte demande la réponse (`REQUEST_DEV_DEP_MSG_IN`, jusqu'à 4096 octets) ;
- le scope répond avec un en-tête USBTMC qui annonce lui-même
  **`TransferSize = 0x171 = 369 octets`, `bmTransferAttributes = 0x01`
  (bit EOM = 1, "fin de message")** — mais ne transmet en réalité que
  **64 octets bruts** (12 d'en-tête + 52 de payload, un seul paquet USB au
  format négocié) puis **s'arrête de parler**, alors que son propre en-tête
  promettait 369 octets.

Le firmware ment donc sur la fin de message : il tronque toute réponse
`WF? DESC`/`DAT2` de plusieurs paquets à un seul, tout en signalant EOM=1 de
façon incorrecte. Un client qui accepte naïvement ce premier paquet tronqué
« réussit » (mais avec des données incomplètes, inexploitables — 52 octets
sur 346+ attendus pour un WAVEDESC) ; un client qui, comme le nôtre
(`query_block`, y compris la variante `/dev/usbtmc0` accumulant jusqu'à la
longueur déclarée par le marqueur `#<n><len>` du bloc IEEE-488.2), essaie
correctement de récupérer le reste du message attend indéfiniment des octets
qui ne viendront jamais → timeout. C'est ce qui expliquait le caractère
apparemment aléatoire des tentatives précédentes : le "succès" ou l'"échec"
dépendait seulement de si le script tentait ou non une lecture de
continuation, pas d'un aléa matériel réel.

**Conclusion définitive : l'USB est structurellement inutilisable pour le
transfert de waveform sur ce scope précis (firmware `8.3.6.1.37R17`),
indépendamment de la pile logicielle** (confirmé sur deux architectures
distinctes : pyusb/libusb en espace utilisateur et pilote noyau `usbtmc`).
Ce n'est pas un bug de code corrigible côté client — seule une mise à jour
du firmware Siglent (si elle existe et corrige ce point) ou un autre
exemplaire de l'instrument pourrait changer ce constat. **Ne pas retenter**
les mêmes pistes logicielles (elles ne peuvent pas contourner un firmware
qui ment sur la longueur de ses propres réponses). Le socket LAN reste donc
le seul transport pour lequel un débit de transfert de waveform a pu être
mesuré et est effectivement utilisable ; il est aussi, dans les faits, le
transport déjà utilisé par défaut pour `capture`/`series`/`live`/`gui` (cf. ci-dessus).

**Rabotages appliqués indépendamment du transport** (mesurés utiles quel que
soit VXI-11/socket/USB) :
- `_drain_terminator` attendait le timeout **en entier** (150 ms) à chaque
  `query_block` pour constater qu'il n'y avait plus rien à lire — payé deux
  fois par frame (DESC + DAT2). Réduit à `DRAIN_TIMEOUT_MS = 30` : les 1-2
  octets de terminaison arrivent avec le payload (même paquet), pas besoin de
  plus.
- Cache de descripteur (`gui.DescriptorCache`, repris par `live.py`) : DESC
  coûte à peu près aussi cher que DAT2 pour un petit payload (mesuré), donc le
  relire à chaque frame double le nombre de requêtes pour rien quand aucun
  réglage n'a changé. Invalidé sur toute commande qui touche vdiv/offset/
  timebase/couplage/voies actives/points ; ré-vérifié via `len(payload) ==
  desc["count"]` (auto-correction si la mémoire change sans commande connue).
  **Auto-guérison sur flux désynchronisé** (filet de sécurité, pas la
  protection principale — cf. anti-rebond ci-dessous) : sur toute exception,
  `fetch` appelle `scope.resync()` puis retente une fois avant de laisser
  l'erreur remonter (`WAVEDESC introuvable`, cf. `docs/usage.md` § Dépannage).
- **Anti-rebond des réglages** (`MainWindow._debounced_put`, `scope/gui.py`,
  `DEBOUNCE_MS = 200`) : cause racine confirmée sur matériel (2026-07-13) d'une
  désynchronisation puis d'un blocage réseau complet du scope — un **débit
  trop élevé** de réglages envoyés coup sur coup (molette/clics rapides sur
  timebase, vdiv, couplage, trigger), pas un changement isolé (testé sans
  délai : aucun souci). Chaque réglage passe par un `QTimer` dédié (clé, ex.
  `("vdiv", "C1")`) : seul le dernier changement après 200 ms d'inactivité est
  réellement envoyé au scope. Détails et repro : `docs/usage.md` § Dépannage.
- Sleep adaptatif dans `AcquisitionWorker.run` : n'ajoute plus `interval_ms`
  plein après un fetch qui a déjà pris du temps (mémoire profonde) — seul le
  temps **restant** jusqu'à l'échéance est attendu.
- `TCP_NODELAY` (Nagle désactivé) sur le socket, best-effort (API interne non
  documentée de `pyvisa-py`, échoue silencieusement si la structure change).

### Pourquoi lecture par bloc octet-exact (`query_block`)

Les réponses `WF? DESC` et `WF? DAT2` sont des **blocs IEEE-488.2** de la forme
`...#<n><len><octets>`. Le binaire contient des `0x0A` (`\n`) qui ne sont PAS des
fins de message. Un `read`/`read_raw` qui s'arrête sur un `\n` couperait le bloc et
laisserait des octets en file → **désynchronisation** : chaque requête suivante
renverrait la réponse de la précédente.

`query_block()` lit donc explicitement :

1. avance jusqu'au marqueur `#` ;
2. lit 1 chiffre = nombre de digits de longueur (`n`) ;
3. lit `n` chiffres = longueur `len` de la charge utile ;
4. lit **exactement** `len` octets ;
5. draine la terminaison résiduelle (`\n` / `\r`) avec un timeout court.

On obtient un transfert déterministe, insensible aux `\n` internes, qui ne laisse
jamais d'octets en file.

### Pourquoi `flush_input` à l'ouverture

Si une session précédente a été interrompue en pleine lecture binaire (Ctrl-C,
timeout), le scope garde des octets en file de sortie. À la connexion suivante, la
première query renverrait ce reliquat → réponses décalées d'un cran.
`flush_input()` lit en brut avec un timeout très court jusqu'à épuisement, repartant
ainsi d'une file propre. `resync()` offre un filet supplémentaire : il répète
`*IDN?` jusqu'à obtenir deux réponses « Siglent » consécutives.

### Mémoire profonde et décimation

Une trame plein écran du SDS1004X-E peut atteindre **~3,5 millions de points**
(CSV ~128 Mo, `.npy` ~56 Mo). `decode` reste vectorisé ; l'export `.npy` est
privilégié pour la compacité et la vitesse. Côté affichage, `plot_static` **décime**
(`step = len // max_points`, défaut 20 000 points/voie) car tracer plusieurs Mpts
est lent et visuellement inutile.

> **Décimation vs zoom — piège identifié le 2026-07-08.** `live.py`/`gui.py`
> plafonnaient auparavant `WFSU NP` (ex. 1400 points) pour un rafraîchissement
> rapide — mais `NP` est un **zoom sur le début du buffer** à résolution native
> (voir plus haut, `docs/scpi-reference.md`), pas une décimation sur toute la
> fenêtre. Résultat observé sur matériel : un signal parfaitement propre affichait
> du bruit ADC brut, car seule une fenêtre de quelques centaines de nanosecondes
> était réellement récupérée. Confirmé par comparaison avec les dépôts Siglent
> étudiés (`inspiration/repos/`) : `SP` (sparsing matériel) ne fonctionne pas non
> plus sur ce dialecte X-E historique (testé à nouveau, y compris `SP`+`NP`+`FP`
> ensemble — aucun effet sur le volume transféré), alors qu'il fonctionne sur le
> dialecte récent `:WAV:INT` (`eelab`, autre modèle). Le projet qui cible
> **exactement** ce modèle (`siglent-sds-mcp`) a la même conclusion : tout
> transférer, décimer côté client.
>
> **Fix appliqué** : `live.py`/`gui.py` ne plafonnent plus `NP` (mémoire native
> systématiquement transférée) ; `decimate()` (`gui.py:41-58`, déjà pure et testée)
> réduit uniquement ce qui est **dessiné**, sur toute la portée reçue. Mesuré en
> `--socket` (défaut pour live/gui depuis, cf. plus haut) : ~1,1s pour un fetch complet
> — un rafraîchissement de l'ordre de la seconde, pas instantané, mais qui montre
> **toute** la fenêtre plutôt qu'un zoom masqué. Le combo « Points affichés » de la
> GUI ne contrôle plus que cette décimation d'affichage (valeur envoyée à
> `decimate()`), plus jamais `WFSU NP`.

### Série d'expérience (time-lapse) — CLI bloquante, GUI non bloquante, mêmes briques

`scope/series.py` fournit une seule logique métier (démarrage à trois modes,
capture décimée par tick, arrêt par cadence×durée, sidecar `meta.json`) sous
**deux formes d'orchestration**, pour deux contraintes d'exécution différentes :

- **CLI** (`run_series`) : boucle bloquante classique — le process n'a rien
  d'autre à faire pendant la série, `sleep()` entre chaque tick est sans
  conséquence.
- **GUI** (`SeriesRunner.step(scope, now)`) : machine à états dont **chaque
  appel ne bloque jamais longtemps** (une capture au plus par appel, jamais de
  `sleep`). Nécessaire car le thread d'acquisition (`AcquisitionWorker`) est le
  **seul propriétaire du `Scope`** — VISA n'est pas thread-safe, et VXI-11
  n'accepte qu'**un seul client à la fois** (un 2ᵉ `Scope()` échoue avec
  `error creating link: 3`, cf. ci-dessus). Impossible donc d'exécuter
  `run_series` telle quelle dans un thread séparé (2ᵉ connexion interdite) ni
  dans le worker existant (elle gèlerait sa boucle — plus de live, plus de
  drain de la file de commandes — pour toute la durée de la série, potentiellement
  des heures). `SeriesRunner.step` est donc appelé **une fois par tour** de la
  boucle `AcquisitionWorker.run()`, en priorité sur le live (qui se retrouve
  suspendu gratuitement tant qu'une série est active, et reprend tout seul à la
  fin) — la file de commandes reste drainée entre deux tops, donc un arrêt
  demandé (`("series_stop",)`) est honoré au tour suivant, pas après la série
  entière.

Pour éviter de dupliquer la logique par-tick entre les deux orchestrations,
`run_series` et `SeriesRunner` partagent les mêmes briques : `arm_threshold`
(configuration du trigger natif), `capture_once` (fetch + décimation +
sauvegarde d'un tick, isolation des échecs par voie) et `write_series_meta`
(sidecar JSON). Seule l'orchestration (boucle bloquante vs pas-à-pas) diffère.

**Une capture = un instantané décimé, comme l'affichage — pas la mémoire
brute.** Choix explicite (revu après un premier test matériel qui produisait
~300 Mo de CSV par capture) : `capture_once` décime systématiquement à
`config.points` (défaut 4000, même défaut que le combo « Points affichés » de
la GUI) avant `save()`. Ça règle le volume disque, **pas** le temps de
transfert : chaque capture reste un fetch de toute la mémoire native, donc le
plancher de cadence réel reste borné par ce temps de fetch (voir tableau
socket/VXI-11 plus haut) — la série ne saute jamais de tick, elle ralentit
simplement si la cadence demandée est irréaliste (cf. `docs/usage.md` §
Dépannage « cadence non tenue »).

### Export multi-voies : un CSV combiné, pas un par voie — 2026-07-14

Signalement utilisateur : cocher C1/C2/C3 produisait bien 3 fichiers, mais
chacun mono-signal (« pas multi signaux ») et nommé par la voie brute (`C1`),
jamais par le label configuré. Root cause : `waveform._save_csv` n'a jamais
géré qu'une seule `Waveform`, et la CLI (`_capture_channels`) n'appliquait pas
`channels_config.json` du tout (contrairement à la GUI).

**Fix** : `waveform.save_combined_csv(waveforms, path_base)` fusionne
plusieurs `Waveform` de la **même acquisition** (même axe temps) en un seul
CSV — une colonne `<display_name>_<unit>` par voie — et lève `ValueError` si
les longueurs diffèrent plutôt que d'aligner silencieusement des colonnes
dépareillées. `waveform.save_capture(waveforms, path_base, formats)` est le
point d'entrée unique (GUI `_do_capture`, CLI `_capture_channels`, série
`capture_once`) : CSV combiné une fois (si demandé) + un fichier par voie pour
les formats binaires (`npy`/`npz`/`hdf5`/`mat`, inchangés). La CLI charge
désormais `channels_config.json` (comme la GUI) pour que le label/unité/
facteur atteignent aussi bien le nom de colonne CSV que la conversion de
valeur, sur les trois chemins d'export.

## Tests

Le découpage pur/I-O rend l'essentiel testable hors matériel : 373 tests passent (2026-10-06) via

```bash
nix-shell --run "pytest -q"
```

- `tests/test_waveform.py` : fige des WAVEDESC et blocs IEEE synthétiques, vérifie
  `parse_float`, `parse_descriptor`, `parse_block` (dont détection de troncature),
  la formule de tension, l'axe temps centré, `fetch` via un `FakeScope`, et l'export
  multi-voies (`save_combined_csv`/`save_capture` : colonnes/en-têtes, rejet des
  axes temps incompatibles, CSV combiné + binaires par voie).
- `tests/test_control.py` : vérifie les chaînes SCPI émises par chaque réglage
  (scope mocké capturant les `write`), y compris `set_trigger_mode`.
- `tests/test_connection.py` : `screen_dump()` (écriture `SCDP`, troncature à la
  taille déclarée par l'en-tête BMP `bfSize`).
- `tests/test_acquisition.py` : parsers purs (`parse_sample_status`, `parse_inr`,
  `triggered`) et `wait_for_trigger()` (horloge/attente injectées, dont la
  régression « bit déjà latché dès le premier sondage »).
- `tests/test_cli.py` : isolation des échecs par voie dans `capture`, et le choix
  du transport par défaut de `capture`/`series`/`live`/`gui` (`_live_transport`).
- `tests/test_gui.py` : dispatch SCPI (`execute_command`) et décimation
  d'affichage (`decimate`).

## Fichiers de référence (`vendor/`)

Le dossier `vendor/` conserve les fichiers d'origine Siglent (driver LabVIEW,
EasyscopeX Windows, firmware) comme **documentation/référence**. Ils ne sont pas
utilisés par le code.
