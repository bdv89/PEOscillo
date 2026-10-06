# Pilotage du Siglent SDS1204X-E

Programme Python **multiplateforme** (NixOS + Windows) pour piloter un oscilloscope
**Siglent SDS1204X-E** (série SDS1004X-E, 4 voies, 200 MHz) en **SCPI sur LAN**, via
**PyVISA + `pyvisa-py`** (Python pur, sans NI-VISA).

## Fonctions

- `idn`        — lire l'identité du scope (`*IDN?`), valide la chaîne réseau→VISA→SCPI
- `scpi`       — console SCPI interactive
- `capture`    — récupérer des waveforms et les sauver : **un seul CSV multi-colonnes**
  (une colonne par voie, nommée par le label configuré) + `.npy` par voie par défaut
  (`--format` pour ajouter NPZ/HDF5/MAT, un fichier par voie) ; `--single` pour attendre
  un déclenchement avant de capturer
- `series`     — **série d'expérience** (time-lapse) : capture d'instantanés **décimés**
  (comme l'affichage, pas la mémoire brute) à intervalle régulier, sous un ID d'expérience,
  jusqu'à une durée max. Trois modes de démarrage : immédiat, compte à rebours, ou seuil de
  tension (trigger natif — sert aussi à « jeter le rodage »). Aussi disponible dans la
  **GUI** (panneau « Série », avec bouton Arrêter). Détails : `docs/usage.md` § `series`.
- `measure`    — mesures automatiques d'une voie (Vpp, fréquence, RMS, rise/fall time…
  via SCPI `PAVA?`)
