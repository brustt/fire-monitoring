# Cartographie des surfaces brûlées en forêt : pilote 2025

* **Auteur :** brustt
* **Date :** 2026-11-03
* **État :** brouillon après une première revue multi-angles ; les décisions de la section « À trancher » restent ouvertes.

## Vue d'ensemble

Le projet produit une carte annuelle des surfaces forestières brûlées, publiée sur brustt-platform. Le pilote est une démonstration sur l'année 2025, sur la zone définie par `~/Documents/Projects/data/fire-monitoring/study_area/pilot_area_bbox.gpkg` : un rectangle tourné de 205 × 477 km, du Tarn et des Pyrénées-Orientales aux Alpes-Maritimes, qui recoupe 15 départements du Sud-Est. Une phase d'analyse viendra ensuite : comparaison entre années et méthodes de détection plus avancées.

Le fonctionnement prévu :

1. Lire les points chauds MODIS de l'année (export EFFIS `hs_modis`) et garder ceux qui touchent la zone, sur le territoire français.
2. Retirer les sources fixes, puis écarter les points qui ne sont pas près d'une forêt.
3. Regrouper les points restants en événements de feu (`FireEvent`).
4. Pour chaque événement, construire une image Sentinel-2 avant le feu et une image après le feu, sans nuages, sur une même grille, en lisant à la demande les COG du catalogue STAC, sans télécharger les scènes.
5. Calculer la zone brûlée avec un modèle remplaçable, d'abord un dNBR (différence de NBR entre avant et après). Les bandes, les NBR, les composites et le dNBR restent en mémoire : aucun raster n'est enregistré.
6. Transformer le résultat en polygones, les annoter, et les enregistrer en GeoParquet avec la date de l'événement.
7. Reconstruire le fichier annuel, générer les PMTiles et publier avec platform-publish.

Le pilote prouve que la chaîne fonctionne. Il ne prouve pas que les surfaces sont justes : la comparaison à EFFIS est un module séparé, hors pilote.

## Analyse

Rien n'a été exécuté pendant cette revue. « Vérifié » signifie qu'un relecteur a lu le code ; « déduit » signifie que le point est un raisonnement non testé. Les faits sur les catalogues STAC, l'IGN et tippecanoe n'ont pas pu être vérifiés. Les chiffres proposés sont des points de départ, pas des valeurs validées.

Les librairies ne bloquent pas le projet. Les problèmes les plus graves sont dans les définitions du concept et dans la gestion des échecs.

### Zone d'étude du pilote

- **Fichier :** `~/Documents/Projects/data/fire-monitoring/study_area/pilot_area_bbox.gpkg`, couche `pilot_area_bbox`. C'est un seul polygone en EPSG:3035, de 205 × 477 km, tourné de 89,9° : ce n'est donc pas un rectangle aligné sur les axes. Il remplace la Nouvelle-Aquitaine et les Pays de la Loire.
- **Emprise WGS84 :** de 2,04 à 8,06 °E et de 42,38 à 44,51 °N, pour 97 900 km².
- **Part en France :** environ 57 % de la surface, soit 55 900 km². Le reste est en mer ou en Italie.
- **Départements recoupés, par surface décroissante :** 04, 34, 83, 30, 13, 06, 11, 12, 84, 81, 66, 26, 48, 05, 07.
- **Grille Sentinel-2 :** la zone chevauche les fuseaux UTM 31 et 32, la limite passant à 6 °E.
- **Fichier voisin :** `study_area/pilot_area.gpkg` est vide (0 entité, avec des fichiers `-wal`/`-shm`) et n'est pas utilisé.
- **Règle :** le pipeline garde les points qui touchent ce polygone et les filtre aussi sur le territoire français, puisque les données MODIS et le masque forêt ne couvrent que la France.

### Mesures sur les points MODIS 2025

Fichier testé : export EFFIS `hs_modis` pour la France en 2025 (1 857 points, EPSG:3035, version `6.1NRT`). Il n'a ni date de début de feu, ni champ de type de source. La date est `acq_at_s`, car `acq_at` ne garde que le jour.

Fichier figé dans `~/Documents/Projects/data/fire-monitoring/MODIS/`. Regroupement testé : composantes connexes à moins de 2 km, sans filtre forêt.

**Zone du pilote** (577 points qui touchent le polygone) :

