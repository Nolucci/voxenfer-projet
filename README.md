# Projet Voxenfer

Vous allez construire **un micro-service** de l'écosystème Voxenfer (le serveur de jeu Luanti et son écosystème de services).

## Par où commencer

1. Lisez **`1-sujet.md`** : le contexte, votre service, ce qu'on attend, le barème.
2. Lisez **`2-contrats.md`** : l'interface partagée (routes, champs JSON, codes). **À lire AVANT de coder**, et à valider tous ensemble en début de séance.
3. Le squelette est dans **`starter/`** (détails dans `starter/README.md`) : copiez `service-template/` pour créer votre `service-<votre-domaine>/`.

## Démarrer l'écosystème

```
cd starter
./demarrer.sh        # lance la gateway + le Postgres + les services déjà présents
./jouer.sh           # idem, avec le serveur de jeu Luanti (jeu fourni hors ligne)
```

La gateway est sur http://localhost:8080 (ex. `curl http://localhost:8080/monde/joueurs`).

Bon projet !
