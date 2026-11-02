# Cartographie des surfaces brûlées en forêt : pilote 2025

* **Auteur :** brustt
* **Date :** 2026-11-02
* **État :** brouillon après une première revue multi-angles ; les décisions de la section « À trancher » restent ouvertes.

## Vue d'ensemble

Le projet produit une carte annuelle des surfaces forestières brûlées, publiée sur brustt-platform. Le pilote est une démonstration sur l'année 2025, en Nouvelle-Aquitaine et Pays de la Loire (17 départements). Une phase d'analyse viendra ensuite : comparaison entre années et méthodes de détection plus avancées.

Le fonctionnement prévu :

1. Récupérer les points chauds FIRMS de l'année sur la zone.
2. Écarter les points qui ne sont pas près d'une forêt.
3. Regrouper les points restants en événements de feu (`FireEvent`).
4. Pour chaque événement, construire une image Sentinel-2 avant le feu et une image après le feu, sans nuages, sur une même grille.
5. Calculer la zone brûlée avec un modèle remplaçable, d'abord un dNBR (différence de NBR entre avant et après).
6. Transformer le résultat en polygones, les annoter, et les enregistrer en GeoParquet avec la date de l'événement.
7. Reconstruire le fichier annuel, générer les PMTiles et publier avec platform-publish.

Le pilote prouve que la chaîne fonctionne. Il ne prouve pas que les surfaces sont justes : la comparaison à EFFIS est un module séparé, hors pilote.

## Analyse

Rien n'a été exécuté pendant cette revue. « Vérifié » signifie qu'un relecteur a lu le code ; « déduit » signifie que le point est un raisonnement non testé. Les faits sur FIRMS, les catalogues STAC, l'IGN et tippecanoe n'ont pas pu être vérifiés. Les chiffres proposés sont des points de départ, pas des valeurs validées.

Les librairies ne bloquent pas le projet. Les problèmes les plus graves sont dans les définitions du concept et dans la gestion des échecs.

### Mesures sur les points MODIS 2025

Fichier testé : export EFFIS `hs_modis` pour la France en 2025 (1 857 points, EPSG:3035, version `6.1NRT`). Il n'a ni date de début de feu, ni champ de type de source. La date est `acq_at_s`, car `acq_at` ne garde que le jour.

Sous-ensemble : `admin1` `FRA.10_1` et `FRA.12_1`, supposés être la Nouvelle-Aquitaine et les Pays de la Loire d'après leurs emprises, soit 453 points. Regroupement testé : composantes connexes à moins de 2 km, sans filtre forêt.

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

**1. L'événement n'est pas défini (déduit).** Avec une règle de voisinage en chaîne (moins de 2 km, moins de 30 jours), une source fixe comme un site industriel ou une longue saison de feux peut former un seul événement de plusieurs mois, sur une très grande emprise. Une partie de la brûlure tomberait alors dans l'image « avant ». Par ailleurs, « une date après le feu » contredit « médiane temporelle ». Il faut :

- un regroupement qui ne dépend pas de l'ordre des points (composantes connexes) ;
- le retrait des sources fixes connues de FIRMS ;
- une durée et une emprise maximales ;
- des fenêtres avant et après calculées depuis le premier et le dernier point ;
- une définition explicite de la « date de l'événement ».

