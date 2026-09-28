# Dossier de voyage collaboratif (trip-planner)

Page web collaborative pour organiser un voyage entre amis : chacun indique ses disponibilités, puis le groupe propose, compare et vote des voyages (vol + logement) et des activités.

Tout tient dans un seul fichier, `index.html` (HTML, CSS et JavaScript sans build ni dépendance npm).

## Comment l'app est hébergée

L'app vit comme un **Artifact Claude** publié en lien public. Le dépôt GitHub sert de sauvegarde et d'historique du code source.

- Lien de l'app : https://claude.ai/artifact/LdRHFRQBhYpatV3MACohk9
- Les données partagées (voyages, votes, disponibilités...) sont stockées via la capacité `db` des Artifacts (`claude.use('db')`), déclarée à la publication (`capabilities: {db: {}}`).
- La page n'est **pas** hébergeable telle quelle sur GitHub Pages ou un autre hébergeur statique : sans le runtime Artifact, `claude.use` n'existe pas, la base est `null` et rien ne se synchronise entre les utilisateurs.

## Fonctionnement en deux phases

Le champ `phase` du document `trip/info` pilote ce que voient tous les visiteurs :

| Phase | Ce que voient les amis |
| --- | --- |
| `availability` | Calendrier de disponibilités (janvier à mars 2027 par défaut) |
| `trips` (défaut si absent) | Carte, voyages proposés, activités, vote |

- Seul l'organisateur (prénom saisi = "Clément", insensible à la casse et aux accents) peut basculer de phase, via les boutons "Ouvrir la page voyages" / "Refermer et revenir aux disponibilités" et le bouton de l'en-tête.
- La phase est stockée en base, donc elle est commune à tous et persiste : un ami qui rouvre l'app après l'ouverture retombe directement sur la page voyages.

## Fonctionnalités

- Identification par prénom (stocké en `localStorage`) pour attribuer ajouts et votes.
- Calendrier de disponibilités : clic ou clic-glisser (souris et tactile) pour sélectionner des plages. Rappel "modifications non enregistrées" tant que la sélection n'est pas sauvegardée. Liste des personnes ayant répondu.
- Tuile "Meilleures dates" : jusqu'à 3 périodes non chevauchantes, durée minimale de 6 jours, extension tant que le même groupe reste disponible, classement par nombre de personnes libres tous les jours de la période. Un clic sur une proposition filtre les voyages dont les dates chevauchent la période.
- Formulaire "Ajouter un voyage" : un vol (trajet, compagnie, dates, prix, durée du vol, temps de route jusqu'au logement) et un logement (lien, titre, lieu, prix, capacité, chambres, avis, piscine/plage/chef...).
- Deux vues des voyages : Normale (carte détaillée) et Mosaïque (tuiles compactes, 2 colonnes sur mobile ; le bandeau vol apparaît dans "Voir tous les détails").
- Filtre de capacité automatique selon le nombre de voyageurs, tri par score ou récence, filtre "hypés".
- Votes (pouce haut/bas) et commentaires sur chaque voyage et activité.
- Grande carte "Destinations" en SVG (géométrie Natural Earth pré-calculée) et mini-carte par logement.
- Thème clair/sombre automatique, mise en page pensée d'abord pour téléphone.

## Droits d'édition

- Modifier le prix et supprimer un voyage ou une activité : uniquement le créateur de la carte ou l'organisateur.
- Voter et commenter : tout le monde.
- Il s'agit de vérifications côté client basées sur le prénom déclaré, pas d'une vraie authentification. C'est suffisant entre amis, pas pour des données sensibles.

## Modèle de données (capacité `db`)

| Chemin | Contenu principal |
| --- | --- |
| `trip/info` | `name`, `people`, `phase`, optionnels `availWindowStart` / `availWindowEnd` |
| `listings/{id}` | destination, lien, titre, lieu, prix, capacité, dates, équipements, `addedBy`, `createdAt`... |
| `flights/{id}` | destination, trajet, compagnie, dates, prix, lien, `duration`, `drivingTime`, `addedBy` |
| `activities/{id}` | destination, titre, lieu, prix, lien, notes, `addedBy` |
| `destinations/{id}` | destinations personnalisées (`label`, `emoji`) |
| `votes/{type__id__voter}` | `itemType`, `itemId`, `voter`, `value` (1, -1 ou 0), `comment` |
| `availabilities/{prénom}` | `name`, `dates` (liste ISO `YYYY-MM-DD`), `updatedAt` |

Chaque vol est rattaché aux logements par la clé de destination (le vol le moins cher de la destination est affiché).

## Mettre à jour l'app

1. Modifier `index.html`.
2. Republier l'Artifact avec l'outil Artifact de Claude Code en passant l'URL existante (`url`) pour garder le même lien. Ne pas repasser `capabilities` : la déclaration `db` est conservée d'une version à l'autre.
3. Commit et push sur `main`.

## Points d'attention et décisions passées

- **Photos** : pas d'upload d'images. La capacité `assets` rend un Artifact interne à l'organisation et non partageable publiquement, ce qui casserait l'accès des amis. Les fiches n'ont donc pas de photo.
- **Google Maps** : l'iframe d'embed est bloquée dans le contexte sandboxé de l'Artifact ("Ce contenu est bloqué"). Les mini-cartes essaient l'embed OpenStreetMap avec les coordonnées de `MAP_DATA.placeLL`, puis retombent sur une mini-carte SVG autonome (`MAP_DATA.places`).
- **Précision des lieux** : la géolocalisation repose sur une table embarquée d'environ 100 villes et pays, pas sur un vrai géocodage. Un lieu absent de la table retombe sur la destination générale.
- **Classes CSS** : `.trip-title` est réservée au grand titre de page ; les titres de cartes utilisent `.card-title`. Ne pas les fusionner (un conflit de cascade avait fait gonfler les titres de cartes sur mobile).
- **Fenêtre de disponibilités** : fixée dans `availWindow()` (2027-01-01 à 2027-03-31). Elle peut être surchargée par `availWindowStart` / `availWindowEnd` dans `trip/info`, mais aucune interface ne les modifie.

## Structure du dépôt

```
index.html   Application complète (HTML, CSS, JS, données cartographiques embarquées)
README.md    Ce fichier
```
