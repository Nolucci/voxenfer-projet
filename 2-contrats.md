---
titre: "Projet Voxenfer - Contrats inter-services"
sous-titre: L'interface partagée par tous les groupes
sous-sous-titre: 3iL Ing - I1 apprentissage
auteur: Philippe \textsc{Roussille}
annee: true
rendu-logo: 3il
---

# À quoi sert ce document

En micro-services, chacun code dans son coin. Pour que les services se **comprennent**, il faut un **contrat commun** fixé **à l'avance** : routes, champs JSON, format du jeton, codes d'erreur.

Ce document est ce contrat, mais **volontairement réduit à sa base** : le **minimum** que les autres groupes supposent présent chez vous. **C'est à vous de le compléter.** Chaque équipe y précise les routes de **son** service (routes étoffées, champs JSON exacts), et on valide l'ensemble **tous ensemble** en début de séance. Une seule règle : on **étoffe sans jamais casser** les signatures de base ci-dessous. G1 (plateforme) tient la version de référence et arbitre les ajustements.

# Règles communes (TOUS les services)

- **Flask**, une **base SQLite** par service, un **Dockerfile** par service.
- Tous écoutent sur le **port 5000** (`app.run(host="0.0.0.0", port=5000)`).
- **Lectures ouvertes, écritures protégées** par JWT (`@require_jwt`).
- Chacun expose **`/health`** et **`/metrics`** (repris du starter).
- Réponses **toujours en JSON**, avec le **bon code HTTP** :

| Code | Sens |
|------|------|
| 200 / 201 | OK / créé |
| 400 | requête mal formée (champ manquant) |
| 401 / 403 | non authentifié / rôle insuffisant |
| 404 | ressource inconnue |
| 409 | conflit (doublon, solde insuffisant, complet...) |
| 503 | un service dont je dépends est injoignable |

> **Exception : `service-monde` (G1)** n'a pas de base SQLite : il lit, **en lecture seule**, la base **PostgreSQL** d'un serveur Luanti. C'est le seul service branché dessus.

# Le jeton JWT

- **Secret partagé** `JWT_SECRET`, **identique** pour tous (fixé dans `docker-compose.yml`).
- **Payload** : l'identité est le `pseudo`, avec une **liste** de rôles.

```json
{ "pseudo": "maxime", "roles": ["joueur"] }
```

- **Rôles** : `joueur` < `moderateur` < `admin`. L'`admin` gère les rôles et le catalogue ; ce **n'est pas un joueur** du jeu (compte de service). `require_role` teste l'appartenance à la liste.
- Émis par **service-comptes** au `/login`, vérifié par **tous** (`auth.py`), transmis en `Authorization: Bearer <jeton>`.
- Le **mod** (côté enseignant) porte un jeton **admin** pour acquitter les files d'actions.

# Routage par la gateway (Caddy)

La gateway expose **`http://localhost:8080`**, route par **préfixe** et **retire le préfixe** avant de transmettre. Donc **à l'intérieur, vos routes n'ont PAS le préfixe**.

| URL publique | Service interne | Reçoit |
|---|---|---|
| `/comptes/...` | `service-comptes:5000` | `/...` |
| `/economie/...` | `service-economie:5000` | `/...` |
| `/boutique/...` | `service-boutique:5000` | `/...` |
| `/classements/...` | `service-classements:5000` | `/...` |
| `/moderation/...` | `service-moderation:5000` | `/...` |
| `/evenements/...` | `service-evenements:5000` | `/...` |
| `/monde/...` | `service-monde:5000` | `/...` |

Exemple : `POST /comptes/login` arrive en `POST /login` sur `service-comptes`.

# Routes de base par service

> Le **minimum** supposé par les autres. **Chaque équipe complète le tableau de SON service** (routes étoffées, champs JSON précis) puis le partage ; les autres complètent en miroir s'ils en dépendent. Détail des niveaux dans `1-sujet.md`. La ligne *« À compléter »* sous chaque tableau est la place pour vos ajouts.

## service-comptes (G2) - identité, émet les jetons

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| POST | `/register` | - | crée un compte `{pseudo, mot_de_passe}` (mot de passe **haché**) |
| POST | `/login` | - | renvoie `{ "token": "..." }` |
| GET | `/joueurs` | - | liste des pseudos |
| GET | `/joueurs/<pseudo>` | - | `{pseudo, roles, profil}` |
| POST | `/joueurs/<pseudo>/roles` | admin | accorde un rôle `{role}` |

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

## service-economie (G3) - les pièces

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| GET | `/solde/<pseudo>` | - | `{pseudo, pieces}` |
| POST | `/crediter` | admin | `{pseudo, montant}` |
| POST | `/debiter` | jwt | `{pseudo, montant}` ; **409** si solde insuffisant |

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

## service-boutique (G4) - les objets (livrés en jeu)

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| GET | `/objets` | - | catalogue `[{id, nom, prix, item}]` (`item` = itemstring Luanti) |
| POST | `/objets` | admin | `{nom, prix, item}` |
| POST | `/acheter` | jwt | `{objet_id}` ; appelle economie `/debiter` (gérer **503**) ; crée une livraison |
| GET | `/livraisons` | serveur | livraisons en attente `[{id, type:"livrer_objet", cible, objet}]` |
| POST | `/livraisons/<id>/fait` | serveur | acquitte |

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

