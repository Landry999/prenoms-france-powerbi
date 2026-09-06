# Les prénoms en France, 1900–2025

Rapport Power BI construit sur le fichier des prénoms de l'INSEE : quels prénoms ont dominé chaque époque, comment les classements se sont renouvelés, et à quel point les choix se sont diversifiés en un siècle.

## Les deux pages

**Evolution** — la courbe d'un ou plusieurs prénoms sur toute la profondeur historique, avec recherche par saisie et filtres par sexe et par période.

**Palmarès** — le top 10 d'une période de référence, avec pour chaque prénom le nombre de naissances et son mouvement de rang depuis la période précédente (`▲ 2`, `▼ 1`, `entrée`). Trois cartes donnent le contexte : le n° 1 de l'année, le poids du top 10 dans l'ensemble des naissances, et une matrice croise les prénoms avec les onze périodes de référence, la couleur de chaque cellule encodant le rang.

## Les données

| Fichier | Contenu | Source |
|---|---|---|
| `prenoms-2025.parquet` | Naissances par prénom, sexe, année et territoire, avec le rang fourni par l'INSEE | INSEE — fichier des prénoms |
| `Prenoms.pbix` | Le rapport : modèle, mesures et mise en page | — |

Le rang venant directement de l'INSEE, aucune mesure de classement n'est recalculée au grain de l'année : `FRANCE[rang]` sert à la fois de valeur affichée et de filtre « top 10 ».

## Le modèle

Schéma en étoile, une table de faits et deux dimensions.

```
PRENOMS (prenom, sexe)  ──┐
                          ├──►  FRANCE (cle_prenom, periode, valeur, rang)
Date (periode, decennie, ─┘
      generation, periode_reference)
```

Deux colonnes de la table `Date` méritent une note :

`periode_reference` marque les onze années servant de points de comparaison (1900, 1930, 1940 … 2010, 2025). Elle est définie en DAX plutôt que par une sélection manuelle dans le volet Filtres, pour que le segment, la matrice et les mesures de mouvement restent synchronisés à partir d'une seule source.

`rang_periode` numérote ces mêmes périodes de 1 à 11. C'est elle qui permet de comparer une période à la précédente **disponible** : les écarts étant irréguliers — trente ans entre 1900 et 1930, quinze entre 2010 et 2025 — un simple `annee - 1` pointerait vers une année absente du palmarès.

## Les mesures

Elles sont regroupées dans une table `Mesure` dédiée et documentées dans [`docs/mesures-dax.md`](docs/mesures-dax.md), avec pour chacune l'intention et les pièges qu'elle contourne.

## Ouvrir le projet

Power BI Desktop, version de juin 2024 ou plus récente — le rapport utilise le format PBIR et les visuels *Carte* et *Segment* nouvelle génération.

Les chemins des sources sont absolus dans le `.pbix`. Après clonage, mets-les à jour via *Accueil → Transformer les données → Paramètres de la source de données*.

## Note sur le versionnement

Le `.pbix` est un binaire de 22 Mo : chaque enregistrement crée un objet complet dans l'historique Git, sans diff lisible. Pour un projet amené à évoluer, l'enregistrer en **`.pbip`** (*Fichier → Enregistrer sous → Projet Power BI*) produit à la place une arborescence de fichiers texte — les mesures deviennent alors comparables d'une version à l'autre, ce qui est tout l'intérêt de versionner un projet BI.
