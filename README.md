# cartograph-data

Dépôt de données de [cartograph](https://github.com/AntoanetaStoyanova/cartograph) : il contient le workflow qui collecte chaque jour les prix et chaque lundi le catalogue des cartes Pokémon depuis [TCGdex](https://tcgdex.dev). Le code du collecteur vit dans `cartograph` et est installé à chaque exécution.

## Ce qui est publié

- **Catalogue** : un fichier `cards-AAAA-MM-JJ.parquet` par semaine dans la Release `catalogue`. Il ne contient aucun prix ni aucune image, seulement les liens vers les images.
- **Résumés d'exécution** : nombres de cartes, échecs et fichiers écrits, sans prix.
- **Manifestes** : un `manifest/AAAA/AAAA-MM-JJ.json` commité par chaque exécution (voir « Manifeste »).

## Manifeste

Chaque exécution commite `manifest/AAAA/AAAA-MM-JJ.json` (auteur `github-actions[bot]`), même en échec. Une relance le même jour remplace le manifeste du jour. Il ne contient aucun prix ni identifiant de carte, seulement des statuts, des nombres et des empreintes :

- `date`, `run_id` : jour de la collecte (UTC) et exécution GitHub Actions ;
- `summary` : `false` si la collecte n'a écrit aucun résumé (`steps` et `files` sont alors vides) ;
- `steps` : par étape (`catalogue`, `prices`), `status`, `fetched` (cartes ou produits récupérés), `failed` (nombre en échec) ;
- `files` : par fichier, `name`, `release` (Release de destination), `rows`, `bytes`, `sha256` (`null` si le fichier manquait sur la machine, il n'est alors pas envoyé) ;
- `repo_kib` : taille de ce dépôt selon GitHub, en Kio (`-1` si GitHub ne l'a pas donnée).

Les étapes d'envoi lisent `files` : un fichier part dans la Release que donne le manifeste, et nulle part ailleurs. Pour vérifier un fichier téléchargé : `sha256sum <fichier>` doit donner son `sha256`.

## Déclenchement

- **Chaque jour à 03:17 UTC**, la tâche cron-job.org « Collecte cartograph » appelle l'API GitHub `workflow_dispatch` de `collect.yml` (réglages et jeton dans « Secrets, variables et services », plus bas ; le jeton expire le 30 octobre 2027). GitHub Actions exécute la collecte.
- **Pas de cron GitHub** (`on: schedule`) : GitHub ne garantit pas ses déclenchements, et celui de 03:17 UTC n'est pas parti les 29 et 30 octobre 2026.
- **Paramètres** : l'appel de cron-job.org n'en envoie aucun, les valeurs par défaut s'appliquent. `catalogue` vaut `auto` (catalogue le lundi, en UTC), `oui` ou `non` pour forcer ; `prices` vaut `true` ; `languages` vide (toutes) ; `code_ref` vaut `main`.
- **Relance manuelle** : onglet *Actions*, `Collecte`, *Run workflow*, ou `gh workflow run collect.yml -R AntoanetaStoyanova/cartograph-data`. Relancer le même jour remplace les fichiers du jour. Un lundi, `auto` relance aussi le catalogue (≈ 3 h 45) : choisir `non` pour les prix seuls (`-f catalogue=non`).

## Surveillance

- **Résumé** : chaque exécution affiche dans sa page le statut des étapes, les nombres de cartes, les échecs, les fichiers écrits et la durée, sans prix.
- **healthchecks.io** : le workflow envoie un signal au début et à la fin de chaque exécution (secret `HEALTHCHECKS_PING_URL`). Une exécution en échec envoie `/fail`, ce qui déclenche un e-mail immédiat. Sans signal de fin 5 heures après 03:17 UTC, un e-mail part aussi : c'est l'alerte quand aucune collecte n'est lancée (cron-job.org en panne, jeton expiré) ou qu'elle reste bloquée.
- **GitHub** : un e-mail est envoyé pour chaque exécution en échec.
- **cron-job.org** : un e-mail au premier appel en échec (réponse autre que `204`) et quand la tâche est désactivée.

## Secrets, variables et services

Ce dépôt est public : seuls les noms et les réglages figurent ici, jamais les valeurs.

| Nom | Type et emplacement | Rôle | Permissions | Expiration | Recréer |
| --- | --- | --- | --- | --- | --- |
| `PRICES_TOKEN` | Secret de `cartograph-data` | Envoie les prix et le brut dans les Releases `data-AAAA-MM` de `cartograph-prices` | Jeton à grain fin, dépôt `cartograph-prices` seul, « Contents : Read and write » | 28 octobre 2027 | *Settings → Developer settings → Fine-grained tokens* : régénérer le jeton (mêmes réglages), puis `gh secret set PRICES_TOKEN -R AntoanetaStoyanova/cartograph-data` |
| `HEALTHCHECKS_PING_URL` | Secret de `cartograph-data` | Adresse de signal du contrôle healthchecks.io (`/start`, succès, `/fail`) | Aucune : l'adresse ne sert qu'à envoyer des signaux | Aucune | Copier l'adresse de signal depuis la page du contrôle, puis `gh secret set HEALTHCHECKS_PING_URL -R AntoanetaStoyanova/cartograph-data` |
| `TCGDEX_USER_AGENT` | Variable de `cartograph-data` | En-tête `User-Agent` envoyé à TCGdex | Aucune | Aucune | Même valeur que `TCGDEX_USER_AGENT` du `.env` de `cartograph` : `gh variable set TCGDEX_USER_AGENT -R AntoanetaStoyanova/cartograph-data` |
| Jeton cron-job.org | En-tête `Authorization` de la tâche cron-job.org, hors GitHub | Lance `collect.yml` par `workflow_dispatch` | Jeton à grain fin, dépôt `cartograph-data` seul, « Actions : Read and write » | 30 octobre 2027, 22:00 UTC | Régénérer le jeton « cron-job.org cartograph-data » (mêmes réglages), puis remplacer `Bearer <jeton>` dans la tâche cron-job.org |

### Tâche cron-job.org « Collecte cartograph »

- URL : `https://api.github.com/repos/AntoanetaStoyanova/cartograph-data/actions/workflows/collect.yml/dispatches`
- Méthode `POST`, corps `{"ref": "main"}`.
- En-têtes : `Authorization: Bearer <jeton>`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`, `Content-Type: application/json`.
- Tous les jours à 03:17, fuseau UTC (fuseau de la tâche, pas celui du compte).
- Réponses gardées dans l'historique ; e-mail au premier échec et à la désactivation de la tâche.
- Réponse attendue : `204 No Content`.

### Contrôle healthchecks.io « Collecte cartograph »

- Cron `17 3 * * *`, fuseau UTC, grâce 5 heures.
- Alerte par e-mail.

## Remettre la collecte en marche

Après une longue pause ou une alerte qui dure, dans cet ordre :

1. **Workflow** : `gh workflow view collect.yml -R AntoanetaStoyanova/cartograph-data` doit afficher `active` ; sinon `gh workflow enable collect.yml -R AntoanetaStoyanova/cartograph-data`.
2. **Jetons** : vérifier les dates d'expiration du tableau. Un jeton expiré se régénère avec les mêmes réglages, puis sa valeur remplace l'ancienne (secret `PRICES_TOKEN`, en-tête de la tâche cron-job.org).
3. **Version de l'API GitHub** : `2022-11-28` est dépréciée depuis le 10 mars 2026 et ne sera plus acceptée après le 10 mars 2028. Avant cette date, ou si l'appel répond par une erreur de version, mettre une version prise en charge dans l'en-tête `X-GitHub-Api-Version` de la tâche cron-job.org (liste sur <https://docs.github.com/rest/about-the-rest-api/api-versions>).
4. **healthchecks.io** : le contrôle doit exister avec les réglages ci-dessus. S'il a été recréé, remplacer le secret `HEALTHCHECKS_PING_URL`.
5. **cron-job.org** : réactiver la tâche (cron-job.org la désactive après des échecs répétés), puis la tester avec son bouton de test : réponse `204`, et une exécution `workflow_dispatch` apparaît dans l'onglet *Actions*.
6. **Exécution de contrôle** : laisser finir l'exécution lancée par ce test (≈ 1 h 15, ≈ 3 h 45 un lundi avec le catalogue). Elle doit réussir, publier ses fichiers du jour et repasser le contrôle healthchecks.io au vert. Sans test cron-job.org, la lancer à la main : `gh workflow run collect.yml -R AntoanetaStoyanova/cartograph-data`, jamais avec une liste `languages` réduite, qui remplacerait les fichiers du jour par des fichiers incomplets.

## Ce qui n'est pas publié

Les prix (Cardmarket, TCGplayer) et les réponses brutes de l'API ne sont jamais publiés ici. Ils sont envoyés dans un dépôt privé : les prix TCGplayer appartiennent à TCGplayer, et ceux de Cardmarket relèvent du droit des bases de données.

## Attribution

Le catalogue provient de [TCGdex](https://github.com/tcgdex/cards-database), sous licence MIT, © TCGdex.

Ce projet n'est ni produit ni soutenu par Nintendo, The Pokémon Company, Creatures ou Game Freak. Pokémon et les noms associés sont des marques de leurs propriétaires.