- `discover`   — découverte réseau du scope (scan du sous-réseau, port SCPI 5025)
- `screenshot` — dump de l'écran du scope (SCPI `SCDP`) en BMP
- `set`        — régler timebase, volts/div, couplage, run/stop, auto-setup, autoscale logiciel
- `live`       — affichage temps quasi-réel (pyqtgraph), toute la mémoire décimée à l'affichage
- `gui`        — panneau de contrôle graphique complet (affichage live + réglages à la souris) ;
  chaque voie peut être **nommée** (ex. « Vbat », « Icharge ») et convertie dans une **unité**
  au choix (V par défaut, A/mA/W… via un facteur volts→unité, ex. sonde de courant) — deux
  unités actives s'affichent sur un **double axe Y**, gauche/droite. Réglages mémorisés dans
  `channels_config.json`, propagés aux exports (CSV/PNG). Panneau **Série** intégré (mêmes
  paramètres que la CLI `series`, avec bouton Arrêter et contrôles gelés pendant la série).
  Deux onglets : **Mesure** (réglages + live) et **Analyse** (dernière capture de série
  avec plateaux surlignés | U/I/f vs t, rempli à chaque capture). **Extraction des
  plateaux** en direct (`scope/plateaux.py`, source unique de l'algorithme, aussi importée par le script
  d'analyse a posteriori `tools/extract_plateaux.py`) : un PNG
  de vérification par capture, puis CSV + `UI_vs_t.png` (mode de pilotage) en fin de
  série. Réglages de l'onglet Mesure mémorisés dans `gui_settings.json`.
  **Fiche d'expérience et traçabilité** (`scope/experiment.py`) : ID daté figé
  (`PEO_2026-10-06_02`), fiche (échantillon, bain, consigne, opérateur, tags, notes) avec
  alerte si incomplète pendant la manip, historique des modifications, empreintes SHA-256
  des données brutes, résultats post-essai et pièces jointes, **Re-traiter** une série
  (même ancienne) avec l'algorithme actuel. Charte Materianova, thème clair/sombre.
  Détails : `docs/usage.md` § `gui`.

## Installation

**Windows : voir [INSTALL.md](INSTALL.md)** — script `install.ps1` (installation et mise à jour,
relançable sans risque).

### NixOS
```bash
nix develop          # avec flakes (flake.nix)
# ou
nix-shell            # sans flakes (shell.nix)
```

### Windows — dossier autonome (recommandé, aucune installation requise)

Pour un poste Windows qui ne doit **rien installer** (pas de Python, pas de pip, pas
d'internet à l'usage), construire une fois le dossier autonome — **depuis NixOS**, pas
besoin de booter Windows pour ça :

```bash
nix-shell -p python3Packages.pip --run "python packaging/build_embed.py"
```

Ça produit `dist/Oscilloscope/` : Python embeddable + toutes les dépendances déjà
installées + le code (`scope/`, `launcher.py`). Copier ce dossier tel quel sous Windows,
puis **double-cliquer `Oscilloscope.bat`** : une petite fenêtre demande l'IP du scope
(défaut `10.11.13.220` pré-rempli), mémorisée ensuite dans `launcher_config.json` — les
lancements suivants s'ouvrent directement sur la GUI, sans console.

- Si SmartScreen bloque le `.bat` (fichier non signé) : « Plus d'infos » → « Exécuter
  quand même ».
- **Journal des erreurs** : `logs/oscilloscope.log` à côté du `.bat` (horodaté, 3 fichiers
  de 1 Mo en rotation) — démarrage, échecs de lancement et toute exception non gérée, y
  compris dans l'interface ouverte et le thread d'acquisition. En cas de problème :
  `Oscilloscope-debug.bat` (console visible) et ce journal.
- Contrôle d'un dossier reconstruit, sans scope ni fenêtre :
  `python\python.exe launcher.py --check` (doit afficher `scope OK`).
- **Copie prête à l'emploi dans ce dépôt** : `Oscilloscope/` (reconstruite le 2026-10-06 :
  Python 3.12.8, pandas 3.0, fiche d'expérience, charte Materianova). Les anciennes copies
  `Oscilloscope_ancien/` et `Oscilloscope_1/` ont été supprimées.
- **Mettre à jour une copie déjà utilisée — ⚠ ne pas simplement la remplacer** : le
  `.bat` lance l'application depuis son propre dossier, et le dossier des séries par
  défaut (`captures`, relatif) se trouve donc **dans** la copie (`Oscilloscope/captures/`),
  à côté des réglages (`launcher_config.json`, `gui_settings.json`, `channels_config.json`)
  et du journal (`logs/`). Pour mettre à jour : reconstruire, puis ne remplacer que
  `python/`, `scope/`, `launcher.py` et les deux `.bat` ; ou bien pointer le **Dossier**
  des séries (colonne Expérience) hors de la copie, ce qui est recommandé de toute façon.
- **Prérequis réseau côté Windows** : l'IP statique `10.11.13.10/24` qui rend
  `10.11.13.220` joignable est un réglage NixOS (`scope-direct`, cf. § Connexion directe
  Ethernet) — sous Windows il faut configurer manuellement l'IPv4 de l'interface Ethernet
  câblée en direct sur le scope (sinon la connexion échoue même si le dossier se lance).

### Windows — environnement de développement (avec Python déjà installé)
```bat
py -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```
Le même code tourne via `pyvisa-py`. NI-VISA reste optionnel s'il est déjà installé.

`python launcher.py` (ou double-clic) détecte que les dépendances manquent, crée `.venv`,
installe `requirements.txt` si besoin, puis ouvre directement le panneau de contrôle
graphique (`gui`) — utile en dev, mais exige Python installé et internet au premier
lancement (contrairement au dossier autonome ci-dessus).

## Brancher le scope (bring-up)

1. Sur le scope : **Utility → I/O → LAN**, relever (ou fixer) l'**IP**.
2. Côté PC : `ping <ip>` doit répondre.
3. Tester la liaison complète :
   ```bash
   python -m scope.cli idn <ip>
   # -> Siglent Technologies,SDS1204X-E,<n° série>,<firmware>
   ```

## Exemples

```bash
python -m scope.cli scpi 192.168.1.50
python -m scope.cli capture 192.168.1.50 --channels C1 C2 --plot
python -m scope.cli capture 192.168.1.50 --single --trigger-timeout 5
python -m scope.cli capture 192.168.1.50 --format csv npz hdf5 mat
python -m scope.cli series 192.168.1.50 --id essai42 --rate 10 --per minute --duration 30m
python -m scope.cli series 192.168.1.50 --id essai42 --rate 1 --per second --duration 5m \
    --start threshold --threshold-channel C1 --threshold-level 1V
python -m scope.cli measure 192.168.1.50 --channels C1 --all
python -m scope.cli discover --cidr 192.168.1.0/24
python -m scope.cli screenshot 192.168.1.50 --out captures/ecran.bmp
python -m scope.cli set 192.168.1.50 --timebase 1MS --c1-vdiv 1V
python -m scope.cli set 192.168.1.50 --autoscale
python -m scope.cli live 192.168.1.50 --channels C1
python -m scope.cli gui 192.168.1.50
```

## Licence

Vitrine (anglais, appel Horizon Europe RAISE) : https://github.com/bdv89/PEO-showcase

Apache 2.0 (voir [LICENSE](LICENSE)). Le nom, le logo et la charte graphique
Materianova ne sont pas couverts par cette licence (voir [NOTICE](NOTICE)).

## Tests

```bash
pytest        # 373 tests (2026-10-06) : décodage, pilotage, mesures, autoscale, discovery, export
              # multi-format, export multi-voies (CSV combiné), statut/trigger, screenshot,
              # série time-lapse (CLI + GUI), CLI, launcher (détection deps + résolution IP),
              # extraction des plateaux (seuils relatifs au bruit, mA/A, bipolaire ;
              # non-régression sur samples/PEO_N_22/41/43, hors dépôt),
              # fenêtre GUI réelle hors écran avec scope simulé (connexion,
              # Arrêter, analyse en direct, installation neuve),
              # réglages GUI mémorisés, fiche d'expérience / meta v2 / empreintes /
              # re-traitement, charte graphique
              # — sans matériel (scope mocké)
```

## Documentation

Documentation détaillée dans [`docs/`](docs/) :

- [`docs/architecture.md`](docs/architecture.md) — rôle de chaque module, flux de
  données et choix de conception (PyVISA/`pyvisa-py`, VXI-11 vs socket, lecture
  octet-exact `query_block`, `flush_input`).
- [`docs/usage.md`](docs/usage.md) — installation (NixOS / Windows), mise en service
  de la liaison, chaque sous-commande de la CLI avec exemples, et un guide de
  dépannage.
- [`docs/scpi-reference.md`](docs/scpi-reference.md) — commandes SCPI utilisées,
  format de lecture waveform (`WF? DESC`/`DAT2`, offsets WAVEDESC) et formule de
  décodage Siglent.
- [`docs/nixos-static-ip.md`](docs/nixos-static-ip.md) — IP statique persistante
  sur l'interface Ethernet (NixOS).
- [`docs/siglent-sds-programming-guide.pdf`](docs/siglent-sds-programming-guide.pdf)
  — guide de programmation officiel Siglent (série SDS1000X-E / SDS1004X-E).

## Notes de connexion

- Transport : **VXI-11** (`TCPIP::<ip>::INSTR`) par défaut pour `idn`/`scpi`/`set`/
  `screenshot` (requêtes ponctuelles, l'écart de vitesse importe peu) ; **socket brut**
  (port 5025) par défaut pour `capture`/`series`/`live`/`gui` (mesuré ~30-50× plus rapide
  pour un fetch complet — `--vxi11`/`--socket` pour forcer l'un ou l'autre). `screenshot`
  reste en VXI-11 quel que soit le contexte : `screen_dump()` lit avec `read_raw()`
  délimité par le flag END, incompatible avec la terminaison `\n` du mode socket sur un
  BMP contenant des octets `0x0A`. Les deux transports sont validés sur le SDS1204X-E.
- À l'ouverture, la connexion **purge automatiquement** tout résidu en file
  (`flush_input`) et les blocs binaires (waveform/descripteur) sont lus **octet par
  octet exactement** (`query_block`) — indispensable car le scope termine ses réponses
  par `\n`/`\n\n` et le binaire contient des `\n` internes (sinon désynchronisation).
- `vendor/` contient les fichiers d'origine (driver LabVIEW, EasyscopeX Windows, firmware) —
  conservés comme **référence/documentation**, non utilisés par le code.
- `inspiration/` : veille de dépôts GitHub externes utiles au projet (catalogue +
  quelques clones de référence), voir `inspiration/README.md`.
- `tools/profile_capture.py` : script autonome de profilage (DESC/DAT2/décodage/écriture),
  pour mesurer avant d'optimiser — ne fait aucune optimisation lui-même.
- `packaging/build_embed.py` : construit `dist/Oscilloscope/` (dossier Windows autonome,
  cf. § Installation Windows) — téléchargements mis en cache dans `packaging/_downloads/`
  et `packaging/_wheelhouse/`, à supprimer sans risque pour forcer un rebuild propre.
  **Vider `_wheelhouse/` est obligatoire après un changement de dépendances** : le script
  extrait *tous* les wheels du cache, et une ancienne version d'un paquet s'y mélangerait
  à la nouvelle (vu le 2026-10-06 : deux numpy dans le même `site-packages`).

## Pièges vérifiés sur le matériel

- **Capture vide** (`acquisition vide : aucun échantillon`) : le scope doit **déclencher**.
  Si la source de trigger est une voie sans signal (ex. trigger sur C2 alors qu'on lit C1),
  en mode Normal il n'acquiert jamais. Brancher un signal et/ou faire un **Auto Setup**, ou
  passer en trigger AUTO sur la bonne voie.
- **Échelles lues depuis le descripteur** : `fetch` lit `C<n>:WF? DESC` (bloc WAVEDESC) pour
  VDIV/OFST/intervalle, puis `C<n>:WF? DAT2` pour les données. Conversion Siglent :
  `volt = code·(VDIV/25) − OFST`, `t[i] = (i − n/2)·intervalle`.
- **Mémoire profonde** : une trame plein écran peut faire plusieurs **millions de points**
  (ex. 3,5 Mpts → CSV ~128 Mo). Le `.npy` est plus compact ; les plots sont **décimés**
  côté client (jamais via un plafond matériel — `WFSU NP` seul est un zoom, pas une
  décimation ; `WFSU SP`/`MSIZ` n'ont aucun effet réel sur ce firmware, cf.
  `docs/scpi-reference.md`).
- **Capture déclenchée** (`capture --single`) : arme un trigger unique et attend le
  déclenchement (sondage `INR?`, pas la commande bloquante `WAIT`) avant de lire —
  fondation pour une future capture automatisée sous condition.
- **`series` : la cadence demandée est un objectif, pas une garantie.** Chaque capture
  reste un fetch mémoire complète (juste décimé *après* réception, cf. `--points`) : si
  l'intervalle demandé est plus court que le temps de transfert réel (**socket par
  défaut depuis l'audit du 2026-07-14** — souvent < 1s, contre ~10s si `--vxi11` est
  forcé, mesuré sur matériel — cf. `docs/architecture.md`), la série enchaîne aussi vite
  que possible **sans sauter de capture**, mais plus lentement que prévu (dérive
  silencieuse sinon détectée que par l'horodatage réel du `meta.json`). Ne pas forcer
  `--vxi11` pour une série si la cadence compte.
- **« Signal écrasé à gauche » en `live`/`gui` — pas un bug d'échelle** : la fenêtre
  affichée par défaut couvre **toute la mémoire captée** (ex. 3,5 Mpts à 500 MSa/s =
  7 ms), pas seulement les `TDIV × 14 divisions` que montrerait l'écran physique du
  scope (ex. ~2,8 ms pour `TDIV=200US`). Un signal bref apparaît donc collé près de
  `t=0` sur un axe bien plus large que prévu. C'est le prix du choix « tout
  transférer, décimer côté client » (cf. point précédent) : zoomer à la molette ou
  clic droit > *View All* pour recentrer sur la portion utile — vérifié normal, cf.
  `docs/usage.md` § `gui`.
- **Rafale de réglages en `live`/`gui` = risque de blocage réseau du scope** : ce n'est
  pas un changement isolé qui pose problème (testé sans délai : aucun souci), mais un
  **débit trop élevé** de réglages envoyés coup sur coup (molette/clics rapides sur
  timebase, vdiv, couplage, trigger) — reproduit et confirmé sur matériel, jusqu'au
  blocage complet de l'interface réseau du scope (redémarrage nécessaire). Le message
  d'écran « Horizontal scale at limit! » observé au passage n'est **pas** la cause
  (`CMR?` restait à `0`). **Anti-rebond appliqué** (`DEBOUNCE_MS = 200` dans
  `scope/gui.py`) : seul le dernier changement après 200 ms d'inactivité par réglage
  est réellement envoyé. Détails : `docs/usage.md` § Dépannage.

## Connexion directe Ethernet (NixOS)

Si le scope est câblé en direct sur le port Ethernet (sous-réseau `10.11.13.0/24` ici),
attribuer une IP à l'interface (temporaire) :

```bash
sudo ip addr add 10.11.13.10/24 dev enp0s31f6
```

Pour rendre ça permanent, ajouter une adresse statique sur l'interface dans
`/etc/nixos/configuration.nix` (`networking.interfaces.enp0s31f6.ipv4.addresses`).