| Écart maximal | Événements | Un seul point | Plus de 30 jours | Durée maximale | Plus grande emprise |
|---|---|---|---|---|---|
| 7 jours | 258 | 185 | 1 | 132 j | 17,5 km |
| 30 jours | 235 | 167 | 2 | 280 j | 17,5 km |

- Juin à octobre regroupe 422 points sur 577 (73 %), avec un pic en août (127). Le calendrier ressemble plus à celui des feux d'été que dans l'ancienne zone.
- **Source fixe :** à 7 jours d'écart, un événement de 96 points dure du 5 juin au 15 octobre, vers 43,45 N, 4,90 E, avec des détections sur 62 jours différents. Il est situé dans la zone industrielle de Fos-sur-Mer d'après sa position ; c'est une déduction, pas une vérification. D'autres groupes au même endroit apparaissent en février et en avril. Le regroupement par écart de temps ne suffit donc pas : le filtre des détections répétées est indispensable.
- **Grand feu :** un événement de 83 points les 5 et 6 août, vers 43,06 N, 2,75 E (Aude), avec une confiance médiane de 100. Il correspondrait au grand feu des Corbières d'août 2025, ce qui reste à confirmer. C'est le candidat « grand feu » des tests. Son emprise de points dépasse les 4 km mesurés dans l'ancienne zone.

**Ancienne zone, pour mémoire** : `admin1` `FRA.10_1` et `FRA.12_1` (Nouvelle-Aquitaine et Pays de la Loire), soit 453 points :

| Écart maximal | Événements | Un seul point | Plus de 30 jours | Durée maximale | Plus grande emprise |
|---|---|---|---|---|---|
| 3 jours | 289 | 180 | 0 | 5 j | 4,0 km |
| 7 jours | 271 | 163 | 0 | 18 j | 4,0 km |
| 14 jours | 252 | 146 | 1 | 32 j | 5,3 km |
| 30 jours | 240 | 136 | 5 | 109 j | 5,3 km |

- Février à avril regroupe 229 points sur 453 (51 %), juin à août 73 (16 %). Le 20 février en compte 89 à lui seul.
- Un site vers 44,72 N, 0,81 O revient de fin avril à mi-août (24 points) : à 30 jours d'écart, il forme un événement de 109 jours.
- `night` vaut 0 partout, y compris à 02:45 : ce champ ne paraît pas fiable.

### Problèmes bloquants

Chaque point indique ce qui est réglé par les décisions prises et ce qui reste à trancher.

**1. Définition de l'événement.**