## service-classements (G5) - les scores

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| GET | `/classement` | - | top joueurs `[{pseudo, points}]`, triés par points décroissants |
| GET | `/scores/<pseudo>` | - | `{pseudo, points}` ; **404** si le pseudo est inconnu (décision documentée) |
| POST | `/scores` | jwt | `{pseudo, points}` (entier ≥ 0) -> **ajoute** `points` au score existant, crée la ligne si absente ; **400** si `pseudo`/`points` invalides |
| GET | `/classement/top/<n>` | - | **étoffé** : les `n` premiers du classement `[{pseudo, points}]` |
| GET | `/classement/periode/<periode>` | - | **bonus** : `periode` = `jour` ou `semaine` ; classement restreint aux points gagnés sur la période `[{pseudo, points}]` |
| GET | `/classement?page=<p>&taille=<t>` | - | **bonus** : pagination du classement complet |
| GET | `/badges/<pseudo>` | - | **bonus** : badges obtenus par seuil de points `[{seuil, nom}]` |

Décisions actées par l'équipe G5 :
- `GET /scores/<pseudo>` renvoie **404** pour un pseudo sans score (pas `{points:0}`).
- `POST /scores` **ajoute** (cumule) les points, il ne fixe pas un total.
- Aucune vérification de l'existence du pseudo auprès de `service-comptes` (confiance au payload/jeton).
- Le bonus "par période" s'appuie sur une table d'historique des gains horodatés, en plus du total cumulé dans `scores`.

*Section complétée par l'équipe G5.*

## service-moderation (G6) - signalements, bans

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| POST | `/signalements` | jwt | `{pseudo_vise, raison}` |
| GET | `/signalements` | moderateur | liste |
| POST | `/bannis` | moderateur | `{pseudo, motif, duree}` |
| GET | `/bannis` | - | liste des bannis `[{pseudo}]` (réconciliation du mod) |
| GET | `/bannis/<pseudo>` | - | `{pseudo, banni: true/false}` |

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

## service-evenements (G7) - tournois et annonces

> **La ressource porte le nom du service** : routes internes à la **racine** (`/`), pas `/evenements`. Via la gateway, la collection est `/evenements/` (**slash final** ; la gateway redirige `/evenements` vers `/evenements/`).

| Méthode | Route (interne) | Auth | Rôle |
|--------:|:------|:-----|:-----|
| GET | `/` | - | liste `[{id, nom, date, places, inscrits, statut}]` |
| POST | `/` | admin | crée `{nom, date, x, y, z, places}` |
| POST | `/<id>/inscription` | jwt | inscrit le joueur du jeton (**409** si complet) |
| GET | `/<id>/inscrits` | - | liste des pseudos |

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

## service-monde (G1) - lecture de la base Luanti

Adaptateur **en lecture seule** sur la base **PostgreSQL** de Luanti. Toutes les routes sont **ouvertes** (rien à protéger). Pas d'ORM : SQL explicite sur un schéma existant.

| Méthode | Route | Auth | Rôle |
|--------:|:------|:-----|:-----|
| GET | `/joueurs` | - | joueurs **enregistrés** `[{pseudo, derniere_connexion, privileges}]` |
| GET | `/joueurs/<pseudo>` | - | fiche d'un joueur (404 si inconnu) |
| GET | `/positions/<pseudo>` | - | dernière position/`hp`/`vivant` (404 si jamais joué) |
| GET | `/joueurs/<pseudo>/inventaire` | - | inventaire sauvegardé |

> « En ligne maintenant » n'est **pas** dans la base : `service-monde` donne le **dernier état connu**, pas le live. Le temps réel, c'est le mod.

*À compléter par l'équipe : routes étoffées et détail des champs JSON.*

# Files d'actions (interface mod <-> services)

Le **mod Luanti** (fourni, côté enseignant) ne sait faire que des appels **sortants** : c'est donc **lui qui interroge** vos services, exécute en jeu, puis **acquitte**.

- **File + ack** (livrer, téléporter, confisquer...) : le service range l'action avec un `id`, un `type` et ses paramètres. `GET /<file>` liste les actions en attente ; le mod les exécute puis `POST /<file>/<id>/fait` (anti-rejeu). Cible hors-ligne : pas d'ack, l'action est **réessayée** à la reconnexion.
- **Réconciliation** (les bannis) : pas de file, le mod relit `GET /moderation/bannis` et kicke. **Idempotent**.
- **Reflet d'état** (solde, top) : pas de file, le mod lit la route existante et l'affiche en HUD.

Table des `type` d'actions (convention figée) :

| `type` | Paramètres | Émetteur |
|---|---|---|
| `livrer_objet` | `cible`, `objet` | boutique |
| `teleporter` | `cible`, `x`, `y`, `z` | evenements |
| `confisquer` | `cible`, `objet` | moderation |
| `donner_objet` | `cible`, `objet` | moderation |

`objet` est une **itemstring Luanti** (`"default:diamond 5"`). Ajouter un type = se mettre d'accord ici.

# Dépendances (qui appelle qui)

```
boutique  --/debiter-->  economie     (seule dépendance d'appel OBLIGATOIRE)
(tous)    --vérifient le JWT émis par-->  comptes
(au choix) --GET /monde/...-->  service-monde   (facultatif)
```

Le reste communique **indirectement** via le JWT (chacun lit `pseudo` et `roles` dans le jeton). Gardez le graphe **simple** : pas de cycle ; si vous appelez un autre service, gérez son indisponibilité (**503**).

# Avant de coder (G1 anime, 15 min)

- Confirmer le `JWT_SECRET`, le payload, la hiérarchie de rôles.
- Confirmer les routes de base ci-dessus (et noter tout ajustement ici).
- Confirmer que chaque service écoute sur **5000** avec `/health`.
- Confirmer la **table des `type`** d'actions (interface avec le mod).