**2. Un événement sans image claire ressemble à un événement sans brûlure (déduit ; les échecs de téléchargement ne produisent qu'un avertissement, vérifié dans `eo-stac/src/eo_stac/download.py:209-213`).** Chaque événement doit finir avec un état et une raison, y compris s'il ne produit aucun polygone. Il faut un registre avec une ligne par événement : état, raison, nombre de scènes, part de pixels clairs, dates, polygones restants après chaque filtre, empreinte des paramètres. Il faut aussi comparer les fichiers demandés et reçus, comme dans `flood-monitoring/src/flood_monitoring/download.py:47-53`.

**3. La fusion annuelle peut compter deux fois la même brûlure (déduit).** Deux événements dont les emprises se recouvrent peuvent produire le même polygone. En outre, `merge_vector` ignore les fichiers illisibles et applique le CRS du premier fichier à tous les autres (vérifié, `eo-pipe/src/eo_pipe/steps/vector/merge.py:59-66`). Il faut écrire en EPSG:2154, lire avec `read_parquet` en échouant sur tout fichier illisible, et fixer une règle de dédoublonnage.

**4. Le filtre forêt arrive trop tard (déduit).** S'il n'est appliqué qu'à la fin, chaque brûlage agricole déclenche des téléchargements Sentinel-2, pour un nombre d'événements encore inconnu. Le masque forêt doit être préparé en premier et servir à trier les points avant tout appel STAC, avec une marge couvrant l'erreur de position des points.

**5. Les identifiants d'événement sont instables et les anciens résultats ne sont jamais invalidés (déduit ; le même défaut est documenté dans `flood-monitoring/README.md:46`).** Il faut figer une copie des données FIRMS, calculer `event_id` comme une empreinte des points triés, et enregistrer une empreinte des paramètres.

### Problèmes majeurs

- **Filtres mal définis (déduit).** « Relié à un point FIRMS » ne peut pas vouloir dire « point dans le polygone » : un pixel VIIRS couvre environ 14 ha au nadir, pour un seuil de 4 ha. Il faut une marge autour du point. « En forêt » a aussi besoin d'une règle, car découper avant ou après le test de 4 ha change le résultat. Solution proposée : créer les polygones une seule fois, les annoter (`area_ha`, `forest_ratio`, `hotspot_connected`), puis filtrer par requête. Cette table servira aussi au module EFFIS.
- **Coupes rases des Landes (déduit).** Un brûlage de rémanents sur une coupe rase peut passer les deux filtres. Ce faux positif restera possible dans le pilote.
- **Filtre nuages au mauvais niveau (filtre vérifié dans `eo-stac/src/eo_stac/params.py:45-49`, effet déduit).** Le pourcentage de nuages d'une scène dit peu de chose sur une petite zone. Il faut un filtre large, puis télécharger la SCL, mesurer la part de pixels clairs sur la zone, et ne télécharger les bandes que pour les dates utiles.
- **La SCL peut effacer les brûlures (déduit).** Une brûlure récente peut être classée « sombre » (classe 2) ou « ombre » (classe 3). Le masque doit être testé sur un feu connu avant d'être figé.
- **Grille commune à écrire (vérifié, `eo-stac/src/eo_stac/download.py:94-96,720-739`).** Les extraits gardent le CRS d'origine et ne sont jamais rééchantillonnés. Autour de la longitude 0°, le recouvrement des tuiles rendrait le mélange de CRS courant (déduit). Il faut assembler les doublons d'un même jour avant la médiane. `event_grid` et `warp` (`flood-monitoring/src/flood_monitoring/merge_stage.py:57,69`) sont réutilisables. Travailler à 20 m avec B8A, B12 et SCL évite de rééchantillonner les bandes ; 4 ha représentent alors 100 pixels (déduit).
- **Offset de réflectance non géré (absence vérifiée par recherche dans `eo-stac`, effet déduit).** Un offset non retiré fausse le NBR, même sur la seule année 2025. Il faut contrôler un produit réel de chaque collection retenue et noter la règle.
- **Prérequis absents (vérifié).** Il manque le dossier de données, les fichiers FIRMS et forêt, `tippecanoe` et les dépendances du `pyproject.toml`. La clé FIRMS et les accès de publication n'ont pas été vérifiés.
- **Détails de publication (vérifié).**
  - `data_shape: single-map` s'écrit à la main, et `start` et `end` sont obligatoires (`platform-publish/docs/manifests.md:41-44`).
  - Il faut platform-publish v0.2.2 (`platform-publish/CHANGELOG.md:7`).
  - Un clic n'affiche que les propriétés des tuiles (`brustt-platform/docs/spec-v2.md:111`) : elles doivent donc porter l'identifiant, la date et la méthode.
  - tippecanoe pourrait faire disparaître les polygones de 4 ha aux zooms régionaux (déduit).
- **Conflit de vocabulaire (vérifié, `platform-publish/docs/manifests.md:39-44`).** « Event » existe déjà côté Platform. L'unité du pipeline s'appelle donc `FireEvent`.
- **Une carte qui peut tromper (déduit).** Elle montrera les feux de forêt détectés par FIRMS, sans validation, et non toutes les surfaces brûlées. La description publiée doit le dire. Il faut aussi contrôler à la main environ 20 polygones et les feux connus de 2025 avant publication.

### Limites des librairies

| Librairie | Limite | Traitement |
|---|---|---|
| eo-pipe | Pas de masque SCL, composite temporel, indices, polygonisation ni interface modèle | Écrit dans ce projet avec rioxarray |
| eo-pipe | `merge_raster` garde le premier pixel, sans tenir compte des nuages | Non utilisé pour les composites |
| eo-pipe | `merge_vector` ignore les fichiers illisibles et ne reprojette pas | Non utilisé pour la fusion annuelle |
| eo-pipe | `ParallelBatch` est séquentiel ; pas de reprise de run | Reprise gérée par événement |
| eo-pipe | Version imposée `gdal==3.12.2` (`pyproject.toml:23`) ; l'import lance bash et met en cache l'environnement OTB (`steps/otb/__init__.py:7-8`) | Motif du retrait d'eo-pipe du pilote |
| eo-stac | Filtre nuages au niveau de la scène uniquement | Part de pixels clairs calculée dans ce projet |
| eo-stac | Alias `swir1`/`swir2` sans `swir16`/`swir22` d'Earth Search | Clés brutes ou `extra_aliases` (`aliases.py:65`) |
| eo-stac | Pas de grille commune ni d'offset ; métadonnées d'assets `raster:bands` perdues (`search.py:317-324`) | Grille et offset gérés dans ce projet |
| eo-stac | Pas de nouvel essai sur la recherche, d'après le code (`search.py:212`) ; noms d'extraits basés sur l'emprise, donc tout regroupement modifié retélécharge (`storage.py:65`) | Figer les événements avant les téléchargements |
| platform-publish | Pas de génération PMTiles ; `brustt:dates` n'est pas pris en charge | tippecanoe ou pilote PMTiles de GDAL, à tester ; dates gardées en colonnes |
| brustt-platform | Pas de filtre temporel ni de style selon les données | Accepté pour le pilote |

Correction par rapport à l'inventaire initial : `S2_TILE_GRID_PATH` n'est pas figé, c'est une valeur par défaut que l'on peut remplacer (`eo-pipe/src/eo_pipe/io/tiles.py:30,120`).

Si le shell exporte l'environnement OTB, retirer eo-pipe ne suffit pas. `fire-monitoring` aura donc besoin de l'équivalent de `flood-monitoring/src/flood_monitoring/_env.py`.

## Objectifs et non-objectifs

**Dans le périmètre :**

- La chaîne complète de FIRMS à la publication, sur 2025 et 17 départements.
- Une interface de modèle minimale : `detect(pre, post, valid) -> DataArray`, sur une seule grille. Elle reçoit les bandes plutôt que le NBR et renvoie trois états : brûlé, non brûlé, non observé.
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
- **`composite.py` de flood-monitoring :** il fabrique une composition colorée radar, pas une médiane.

**Encore à comparer :**

- stackstac ou odc-stac contre les extraits d'eo-stac : lecture à la demande, mais contraire à la préférence pour eo-stac.
- Pilote PMTiles de GDAL contre tippecanoe, à tester avec `platform-publish validate`.
- VIIRS seul contre VIIRS et MODIS.
- BD Forêt V2 contre une couche européenne de couvert arboré.
- Plus tard : déplacer GDAL et OTB d'eo-pipe derrière des options, avec un import différé.

## Décisions et étapes suivantes

### Décisions prises

- Noyau de traitement en rioxarray ; eo-stac pour la recherche et le téléchargement.
- eo-pipe retiré du pilote, contrairement à la consigne initiale : `rasterio.features` doit remplacer `SieveStep` et la polygonisation (non testé). Les limites restent documentées.
- Zone : Nouvelle-Aquitaine et Pays de la Loire, année 2025.
- Source des points : l'export EFFIS `hs_modis` 2025 déjà téléchargé, à copier dans `~/Documents/Projects/data/fire-monitoring` pour figer la version utilisée. VIIRS ne sera comparé que si les feux proches de 4 ha manquent.
- Seuil de surface : 4 ha.
- Regroupement : composantes connexes, moins de 2 km et 7 jours d'écart au plus, plus un filtre des détections répétées au même endroit (seuil à fixer d'après l'étape 1 du pilote).
- Dates : `date_start` est le premier point et sert de date de l'événement. `last_detection` est le dernier point : c'est la dernière détection, pas la fin réelle du feu.
- Image avant : recherche de `date_start − 60 j` à `date_start − 1 j`, les 3 dates les plus claires, en privilégiant les plus récentes.
- Image après : recherche de `last_detection + 3 j` à `last_detection + 45 j`, les 3 premières dates claires. Sans image exploitable, l'événement prend l'état `no_clear_post`.
- Masque forêt : BD Forêt V2 de l'IGN, sous réserve de vérifier sa disponibilité, sa couverture et sa licence.
- Carte annuelle sans filtre de date.
- `FireEvent` comme nom de l'unité du pipeline.

### Déroulé du pilote

1. Sans télécharger de Sentinel-2, passer FIRMS, le filtre forêt et le regroupement sur toute la zone. Mesurer le nombre d'événements, leur durée et leur emprise, puis décider de continuer ou non selon le budget disque et temps.
2. Traiter de bout en bout cinq événements choisis : un grand feu, un feu proche de 4 ha, une coupe rase avec point chaud, un cas nuageux, un cas proche de 0° de longitude. Ajouter une longue chaîne si l'étape 1 en trouve une.
3. Publier sous un identifiant temporaire, mesurer temps, volume et échecs par événement, contrôler à la main.
4. Lancer ensuite les 17 départements, un événement à la fois.

### À trancher

1. **Couverture de l'image après :** quel taux de pixels valides exiger ? Proposition : 80 %.
2. **Filtres :** quelle marge autour des points (proposition : 750 m, points du même événement), quel seuil sur `confidence` et quelle règle de forêt ?
3. **Seuil dNBR :** proposition de 0,27. Faut-il le faire varier selon le type de forêt ?
4. **Fusion :** quelle règle de dédoublonnage, et quelle date garde un polygone fusionné ? Faut-il aussi arrêter la fenêtre « après » au début d'un événement voisin ?
5. **Catalogue :** Planetary Computer (utilisé par flood-monitoring) ou Earth Search ?
6. **Masque forêt :** quelles classes et quel millésime ?
7. **Événements non évalués :** faut-il les afficher sur la carte ou seulement dans le rapport ? Le registre est-il publié ?
8. **Limites d'année :** quelle année possède un feu à cheval sur deux années ? Les feux de janvier et de décembre demandent aussi des images de 2024 et 2026.
9. **Stockage :** faut-il conserver les composites ? Le budget disque sera connu après l'étape 1.
10. **Identifiant de publication :** les slugs ne sont jamais réutilisés. Un par année, ou un seul pour plusieurs années plus tard ?

Questions d'implémentation à régler pendant la conception :

- agrandir la zone quand une brûlure touche son bord ;
- garder dask ou non : d'après les emprises mesurées (4 km au plus), il semble inutile par événement ; à confirmer après l'étape 1 ;
- garder uniquement le NBR ou aussi d'autres bandes pour les futurs modèles (l'interface proposée reçoit les bandes) ;
- choisir la liste des états d'un événement.
