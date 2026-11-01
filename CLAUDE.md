# Parameteo

Application web d'aide à la décision pour le **parapente** : émagramme AROME interactif, plafonds thermiques, sites de vol FFVL notés et balises de vent. Public : pilotes de vol libre francophones. **Toute l'interface et les messages sont en français.**

Site publié : https://raphaelvdv.github.io/Parameteo/ (GitHub Pages, branche `main`, dossier racine). Pousser sur `main` met le site en ligne en 1 à 2 minutes. Il n'y a ni build ni dépendance npm : on édite les fichiers tels quels.

## Fichiers

| Fichier | Rôle |
|---|---|
| `index.html` | Toute l'application : HTML, CSS et JavaScript dans un seul fichier (~1 900 lignes) |
| `logo.svg` | Logo « Aube » (voile fuchsia, relief bleu à courbes de niveau, cumulus, ascendance, isothermes à 45°) |
| `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` | Icônes générées à partir de `logo.svg` (à régénérer si le logo change) |
| `manifest.webmanifest` | Installation sur l'écran d'accueil (nom, couleurs, icônes) |
| `.nojekyll` | Sert les fichiers tels quels sur Pages |

Le logo est aussi **inclus en ligne** dans l'en-tête de `index.html` (`<div class="brand">`), avec des identifiants préfixés `lg-` : le mettre à jour aux deux endroits.

## Organisation du script (`index.html`)

Sections repérées par des commentaires `// ---------- … ----------`, dans cet ordre :

1. **Constantes et thermodynamique** : `es`, `mixr`, `theta`, `moistStep` (pseudo-adiabatique en RK4), `lclOf` (NCA de Bolton), `thetaW`, `thetaE`.
2. **Exemple fictif** (`EXAMPLE`) : profil affiché au démarrage et dans le didacticiel, toujours signalé comme fictif.
3. **Préparation du sondage** (`prepare`) et interpolation en ln p (`interpS`, `envT`, `envTd`, `envZ`, `windAtZ`, `pAtZ`).
4. **Analyse** (`analyze`) : particule (surface, couche mélangée, plus instable), CAPE/CIN en température virtuelle, NCA/NCL/NE, indices (LI, K, TT), eau précipitable, isotherme 0 °C, cisaillement 0–6 km, **plafond sol et plafond relief** (`thermalFrom`) avec diagnostic « couche brassée » / « couvercle sous les crêtes ».
5. **Rendu** (`draw`) : émagramme 761 sur canvas, isothermes inclinées à 45°, couleurs lues dans les variables CSS.
6. **Panneaux**, **Import Open-Meteo / texte** (`loadOpenMeteo`, `sndFromHour`, `parseText`), **Événements**.
7. **Didacticiel** (`TUT`) et **Lecture commentée** (`READ`) : même moteur d'étapes (`renderStep`, `curSteps`), textes de la lecture générés à partir des données.
8. **Sites de vol** : terrains FFVL (`ffvlTerrains`, `ffvlSitesAround`), OpenStreetMap / ParaglidingEarth en option (`getExtraSites`), prévision par paquets (`fetchForecasts`), analyse par site (`analyzeSpot`), **ensoleillement des pentes** (`sunPos`, `slopeSun`), **cohérence du secteur** (`loadSector`, `sectorEval`, `windAtAltD`), **notation** (`RULES`, `scoreSpot`), carte Leaflet et classement en volet (`renderSpots`, `setSide`), **clic n'importe où sur la carte** (`clickPoint`, `pointHtml` ; `loadSpot("point")` charge son sondage).
9. **Balises** : FFVL et Pioupiou (`fetchFFVL`, `fetchPioupiou`, `beaconFor`), comparaison prévu / mesuré (`compareBeacon`, `cmpHtml`).
10. **Paramètres** (`rulesHtml`) : fenêtre en lecture seule générée depuis `RULES`.

## Règles métier à respecter

