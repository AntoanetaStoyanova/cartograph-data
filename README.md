# cartograph-data

Dépôt de données de [cartograph](https://github.com/AntoanetaStoyanova/cartograph) : il contient le workflow qui collecte chaque semaine le catalogue et les prix des cartes Pokémon depuis [TCGdex](https://tcgdex.dev). Le code du collecteur vit dans `cartograph` et est installé à chaque exécution.

## Ce qui est publié

- **Catalogue** : un fichier `cards-AAAA-MM-JJ.parquet` par semaine dans la Release `catalogue`. Il ne contient aucun prix ni aucune image, seulement les liens vers les images.
- **Résumés d'exécution** : nombres de cartes, échecs et fichiers écrits, sans prix.

## Ce qui n'est pas publié

Les prix (Cardmarket, TCGplayer) et les réponses brutes de l'API ne sont jamais publiés ici. Ils sont envoyés dans un dépôt privé : les prix TCGplayer appartiennent à TCGplayer, et ceux de Cardmarket relèvent du droit des bases de données.

## Attribution

Le catalogue provient de [TCGdex](https://github.com/tcgdex/cards-database), sous licence MIT, © TCGdex.

Ce projet n'est ni produit ni soutenu par Nintendo, The Pokémon Company, Creatures ou Game Freak. Pokémon et les noms associés sont des marques de leurs propriétaires.
