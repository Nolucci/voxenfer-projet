# Groupe 5 — service-classements

## Rôles

- Julien, Nathan, Mathis — implémentation du service (`app.py`, `db.py`,
  `auth.py`), tests et documentation.

## Journal heure par heure

### Séance 1 (avant la pause)

- Lecture du contrat (`2-contrats.md`), validation des noms de
  champs (`pseudo`, `points`) et décision : `/scores/<pseudo>` renvoie 404
  pour un pseudo inconnu.
- Copie de `starter/service-template/` vers `service-classements/`,
  modèle `Score` ajouté dans `db.py`, complétion de `auth.py`
  (`require_jwt`, `require_role`).
- Implémentation de `GET /classement`, `GET /scores/<pseudo>` et
  `POST /scores` (cumul des points, validation 400). Tests `curl` en local.

### Séance 2 (après la pause)

- Début — Ajout de `GET /classement/top/<n>`. Vérification des codes
  d'erreur (401 sans jeton, 400 sur payload invalide, 404 sur pseudo inconnu).
- Milieu — Intégration dans le scénario complet (mort → score → classement
  mis à jour), branchement gateway (`/classements/*`).
- Fin — Rédaction de `README.md`, complétion de la
  section `service-classements` dans `2-contrats.md`, rédaction de ce
  journal. Vérification du service via `http://localhost:8080/classements/classement`.
- Bonus — Ajout de la table d'historique `Gain` (`db.py`) et des routes
  `/classement/periode/<jour|semaine>`, `/badges/<pseudo>`, et de la
  pagination sur `/classement` (`?page=&taille=`). Suite de tests
  `pytest` (`test_app.py`, 12 tests) couvrant routes de base, JWT et bonus.
- Intégration finale — Alignement du `JWT_SECRET` et de `auth.py` sur la
  version convenue par les 7 équipes (`je-suis-le-secret-tres-secret-12`).
  Rédaction du bloc `docker-compose.yml` à fournir à G1 (section dédiée du
  README). Script `test.sh` pour valider la stack via la gateway en une
  commande. Test croisé avec les dépôts G1/G2/G6 (JWT inter-équipes
  fonctionnel).