- **Les notes (décollage et thermique, sur 100) reposent uniquement sur la prévision.** Les balises ne servent qu'à la comparaison prévu / mesuré, active à moins de 2 h de l'heure choisie.
- **Tous les seuils de notation sont dans l'objet `RULES`.** Modifier un seuil = modifier `RULES` uniquement : `scoreSpot`, `compareBeacon` et la fenêtre « Paramètres » le lisent. Ne pas écrire de seuil en dur ailleurs. L'utilisateur ne peut pas modifier les règles depuis l'interface (choix voulu).
- Les textes affichés doivent rester prudents : la note est indicative et ne remplace pas les bulletins officiels, les balises ni la fiche du site.
- **Chauffe des pentes** : le réglage « Chauffe max. » vaut pour un soleil perpendiculaire ; pour chaque déco, elle est multipliée par l'ensoleillement de la pente (orientation moyenne du déco, pente supposée de 30°, position du soleil à l'heure choisie) et réduite par la nébulosité totale (`cloud_cover`). Un déco ouvert à 5 orientations ou plus est traité comme un sommet.
- **Cohérence du secteur** : pour les 10 meilleurs décos, 4 points voisins à 2,5 km (requête légère : vent, pluie, nuages bas) ; la variabilité rend la note plus prudente (retrait plafonné à 30 par note). Les autres décos affichent « non évaluée ».
- Un site FFVL **interdit** est noté 0 ; afficher statut, restrictions, avertissements et alertes FFVL dans les fiches.

## Sources de données (appelées directement depuis le navigateur)

| Donnée | Adresse | Remarques |
|---|---|---|
| Prévisions | `api.open-meteo.com/v1/forecast` (modèles `meteofrance_arome_france`, `meteofrance_arpege_europe`, `meteofrance_seamless`), `/v1/elevation` | Niveaux de pression 1000 à 100 hPa, vent en nœuds (`wind_speed_unit=kn`) |
| Terrains FFVL | `https://www.data.gouv.fr/api/1/datasets/r/4375c95d-cf26-4e71-b72d-2e09be5a0e36` | `terrains.json`, ~3,7 Mo, mis à jour chaque jour ; gardé 24 h en `localStorage` |
| Balises FFVL | `…/datasets/r/8d795154-9145-4ab2-99de-6768a36f31ea` | `balises.json`, gardé 24 h |
| Relevés FFVL | `…/datasets/r/047b2ab9-627f-4e5f-9d68-5b948433c61d` | `releves.json`, toutes les 5 min ; vitesses en km/h |
| Balises Pioupiou | `https://api.pioupiou.fr/v1/live/all` | Les stations OpenWindMap ne sont pas dans les relevés FFVL |
| Sites (option) | Overpass (4 miroirs), ParaglidingEarth | Gardés 7 jours |
| Fonds de carte | OpenTopoMap, OpenStreetMap, Esri | Leaflet 1.9.4 via cdnjs |

L'API `data.ffvl.fr` exige une clé : **ne pas l'utiliser**, les mêmes données sont en open data sur data.gouv.fr (le lien `/api/1/datasets/r/<id>` redirige vers le fichier le plus récent et autorise les appels depuis le navigateur).

Pour tester depuis un environnement cloud, autoriser dans le réseau de l'environnement : `data.ffvl.fr`, `www.data.gouv.fr`, `static.data.gouv.fr`. Open-Meteo, Overpass et les tuiles n'étaient pas joignables depuis l'environnement de développement : les tester avec des réponses simulées (Playwright, `page.route`).

## Design

- **Thème « Aube »**, dérivé du logo : en-tête en dégradé abricot (`--band-top` → `--band`), commandes en bleu relief (`--accent`), **plafonds en fuchsia** (`--hi`), isothermes orangées (`--iso`). Mode sombre « crépuscule » via `prefers-color-scheme` et `[data-theme="dark"]`.
- Toutes les couleurs passent par les variables du bloc `:root` (et leurs deux redéfinitions sombres). Pas de couleur en dur dans les composants.
- Polices : **Archivo Narrow** (titres, étiquettes, boutons), **Archivo** (texte), **JetBrains Mono** (chiffres).
- Conventions de l'émagramme à conserver : courbe d'état rouge, point de rosée bleu, particule en tirets, notes vert ≥ 70 / orange 40–69 / rouge < 40.
- La page doit fonctionner sur téléphone (≈ 400 px, aucun défilement horizontal) et en plein soleil (contrastes suffisants).

## Vérifier une modification

Chromium et Playwright sont disponibles. Charger `file:///…/index.html`, simuler les réponses réseau avec `page.route`, vérifier l'absence d'erreur (`pageerror`) et faire une capture en clair et en sombre, puis une à 400 px de large.