- *Problème :* une règle de voisinage en chaîne peut former un événement de plusieurs mois, si bien qu'une partie de la brûlure tombe dans l'image « avant ».
- *Réglé :*
  - regroupement par composantes connexes, qui ne dépend pas de l'ordre des points, à moins de 2 km et 7 jours d'écart au plus ;
  - `date_start` (premier point) sert de date de l'événement, et `last_detection` est le dernier point ;
  - les fenêtres avant et après sont calculées depuis ces deux dates ;
  - chaque image est un composite des dates claires retenues (3 au plus, 1 ou 2 acceptées pour l'image après) : il n'y a plus de contradiction entre « une date » et « médiane ».
- *Mesuré :* l'export EFFIS n'a pas de champ de type de source, donc les sources fixes ne peuvent être repérées que par leurs détections répétées. En reliant les points à moins de 1 km sur toute l'année, on obtient 308 sites dans la zone :

  | Jours distincts de détection | Sites | Points |
  |---|---|---|
  | 2 ou plus | 48 | 287 |
  | 3 ou plus | 8 | 175 |
  | 4 ou plus | 3 | 149 |
  | 10 ou plus | 1 | 135 |

  Le site de Fos-sur-Mer (supposé) compte 91 jours sur 11 mois. Viennent ensuite un site à 8 jours sur 158 jours (43,63 N, 4,78 E) et un site à 4 jours sur 168 jours (43,58 N, 4,40 E). Les vrais feux restent groupés sur un même mois, comme le site de l'Aude avec 3 jours en 11 jours.
- *Règle des sources fixes (décidée) :* on relie les points à moins de 1 km sur toute l'année. Un site qui compte plus de 4 détections étalées sur au moins 60 jours est une source fixe, et tous ses points sont retirés avant le regroupement. Sur 2025 :
  - parmi les 7 sites de plus de 4 détections, les feux s'étalent sur 11 jours au plus, et les deux sources fixes sur 158 et 355 jours ;
  - la règle retire Fos (supposé, 135 points) et le site de 43,62 N, 4,78 E (10 points) ;
  - le site de 43,58 N, 4,40 E (4 détections sur 168 jours) reste. Il coûtera quelques lectures Sentinel-2, sans fausser la carte si le dNBR n'y voit pas de brûlure.
  - X = 60 jours laisse de la marge pour un feu qui repart plusieurs semaines plus tard. L'échantillon est petit (2 sources fixes, 5 feux) : la règle est à revoir sur d'autres années ou zones.
- *Reste à trancher :* rien. Aucune durée maximale d'événement n'est ajoutée : une fois les sources fixes retirées, l'écart de 7 jours suffit sur 2025.

**2. Événements sans résultat.**

- *Problème :* un événement sans image claire ressemble à un événement sans brûlure. Une lecture distante qui échoue (réseau, catalogue) ne doit pas non plus passer pour une absence de brûlure.
- *Réglé :*
  - un registre `events.parquet`, avec une ligne par événement, y compris quand il ne produit aucun polygone ;
  - il contient l'état, la raison, le nombre de scènes, la part de pixels clairs, les dates, les polygones restants après chaque filtre et l'empreinte des paramètres ;
  - les dates retenues et les dates effectivement lues sont comparées ; une erreur de lecture, après les nouveaux essais de GDAL (`GDAL_HTTP_MAX_RETRY`, à régler), met l'événement en échec avec un état distinct de `no_clear_post` ;
  - un événement en échec est enregistré, le run continue, puis se termine avec un code d'erreur.
- *Reste à trancher :* la liste des états, choix d'implémentation fait dans le Technical Plan, et le taux de pixels clairs exigé, réglé par la règle de l'image après (voir ci-dessous).

**3. Doublons dans la fusion annuelle.**

- *Problème :* deux événements voisins dans le temps peuvent produire le même polygone. C'est le cas d'un feu qui reprend au même endroit plus de 7 jours après : l'image « après » du premier événement contient la brûlure du second.
- *Réglé :*
  - la fusion n'utilise pas `merge_vector` (vérifié, `eo-pipe/src/eo_pipe/steps/vector/merge.py:59-66`) ;
  - lecture avec `read_parquet`, avec échec sur tout fichier illisible ;
  - tous les polygones sont écrits en EPSG:2154.
- *Mesuré :* sur 2025, après retrait des sources fixes (577 points, dont 145 retirés, et 243 événements, sans filtre forêt), on compte les paires où un événement B commence dans la fenêtre « après » d'un événement A (`last_detection + 3 j` à `+ 45 j`) :

  | Distance entre points | Paires | B moins de 10 j après A | Événements concernés |
  |---|---|---|---|
  | 1 km ou moins | 11 | 6 | 22 |
  | 2 km ou moins | 13 | 6 | 25 |
  | 5 km ou moins | 68 | 29 | 89 |

  Environ 10 % des événements sont concernés à la marge de 1 km. Le feu de l'Aude en fait partie : un événement de 2 points le suit 9,7 jours plus tard, peut-être une reprise (non vérifié). Le doublon ne va que dans un sens : l'image « avant » de B montre déjà la brûlure de A, donc B ne la redétecte pas. Seule l'image « après » de A peut contenir la brûlure de B.
- *Règle de dédoublonnage (décidée) :* à la fusion annuelle, les polygones de B sont soustraits de ceux de A.
  - B est un voisin de A s'il commence après le `last_detection` de A et avant la dernière date de l'image « après » de A, avec des points à moins de 2 km de ceux de A.
  - Chaque surface garde la date de l'événement dont les images avant et après l'encadrent au plus près, c'est-à-dire B. Rien n'est fusionné, donc aucun polygone n'a à choisir entre deux dates.
  - Après la soustraction, le seuil de 4 ha est réappliqué aux polygones de A.
  - Si B est en échec, A garde la surface sous sa propre date, et le registre le signale par `overlaps_failed_neighbor`.
- *Reste à trancher :* rien.

**4. Ordre des filtres.**

- *Problème :* si le filtre forêt passe en dernier, chaque brûlage agricole déclenche des lectures Sentinel-2.
- *Réglé :* le tri des points par le masque forêt passe avant tout appel STAC. Il est fait une fois par run, sur la zone du pilote.
- *Mesuré :* dans le Sud-Est, ce tri retire peu de points. Sur 576 points de la zone, 183 sont dans un polygone forêt, 531 (92 %) à moins de 500 m et 567 à moins de 1 km. Les deux sources fixes sont à 0 et 55 m d'une forêt. Le masque ne remplace donc pas le filtre des sources fixes : son rôle principal est le filtrage des polygones brûlés.
- *Marge (décidée) :* 1 km fixe autour de chaque point MODIS, pour les deux usages :
  - garder un point si un polygone forêt est à moins de 1 km (567 points sur 576 dans la zone) ;
  - rattacher un polygone brûlé à son événement s'il est à moins de 1 km d'un point de cet événement.
- *Conséquence acceptée :* dans le Sud-Est, le tri ne réduit presque pas les lectures, et une brûlure sans lien avec le feu, à moins de 1 km, peut y être rattachée.
- *Reste à trancher :* rien.

**5. Identifiants et résultats périmés.**

- *Problème :* des identifiants instables et des résultats jamais invalidés, défaut déjà documenté dans `flood-monitoring/README.md:46`.
- *Réglé :*
  - les points sont figés dans `~/Documents/Projects/data/fire-monitoring/MODIS/` ;
  - `event_id` est une empreinte des clés triées de ses points ;
  - chaque événement enregistre une empreinte des paramètres, et un résultat dont l'empreinte diffère est recalculé.
- *Reste à trancher :* rien.

### Problèmes majeurs

- **Filtres mal définis (déduit).** « Relié à un point MODIS » ne peut pas vouloir dire « point dans le polygone » : un pixel MODIS couvre environ 100 ha au nadir, et plus en bord de fauchée, pour un seuil de 4 ha. Il faut une marge autour du point. « En forêt » a aussi besoin d'une règle, car découper avant ou après le test de 4 ha change le résultat. Solution proposée : créer les polygones une seule fois, les annoter (`area_ha`, `forest_ratio`, `hotspot_connected`), puis filtrer par requête. Cette table servira aussi au module EFFIS.
- **Coupes rases et écobuages (déduit).** Un brûlage de rémanents sur une coupe rase, ou un écobuage en lisière, peut passer les deux filtres. Ce faux positif restera possible dans le pilote.
- **Végétation méditerranéenne (déduit).** Dans le Sud-Est, beaucoup de feux brûlent de la garrigue ou du maquis. Le masque ne garde que la classe `Forêt` : la carte ne montrera que la part forestière de ces feux, et la description publiée doit le dire. Le seuil dNBR de 0,27 a été proposé sans tenir compte de ces milieux.
- **Filtre nuages au bon niveau (déduit).** Le pourcentage de nuages d'une scène (`eo:cloud_cover`) dit peu de chose sur une petite zone. Il faut un filtre large dans la recherche rustac, puis lire la SCL seule sur l'emprise de l'événement, mesurer la part de pixels clairs, et ne lire B8A et B12 que pour les dates retenues.
- **La SCL peut effacer les brûlures (déduit, un indice mesuré).** Une brûlure récente peut être classée « sombre » (classe 2) ou « ombre » (classe 3). Sur le feu de l'Aude, la première date claire après le feu (11 août 2025) classe 4 % de l'emprise en classe 2, probablement la brûlure elle-même. Sur l'ensemble des événements, compter la classe 2 comme claire ne change la part d'événements à 3 dates claires que d'un point. Le masque doit être testé sur un feu connu avant d'être figé.
- **Couverture de l'image après (mesuré).** SCL de `sentinel-2-c1-l2a` lue sur l'emprise de chaque événement 2025 (enveloppe de ses points + 1 km), pour chaque date de `last_detection + 3 j` à `+ 45 j` : 243 événements, 5 697 lectures, sans erreur. Classes claires : 4, 5, 6 et 7. Une date est claire si au moins T % de l'emprise est claire. Les fourchettes viennent de deux façons de combiner les tuiles d'un même jour (meilleure tuile seule, ou somme) :

  | Seuil T | Fenêtre | 3 dates claires | Juin à octobre | Reste de l'année | Aucune date claire | Jours jusqu'à la 3e date (médiane / 90e centile) |
  |---|---|---|---|---|---|---|
  | 50 % | 45 j | 98 % | 100 % | 96 % | 1 | 14 / 29 |
  | 80 % | 45 j | 97 à 98 % | 99 à 100 % | 93 à 94 % | 1 | 15 / 33 |
  | 90 % | 45 j | 94 à 97 % | 97 à 99 % | 89 à 92 % | 2 à 3 | 16 / 34 |
  | 80 % | 20 j | 68 à 70 % | 82 à 85 % | 43 à 45 % | 9 à 10 | 12 à 13 / 19 |

  Le feu de l'Aude, à 80 %, n'a des dates claires qu'à 5, 28 et 43 jours de sa dernière détection : exiger 3 dates étale son composite sur 38 jours, avec un risque de repousse. D'où la règle retenue : accepter 1 ou 2 dates.
- **Grille commune (non testé).** La zone chevauche les fuseaux UTM 31 et 32 (limite à 6 °E) : vers cette longitude, des scènes d'un même jour arrivent dans deux CRS (déduit). Les scènes doivent être lues sur une grille imposée (CRS, résolution, emprise), et celles d'un même jour assemblées avant la médiane. Il reste à vérifier ce que lazycogs fait lui-même (reprojection, assemblage par jour) et ce qu'il faut écrire en xarray, sur un événement proche de 6 °E. `event_grid` (`flood-monitoring/src/flood_monitoring/merge_stage.py:57`) peut servir à définir l'emprise. Travailler à 20 m avec B8A, B12 et SCL évite de rééchantillonner les bandes ; 4 ha représentent alors 100 pixels (déduit).
- **Offset de réflectance (mesuré sur une scène).** Un offset mal géré fausse le NBR : avec des valeurs typiques, un dNBR de 0,58 tombe à 0,40 si l'offset de −1000 reste dans les valeurs. Earth Search a deux collections L2A, et les deux déclarent `scale: 0.0001` et `offset: -0.1` dans `raster:bands` pour B8A et B12. Sur la scène `31TDH` du 29 août 2025 (référence de traitement 05.11), en médiane sur les pixels SCL eau et végétation :

  | Collection | B12 eau | B8A eau | B12 végétation | B8A végétation |
  |---|---|---|---|---|
  | `sentinel-2-l2a` | 87 | 74 | 1 013 | 3 016 |
  | `sentinel-2-c1-l2a` | 1 087 | 1 072 | 2 005 | 4 009 |

  Dans `sentinel-2-l2a`, les pixels ont déjà perdu les 1000 (`earthsearch:boa_offset_applied: true`), mais `raster:bands` annonce encore l'offset : un lecteur qui l'applique le retirerait deux fois et donnerait une eau à −0,09. Dans `sentinel-2-c1-l2a`, les pixels gardent les 1000 et les métadonnées sont justes : appliquer `raster:bands` une seule fois donne une eau à 0,009. Règle : collection `sentinel-2-c1-l2a`, réflectance = valeur × 0,0001 − 0,1, appliquée une seule fois, soit par le lecteur (lazycogs : à vérifier s'il lit `raster:bands`), soit par notre code, jamais les deux. Un test vérifie que l'eau claire sort entre 0 et 0,03.
- **Prérequis (vérifié).** Le dossier `~/Documents/Projects/data/fire-monitoring/` contient désormais la zone d'étude, les points MODIS et le masque forêt. Il manque encore `tippecanoe` et les dépendances du `pyproject.toml`. Les accès de publication n'ont pas été vérifiés. Aucune clé FIRMS n'est nécessaire pour le pilote.
- **Détails de publication (vérifié).**
  - `data_shape: single-map` s'écrit à la main, et `start` et `end` sont obligatoires (`platform-publish/docs/manifests.md:41-44`).
  - Il faut platform-publish v0.2.2 (`platform-publish/CHANGELOG.md:7`).
  - Un clic n'affiche que les propriétés des tuiles (`brustt-platform/docs/spec-v2.md:111`) : elles doivent donc porter l'identifiant, la date et la méthode.
  - tippecanoe pourrait faire disparaître les polygones de 4 ha aux zooms régionaux (déduit).
- **Conflit de vocabulaire (vérifié, `platform-publish/docs/manifests.md:39-44`).** « Event » existe déjà côté Platform. L'unité du pipeline s'appelle donc `FireEvent`.
- **Une carte qui peut tromper (déduit).** Elle montrera les feux de forêt détectés par MODIS, sans validation, et non toutes les surfaces brûlées. La description publiée doit le dire. Il faut aussi contrôler à la main environ 20 polygones et les feux connus de 2025 avant publication.

### Limites des librairies

| Librairie | Limite | Traitement |
|---|---|---|
| eo-pipe | Pas de masque SCL, composite temporel, indices, polygonisation ni interface modèle | Écrit dans ce projet avec rioxarray |
| eo-pipe | `merge_raster` garde le premier pixel, sans tenir compte des nuages | Non utilisé pour les composites |
| eo-pipe | `merge_vector` ignore les fichiers illisibles et ne reprojette pas | Non utilisé pour la fusion annuelle |
| eo-pipe | `ParallelBatch` est séquentiel ; pas de reprise de run | Reprise gérée par événement |
| eo-pipe | Version imposée `gdal==3.12.2` (`pyproject.toml:23`) ; l'import lance bash et met en cache l'environnement OTB (`steps/otb/__init__.py:7-8`) | Motif du retrait d'eo-pipe du pilote |
| platform-publish | Pas de génération PMTiles ; `brustt:dates` n'est pas pris en charge | tippecanoe ou pilote PMTiles de GDAL, à tester ; dates gardées en colonnes |
| brustt-platform | Pas de filtre temporel ni de style selon les données | Accepté pour le pilote |

eo-stac n'est pas utilisé pour l'instant : il télécharge des extraits, contraire à l'exigence de lecture cloud native. Ses limites relevées (filtre nuages par scène, alias `swir16`/`swir22` absents, pas de grille commune, `raster:bands` perdu, pas de nouvel essai sur la recherche) ne concernent donc plus le pilote.

Correction par rapport à l'inventaire initial : `S2_TILE_GRID_PATH` n'est pas figé, c'est une valeur par défaut que l'on peut remplacer (`eo-pipe/src/eo_pipe/io/tiles.py:30,120`).

Si le shell exporte l'environnement OTB, retirer eo-pipe ne suffit pas. `fire-monitoring` aura donc besoin de l'équivalent de `flood-monitoring/src/flood_monitoring/_env.py`.

## Objectifs et non-objectifs

**Dans le périmètre :**

- La chaîne complète, des points MODIS à la publication, sur 2025 et la zone `pilot_area_bbox`.
- Une interface de modèle minimale : `detect(pre, post, valid) -> DataArray`, sur une seule grille. Elle reçoit les bandes plutôt que le NBR et renvoie trois états : brûlé, non brûlé, non observé.
- Une lecture cloud native : recherche avec rustac, lecture à la demande des COG avec lazycogs, calcul en xarray et rioxarray. Aucune scène Sentinel-2 n'est téléchargée et aucun composite n'est écrit.
- Aucun raster enregistré : le dNBR et le masque brûlé sont calculés en mémoire, et seuls les polygones de l'événement sont conservés.
- Un GeoParquet par événement, qui sert de point de reprise ; le fichier annuel est reconstruit à chaque fois.
- Un registre de tous les événements et une reprise par événement. Le fichier d'état est écrit en dernier par `os.replace`, comme dans `flood-monitoring/src/flood_monitoring/raster.py:85-103`. Un événement en échec est enregistré, le run continue, puis se termine avec un code d'erreur.
- Un document des limites des librairies dans `docs/`.

**Hors périmètre :**

- La validation EFFIS.
- Le filtre par date et le style selon les données sur la Platform.
- Les modèles d'apprentissage et leur entraînement.
- La France entière, la comparaison entre années et le suivi en temps réel.
- Les frameworks de pipeline et les registres de modèles.

## Alternatives étudiées

**Rejetées :**

- **EOReader :** gère un seul produit à la fois et n'assemble pas de composites ; la page consultée ne décrit que des produits complets.
- **pyramids-core :** pas de mosaïque ni de masque nuages, et pas de rioxarray. Le conflit de GDAL n'a pas été testé et n'est plus l'argument principal si eo-pipe est retiré.
- **`merge_raster` et `merge_vector` d'eo-pipe :** voir les limites ci-dessus.
- **eo-pipe dans le pilote :** seuls `plan_chunks` et peut-être `SieveStep` serviraient, pour une version imposée de GDAL et des options de compilation propres à la machine.
- **Fenêtre fixe de plus de 30 jours après le début du feu :** une image prise pendant le feu sous-estime la brûlure, et une attente inutile laisse plus de temps à la repousse.
- **Écart de 30 jours pour le regroupement :** sur les données 2025, il crée un événement de 109 jours sur un même site.
- **Fichier unique alimenté au fil de l'eau (Parquet ou GeoPackage) :** Parquet se prête mal à l'ajout, et une reprise par événement devient plus difficile.
- **Framework de pipeline, abstraction de client FIRMS, classes abstraites et registre de modèles :** trop lourds pour une seule implémentation.
- **odc-stac et stackstac :** remplacés par lazycogs (Development Seed, v0.8.0 sur PyPI), qui lit les COG des items STAC en `DataArray` xarray paresseux par lecture asynchrone (`async-geotiff`, `obstore`), dask étant optionnel. stackstac pourra servir de référence si lazycogs bloque sur la grille ou l'offset.
- **Extraits téléchargés par eo-stac :** contraires à l'exigence de lecture cloud native. eo-stac pourra être réévalué plus tard.
- **`composite.py` de flood-monitoring :** il fabrique une composition colorée radar, pas une médiane.
- **Fenêtre « après » de 20 jours :** à 80 % de pixels clairs, seuls 68 à 70 % des événements y trouvent 3 dates claires, et 43 à 45 % hors de juin à octobre.
- **Arrêter la fenêtre « après » de A au début de B :** sur 2025, B commence 7 à 10 jours après A dans 6 paires. La fenêtre de A tomberait alors à 4 à 7 jours, avec un risque de `no_clear_post`.
- **Fusionner les polygones qui se recouvrent en gardant la date la plus ancienne :** la brûlure de B serait datée de A, avec jusqu'à 45 jours d'erreur.

**Encore à comparer :**

- Pilote PMTiles de GDAL contre tippecanoe, à tester avec `platform-publish validate`.
- VIIRS seul contre VIIRS et MODIS.
- Masque forêt IGN (millésimes 2021-2023) contre une couche plus récente, si les forêts disparues depuis gênent.
- Plus tard : déplacer GDAL et OTB d'eo-pipe derrière des options, avec un import différé.

## Décisions et étapes suivantes

### Décisions prises

- Lecture cloud native : rustac pour la recherche (déjà une dépendance de lazycogs, à la place de pystac-client), lazycogs pour la lecture à la demande des COG, xarray et rioxarray pour le calcul. Aucun téléchargement brut de Sentinel-2 ; eo-stac n'est pas utilisé pour l'instant.
- Catalogue : Earth Search d'Element84 (`https://earth-search.aws.element84.com/v1`), collection `sentinel-2-c1-l2a` (Collection 1 de l'ESA, COG publics dans `s3://e84-earth-search-sentinel-data`, région `us-west-2`, sans signature). La collection `sentinel-2-l2a` (seau `sentinel-cogs`, celui de la page du registre AWS) est écartée : ses métadonnées d'offset contredisent ses pixels. Sur la zone en 2025, les deux collections ont le même nombre de scènes à quelques unités près (par exemple 431 et 432 en août).
- Stockage : seuls les polygones de chaque événement sont enregistrés (GeoParquet). Les bandes, les NBR, les composites, le dNBR et le masque restent en mémoire. Conséquence acceptée : changer le seuil dNBR ou le modèle oblige à relire les images, ce que déclenche déjà l'empreinte des paramètres.
- eo-pipe retiré du pilote, contrairement à la consigne initiale : `rasterio.features` doit remplacer `SieveStep` et la polygonisation (non testé). Les limites restent documentées.
- Zone : polygone de `~/Documents/Projects/data/fire-monitoring/study_area/pilot_area_bbox.gpkg` (Sud-Est, 15 départements, partie française seulement), année 2025.
- Source des points : l'export EFFIS `hs_modis` 2025, figé dans `~/Documents/Projects/data/fire-monitoring/MODIS/`. VIIRS ne sera comparé que si les feux proches de 4 ha manquent.
- Seuil de surface : 4 ha.
- Marge autour des points MODIS : 1 km fixe, pour le tri par la forêt et pour le rattachement des polygones.
- Regroupement : composantes connexes, moins de 2 km et 7 jours d'écart au plus, après retrait des sources fixes : sites de 1 km avec plus de 4 détections étalées sur au moins 60 jours dans l'année.
- Dates : `date_start` est le premier point et sert de date de l'événement. `last_detection` est le dernier point : c'est la dernière détection, pas la fin réelle du feu.
- Image avant : recherche de `date_start − 60 j` à `date_start − 1 j`, les 3 dates les plus claires, en privilégiant les plus récentes.
- Image après : recherche de `last_detection + 3 j` à `last_detection + 45 j`. Une date est claire si au moins 80 % de l'emprise de l'événement (enveloppe de ses points + 1 km) est claire dans la SCL (classes 4, 5, 6 et 7). On prend les 3 premières dates claires ; s'il n'y en a qu'une ou deux, on les utilise et le registre note `n_post_dates`. Sans aucune date claire, l'événement prend l'état `no_clear_post` (1 événement sur 243 en 2025). Dans le composite, un pixel sans observation claire est « non observé ».
- Masque forêt : `~/Documents/Projects/data/fire-monitoring/masque_foret/masque-foret1_fr_hex.gpkg`, couche `masque-foret1` : 1,26 million de polygones en EPSG:2154, sur 96 départements, avec la seule classe `NATURE = Forêt`. Ses millésimes vont de 2021 à 2023 selon le département, produits par deux modèles IA. Sur la zone, il s'agit de 2021 pour 84, 30, 34, 11, 04, 66 et 48, de 2022 pour 81, 12 et 05, et de 2023 pour 26, 83, 07, 06 et 13. Une forêt brûlée ou coupée entre son millésime et 2025 peut encore y figurer. La licence n'a pas été vérifiée.
- Dédoublonnage à la fusion annuelle : les polygones d'un événement voisin B, qui commence pendant la fenêtre « après » de A à moins de 2 km, sont soustraits de ceux de A ; le seuil de 4 ha est réappliqué ; si B est en échec, A garde la surface et le registre note `overlaps_failed_neighbor`.
- Carte annuelle sans filtre de date.
- `FireEvent` comme nom de l'unité du pipeline.

### Déroulé du pilote

1. Sans lire de Sentinel-2, passer les points MODIS, le filtre des sources fixes, le filtre forêt et le regroupement sur toute la zone. Mesurer le nombre d'événements, leur durée et leur emprise, puis décider de continuer ou non selon le budget de temps et le volume à lire.
2. Traiter de bout en bout cinq événements choisis : un grand feu, un feu proche de 4 ha, une coupe rase avec point chaud, un cas nuageux, un cas proche de 6 °E (limite des fuseaux UTM). Ajouter une source fixe répétée, comme celle de Fos-sur-Mer, pour tester son filtre.
3. Publier sous un identifiant temporaire, mesurer temps, volume et échecs par événement, contrôler à la main.
4. Lancer ensuite toute la zone du pilote, un événement à la fois.

### À trancher

1. **Couverture de l'image après :** décidée, 80 % de l'emprise claire par date, 1 à 3 dates (voir « Décisions prises »).
2. **Filtres :** quel seuil sur `confidence` et quelle règle de forêt pour les polygones ? La marge est décidée : 1 km.
3. **Seuil dNBR :** proposition de 0,27. Faut-il le faire varier selon le type de forêt ?
4. **Fusion :** décidée, voir le problème bloquant 3 et « Décisions prises ».
5. **Catalogue :** décidé, Earth Search, collection `sentinel-2-c1-l2a` (voir « Décisions prises »).
6. **Masque forêt :** faut-il garder les trois usages (sans précision, « Forêt avec usage agricole déclaré », « Forêt urbaine probable ») ou exclure les deux derniers ?
7. **Événements non évalués :** faut-il les afficher sur la carte ou seulement dans le rapport ? Le registre est-il publié ?
8. **Limites d'année :** quelle année possède un feu à cheval sur deux années ? Les feux de janvier et de décembre demandent aussi des images de 2024 et 2026.
9. **Stockage :** décidé, seuls les polygones sont conservés (voir « Décisions prises »).
10. **Identifiant de publication :** les slugs ne sont jamais réutilisés. Un par année, ou un seul pour plusieurs années plus tard ?

Questions d'implémentation à régler pendant la conception :

- agrandir la zone quand une brûlure touche son bord ;
- lazycogs, à vérifier sur un prototype : reprojection vers une grille commune, assemblage des scènes d'un même jour, application de `raster:bands`, besoin ou non de dask (optionnel dans lazycogs), et passage des résultats de recherche rustac à lazycogs ; un grand feu comme celui de l'Aude, avec une emprise de points de 17,5 km, sert de cas de mesure ;
- choisir la liste des états d'un événement.

