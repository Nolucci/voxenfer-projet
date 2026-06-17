# service-classements (G5)

Service feuille du projet Voxenfer : enregistre les points marqués par les
joueurs et expose le classement. Pas d'appel sortant vers un autre service.

Ce dossier est **autonome** : tout ce qu'il faut pour lancer et tester
`service-classements` (code, contrat, harnais Docker, tests) est ici. On
peut cloner uniquement `service-classements/` sans dépendre du reste du
dépôt.

## Lancement

### En local

```bash
pip install -r requirements.txt
python app.py
curl localhost:5000/health
```

### Via docker compose (depuis ce dossier)

Le service écoute toujours sur le port **5000 en interne** (imposé par le
contrat) ; il n'est jamais exposé directement, on passe par la **gateway**
sur `:8080`.

```bash
docker compose up --build
curl http://localhost:8080/classements/health
curl http://localhost:8080/classements/classement
```

> Le `docker-compose.yml` et le `Caddyfile` de ce dossier sont un
> **harnais de test local pour G5** (gateway + service-classements
> seulement), en attendant le compose officiel de **G1** qui assemblera
> tous les `service-*/`. Le bloc `service-classements` y est écrit pour
> être recopié tel quel dans le compose final de G1 — aucune divergence
> de port, de nom de variable ou de secret JWT par rapport au contrat, donc
> aucun impact sur l'intégration avec les autres groupes.

## Bloc docker-compose à fournir à G1

À recopier tel quel dans le `docker-compose.yml` commun (sous `services:`),
avec un volume `classements-data` déclaré dans `volumes:` :

```yaml
# G5 : Service classements
service-classements:
  build: ./service-classements
  environment:
    <<: *jwt-secret
    DB_PATH: /data/classements.db
  volumes:
    - classements-data:/data
```

Et dans `volumes:` (au même niveau que les autres) :

```yaml
volumes:
  classements-data:
```

Pas de dépendance sortante (`depends_on`) : `service-classements` est un
service feuille, il n'appelle aucun autre service.

## Routes

| Méthode | Route | Auth | Comportement |
|--------:|:------|:-----|:-------------|
| GET | `/classement` | ouverte | Liste triée par points décroissants `[{pseudo, points}]` ; supporte `?page=&taille=` (bonus, pagination) |
| GET | `/classement/top/<n>` | ouverte | Les `n` premiers du classement |
| GET | `/classement/periode/<periode>` | ouverte | **Bonus** : `periode` = `jour` ou `semaine` ; points gagnés sur la période `[{pseudo, points}]` ; **400** si `periode` invalide |
| GET | `/badges/<pseudo>` | ouverte | **Bonus** : badges obtenus par seuil de points `{pseudo, points, badges:[{seuil, nom}]}` ; **404** si pseudo inconnu |
| GET | `/scores/<pseudo>` | ouverte | `{pseudo, points}` ; **404** si le pseudo est inconnu |
| POST | `/scores` | JWT requis | `{pseudo, points}` → ajoute `points` au score existant (crée la ligne si absente), et enregistre le gain dans l'historique (table `gains`) |

Vu de l'extérieur (gateway Caddy), ces routes sont préfixées par `/classements`
(ex. `POST /classements/scores`).

### Décisions documentées

- `GET /scores/<pseudo>` renvoie **404** si le pseudo n'a aucun score enregistré
  (plutôt que `{pseudo, points: 0}`).
- `POST /scores` **cumule** les points : rejouer le même pseudo additionne au
  total existant, ne le remplace pas.
- Pas de vérification de l'existence du pseudo auprès de `service-comptes` :
  on fait confiance au payload / au jeton.

### Codes d'erreur

- `POST /scores` sans jeton → `401`.
- `POST /scores` avec `pseudo` absent/invalide ou `points` non entier/négatif → `400`.
- `GET /scores/<inconnu>` → `404`.
- Jamais de `500` : toute entrée mal formée est interceptée avant l'ORM.

## Exemples curl

```bash
# Ajouter des points (jeton requis)
curl -X POST localhost:8080/classements/scores \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pseudo":"maxime","points":10}'

# Rejouer -> le total cumule (20 si on rejoue le même +10)
curl -X POST localhost:8080/classements/scores \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pseudo":"maxime","points":10}'

# Score d'un joueur
curl localhost:8080/classements/scores/maxime

# Classement trié
curl localhost:8080/classements/classement

# Top 3
curl localhost:8080/classements/classement/top/3

# Sans jeton -> 401
curl -X POST localhost:8080/classements/scores \
  -H "Content-Type: application/json" \
  -d '{"pseudo":"maxime","points":10}'

# Score d'un inconnu -> 404
curl localhost:8080/classements/scores/inconnu

# Classement paginé (bonus)
curl "localhost:8080/classements/classement?page=1&taille=10"

# Classement sur les dernières 24h / 7 jours (bonus)
curl localhost:8080/classements/classement/periode/jour
curl localhost:8080/classements/classement/periode/semaine

# Badges d'un joueur (bonus)
curl localhost:8080/classements/badges/maxime
```

## Tests

### Tests unitaires (pytest)

```bash
pip install -r requirements-dev.txt
pytest
```

### Tests bout-en-bout (curl, via la gateway)

`test.sh` exécute en série les commandes `curl` ci-dessus contre la stack
qui tourne (gateway + service-classements) et vérifie les codes HTTP :

```bash
docker compose up --build -d
./test.sh
# ou contre une autre base :
./test.sh http://localhost:8080
```

## Bonus implémentés

- **Classements par période** (`/classement/periode/<jour|semaine>`) : s'appuie sur une
  table d'historique `gains` (un enregistrement horodaté par `POST /scores`), agrégée par
  fenêtre glissante.
- **Pagination** sur `/classement` via les paramètres `page` et `taille` (le comportement
  par défaut, sans paramètres, reste inchangé : tout le classement).
- **Badges** (`/badges/<pseudo>`) : seuils fixes (bronze 100, argent 500, or 1000 points).
