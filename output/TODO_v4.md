# TODO_v4 — Onglet Mesure lisible, départ au seuil, installation idempotente

> Date : 2026-10-06 — session « peo-plateau-extraction ». Phase 0 : analyse de la colonne
> Scope, choix validés en session (refonte sans assistant ; enregistrement dans la colonne
> Expérience ; départ au seuil par défaut puis mémorisé ; seuil en volts du scope ;
> U = C2 / I = C3 par défaut).

## Constat
- Codes SCPI bruts à l'écran (`2MV` lu « mégavolt », `1MS`, `A1M`, `POS`,
  `now/countdown/threshold`) ; voies sans en-têtes (2 lignes × 4) ; tout dans une colonne.
- Deux déclenchements concurrents (« Trigger » et « Seuil » du départ).
- Départ « now » par défaut ; analyse U/I = C1/C2 alors que le montage PEO est C2/C3.
- Lancement depuis les sources cassé par un `.venv` en Python 3.15 alpha (h5py sans wheel).

## Réalisé (TDD)
- [x] Libellés lisibles (`vdiv_label`, `tdiv_label`, `COUPLING_LABELS`, `SLOPE_LABELS`,
      `START_MODE_LABELS`, `PER_UNIT_LABELS`), codes SCPI en `itemData` ; `trigger_level_scpi`
      (« 0,5 » -> `0.5V`, ancien `1.0V` accepté).
- [x] Colonne Scope en cadres : Acquisition, Voies (tableau à en-têtes), Base de temps et
      affichage, Déclenchement, Capture ponctuelle.
- [x] Cadre « Enregistrement » dans la colonne Expérience : cadence, durée, départ en boutons
      radio (champs du mode seulement), seuil = Déclenchement (résumé en direct), analyse U/I
      avec noms de voies.
- [x] Réglages : `start_mode` = threshold, `analysis_u/i` = C2/C3, `start_timeout` ; anciennes
      clés ignorées -> nouveaux défauts appliqués une fois, puis mémorisés.
- [x] Thème : boutons d'un cadre invisibles (règle `QGroupBox > QWidget` transparente) et
      boutons radio sans cercle -> corrigés.
- [x] `install.ps1` (idempotent) + `INSTALL.md`.

## Vérification
- `tests/` : 315 OK, 1 skip.
- Rendu offscreen avec le Python embarqué (faux worker, ancien `gui_settings.json`) :
  départ au seuil, U/I = C2/C3, libellés, plus de débordement horizontal ; armement d'une
  série -> `start_mode=threshold`, seuil `{C2, 0.5V, POS, 30 s}`, champs verrouillés.
- `install.ps1` sur copie temporaire : 1er passage 43 s ; 2e passage 2 s sans changement ;
  `.venv` 3.14 détecté et recréé en 3.12 ; `--check` OK (sources et dossier autonome).
- Dossier Windows aligné sur les sources, `--check` OK.

## Critique Prisme 1
| Question | Réponse |
|----------|---------|
| Solution la plus simple ? | Oui : réorganisation des widgets existants, aucune logique d'acquisition modifiée (`series.py` inchangé) |
| Abstractions prématurées ? | Non : 2 petits helpers Qt (`_coded_combo`, `_select_code`) utilisés partout |
| Fonctionnalités spéculatives ? | Non : assistant écarté à la demande de l'utilisateur |

## Reste
- Essai sur matériel (scope réel) de la nouvelle colonne.
- `.venv` de l'utilisateur à réparer en lançant `install.ps1`.

## Correctifs terrain (2026-10-06 après-midi, installation sur une autre machine)
Symptômes : « Armer puis Arrêter n'arrête pas », « l'onglet Analyse reste muet ».
Journal de la machine : liaison ouverte (13:28:15) puis `*IDN?` expiré (13:28:29) ->
exception non interceptée -> thread d'acquisition mort, UI figée en « armée », aucune capture.
- [x] Connecté = `*IDN?` a répondu (ouverture + IDN dans le même try) ; Armer/Capturer
      grisés tant que non connecté ; fin du thread -> série remise à zéro ; Arrêter remet
      toujours l'UI à zéro.
- [x] `SeriesRunner.step` : toute erreur termine la série proprement (`_abort`, captures et
      meta conservés) au lieu de remonter.
- [x] Installation neuve : C2/C3 cochées par défaut (sinon série sur C1 seule, analyse
      désactivée en silence — reproduit) ; dialogue « Cocher et armer / Armer sans analyse /
      Annuler » si les voies d'analyse ne sont pas cochées.
- [x] Journal : connexion (succès/échec) et étapes de la série dans `logs/oscilloscope.log`.
- [x] `tests/test_gui_window.py` : fenêtre réelle hors écran, scope simulé (connexion
      échouée / IDN muet / arrêt pendant l'attente / thread disparu / erreur scope / analyse
      en direct / installation neuve / dialogue). Suite : 339 OK (.venv), 327 OK + 2 skip (Python
      système sans PyQt5).
- Cause matérielle restante : interface réseau du scope bloquée (piège documenté :
  rafale de réglages) -> redémarrer le scope ; diagnostic dans INSTALL.md.

## Scope resté sur Stop pendant l'enregistrement (2026-10-06, constaté sur le scope)
- [x] Le départ au seuil arme un trigger SINGLE, arrêté par la détection : scope sur Stop
      pendant toute la série, chaque capture relisait la même trace figée. `resume_acquisition`
      au début de l'enregistrement (GUI et CLI) : `TRMD AUTO` + `ARM` après un seuil, `ARM` seul
      sinon. AUTO plutôt que NORM : générateur arrêté => captures « sans signal », pas une
      ancienne trace répétée. Tests sur les séquences SCPI envoyées.
- Les séries enregistrées AVANT ce correctif ne contiennent qu'une vraie capture (la première).

## Confidentialité (2026-10-06)
- [x] Un test (branche de l'agent) citait un chemin réseau interne avec des prénoms ; publié dans
      le commit de fusion. Commit réécrit (force-push avec lease), chemin retiré, test limité à la
      copie locale `samples/c`. INSTALL.md : rattrapage `git fetch; git reset --hard origin/main`.
- Leçon : balayer les fichiers publiés SANS tronquer la sortie (un `head` avait masqué la ligne).
