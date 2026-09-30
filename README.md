# Dossier de voyage collaboratif (trip-planner)

Page web collaborative pour organiser un voyage entre amis : chacun se connecte avec Google, indique ses disponibilités, puis le groupe propose, compare et vote des voyages (vol + logement) et des activités.

Tout tient dans un seul fichier, `index.html` (HTML, CSS et JavaScript sans build ni dépendance npm, plus un `<script type="module">` en fin de page pour Firebase).

## Lien de l'app et hébergement

- **Lien à partager avec le groupe : https://clementg97-beep.github.io/trip-planner/**
- Hébergée sur **GitHub Pages**, servie directement depuis la racine de la branche `main` de ce dépôt (public, requis pour Pages gratuit).
- Les données partagées (voyages, votes, disponibilités, qui est connecté...) vivent dans **Firebase Firestore**, projet `trip-planner-f3e79`.
- L'identité vient de **Firebase Authentication** (connexion Google) — pas de compte Claude requis pour les amis.

Il existe aussi un Artifact Claude historique (https://claude.ai/artifact/LdRHFRQBhYpatV3MACohk9) gardé en synchronisation pour référence, mais **il ne fonctionne plus pour la connexion/les données** : la politique de sécurité (CSP) des Artifacts bloque les scripts chargés depuis `gstatic.com`, donc Firebase ne peut pas s'y initialiser. Ne pas partager ce lien aux amis ; republier dessus n'est plus nécessaire mais reste inoffensif.

## Pourquoi Firebase et pas la capacité `db` de Claude

Le projet utilisait au départ la capacité `db` des Artifacts Claude (`claude.use('db')`). Ça marchait, mais obligeait chaque ami à avoir un compte Claude pour lire/écrire les données partagées. Migré vers Firebase (Firestore + connexion Google) pour que n'importe qui avec un compte Google gratuit puisse participer, et pour pouvoir héberger l'app sur un lien public classique (GitHub Pages) plutôt qu'un lien Artifact.

La migration n'a touché que la couche connexion/stockage : un `<script type="module">` en fin de fichier initialise Firebase et expose sur `window` un adaptateur (`__fbDbReady`, `__fbSignIn`, `__fbSignOut`, événement `fb-auth-changed`) qui reproduit **exactement** la forme de l'ancien objet `db` (`doc()/collection()/set()/update()/delete()/onSnapshot()`), donc aucun appel `db.doc()/db.collection()` du reste du code n'a changé.

## Fonctionnement en deux phases

Le champ `phase` du document `trip/info` pilote ce que voient tous les visiteurs :

| Phase | Ce que voient les amis |
| --- | --- |
| `availability` (défaut si le champ n'existe pas encore) | Calendrier de disponibilités (janvier à mars 2027 par défaut) |
| `trips` | Carte, voyages proposés, activités, vote |

- Seul l'organisateur (premier mot du nom Google normalisé = "clement", insensible aux accents/majuscules — ex. "Clément Gustin" passe) peut basculer de phase : bouton discret dans l'en-tête, gros bouton "🚀 Ouvrir la page voyages pour tout le monde" sur la page dispos, bouton "🔒 Refermer et revenir aux disponibilités" sur la page voyages.
- Pour tout le monde d'autre, ces contrôles sont visibles mais désactivés ("🔒 Réservé à l'organisateur"), plus un encart "🔒 En attente que Clément ouvre..." sur la page dispos.
- La phase est stockée en base, donc commune à tous et persistante : un ami qui rouvre l'app après l'ouverture retombe directement sur la page voyages.
- Un nouveau voyage (base neuve, champ `phase` jamais écrit) démarre sur la page disponibilités — pas la page voyages.

## Fonctionnalités

- Connexion Google obligatoire (Firebase Auth) pour identifier les ajouts, votes et commentaires. Le nom affiché est le nom Google complet.
- Calendrier de disponibilités : clic ou clic-glisser (souris et tactile) pour sélectionner des plages. Rappel "modifications non enregistrées" tant que la sélection n'est pas sauvegardée. Liste de qui a répondu **et** de qui est connecté mais n'a pas encore répondu.
- Tuile "Meilleures dates" : jusqu'à 3 périodes non chevauchantes, durée minimale de 6 jours, extension tant que le même groupe reste disponible, classement par nombre de personnes libres tous les jours de la période. Cliquer une proposition filtre les voyages dont les dates chevauchent la période.
- Formulaire "Ajouter un voyage" : un vol (trajet, compagnie, dates, prix, durée du vol, temps de route jusqu'au logement) et un logement (lien, titre, lieu, prix, capacité, chambres, avis, piscine/plage/chef...). Pas de photo.
- Badge doré "total par personne" (logement + vol) en haut à droite de chaque carte, et tri "🐀 Moins cher" basé sur ce même chiffre.
- Deux vues, pour les voyages **et** les activités : Normale (carte détaillée) et Mosaïque (tuiles compactes, 2 colonnes sur mobile).
- Filtre de capacité automatique selon le nombre de voyageurs, tri par score/récence/prix, filtre "hypés".
- Votes (pouce haut/bas) et commentaires sur chaque voyage et activité ; chaque commentaire est modifiable/supprimable par son auteur (ou l'organisateur).
- Édition du prix et suppression d'une fiche : réservées au créateur de la fiche ou à l'organisateur.
- Grande carte "Destinations" en SVG (géométrie Natural Earth pré-calculée, ville + pays uniquement) et mini-carte par logement (OpenStreetMap avec repli SVG).
- Thème clair/sombre automatique, mise en page pensée d'abord pour téléphone.

## Droits et identité — limites à connaître

- L'organisateur est déterminé par le **nom du compte Google connecté**, pas par un vrai rôle en base. Quelqu'un qui nommerait son compte Google "Clément" passerait la vérification — acceptable entre amis, pas une vraie sécurité.
- `canEdit()` (modifier le prix, supprimer) compare le nom affiché au champ `addedBy` de la fiche. Pas de vraie ACL côté serveur au-delà des règles Firestore (voir plus bas), qui ne font qu'exiger d'être connecté.

## Modèle de données (Firestore)

| Collection | Contenu principal |
| --- | --- |
| `trip/info` (doc unique) | `name`, `people`, `phase` |
| `listings/{id}` | destination, lien, titre, lieu, prix, capacité, dates, équipements, `addedBy`, `createdAt`... |
| `flights/{id}` | destination, trajet, compagnie, dates, prix, lien, `duration`, `drivingTime`, `addedBy` |
| `activities/{id}` | destination, titre, lieu, prix, lien, notes, `addedBy` |
| `destinations/{id}` | destinations personnalisées (`label`, `emoji`) — les 3 destinations de base (Samaná, Cartagène, Monterrico) sont codées en dur dans le JS, pas dans cette collection |
| `votes/{type__id__voter}` | `itemType`, `itemId`, `voter`, `value` (1, -1 ou 0), `comment` |
| `availabilities/{nom}` | `name`, `dates` (liste ISO `YYYY-MM-DD`), `updatedAt` |
| `users/{uid}` | `name`, `lastSeenAt` — un doc par personne qui s'est déjà connectée, écrit dès la connexion même si elle ne fait rien d'autre ; sert à afficher qui n'a pas encore répondu |

Chaque vol est rattaché aux logements par la clé de destination (le vol le moins cher de la destination est affiché).

**Piège Firestore à connaître** : `updateDoc()` (la vraie fonction Firestore) échoue si le document n'existe pas encore — contrairement à l'ancienne capacité `db` de Claude, plus tolérante. L'adaptateur contourne ça : `db.doc(x).update(patch)` appelle en réalité `setDoc(ref, patch, {merge:true})`, qui crée le document au besoin. Ne pas revenir à un `updateDoc()` direct sans réintroduire ce bug (vécu en prod : les boutons de bascule de phase ne faisaient plus rien sur une base neuve).

## Console Firebase — réglages à ne pas perdre

- **Authentication → Settings → Authorized domains** : doit contenir `clementg97-beep.github.io` (sinon la connexion Google échoue avec une erreur de domaine non autorisé).
- **Firestore Database → Règles** : doit être publié avec au minimum
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /{document=**} {
        allow read, write: if request.auth != null;
      }
    }
  }
  ```
  (accès réservé aux utilisateurs connectés, pas de restriction par propriétaire — cohérent avec le modèle de droits côté client décrit plus haut).
- La config Firebase (`apiKey`, `projectId`...) est **publique par design** dans `index.html` — ce n'est pas un secret, la sécurité vient des règles Firestore ci-dessus, pas de la confidentialité de cette config.

## Mettre à jour l'app

1. Modifier `index.html`.
2. Republier l'Artifact avec l'outil Artifact de Claude Code en passant l'URL existante (`url`) — optionnel/best-effort maintenant que ce lien ne sert plus aux amis, mais garde les deux copies en phase.
3. Commit et push sur `main` : GitHub Pages se met à jour automatiquement en ~1 minute, sans interruption pour qui a déjà la page ouverte (fichiers statiques + base séparée, pas de redémarrage de service).
4. Un changement de code seul ne touche jamais aux données Firestore existantes ; une vraie migration de données se fait à part (voir historique de commits pour l'exemple du reset avant lancement).

## Points d'attention et décisions passées

- **Photos** : pas d'upload d'images. Ni la capacité `assets` de Claude (rendait l'Artifact interne à l'organisation) ni Firebase Storage n'ont été mis en place. Les fiches n'ont donc pas de photo, juste une mini-carte.
- **Google Maps embed** : testé puis abandonné — bloqué dans le contexte sandboxé de l'ancien Artifact ("Ce contenu est bloqué"). Les mini-cartes essaient l'embed OpenStreetMap (coordonnées de `MAP_DATA.placeLL`), puis retombent sur une mini-carte SVG autonome (`MAP_DATA.places`) si aucune coordonnée n'est trouvée.
- **Précision des lieux** : repose sur une table embarquée d'une centaine de villes/pays, pas un vrai géocodage. Un lieu absent de la table retombe sur la destination générale.
- **Classes CSS** : `.trip-title` est réservée au grand titre de page ; les titres de cartes utilisent `.card-title`. Ne pas les fusionner (un conflit de cascade avait fait gonfler les titres de cartes sur mobile).
- **Fenêtre de disponibilités** : fixée dans `availWindow()` (2027-01-01 à 2027-03-31), aucune interface ne permet de la changer actuellement.
- **Mosaïque** : la classe `.mosaic` sur le conteneur suffit à appliquer le style compact — `#activityGrid` et `#listingGrid` partagent déjà la classe `.trip-list`, donc aucune règle CSS dupliquée entre les deux.

## Structure du dépôt

```
index.html   Application complète (HTML, CSS, JS, module Firebase, données cartographiques embarquées)
README.md    Ce fichier
```
