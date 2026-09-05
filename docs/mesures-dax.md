# Mesures DAX

Référence des mesures et colonnes calculées du rapport, avec l'intention de chacune. En cas de divergence, `Prenoms.pbix` fait foi.

Convention de nommage des tables : `FRANCE` (faits), `PRENOMS` (dimension prénom), `'Date'` (dimension temps, entre apostrophes car `DATE` est une fonction DAX réservée), `Mesure` (table d'accueil des mesures).

---

## Colonnes calculées — table `Date`

### `periode_reference`

```dax
periode_reference = 
'Date'[annee] IN { 1900, 1930, 1940, 1950, 1960, 1970, 1980, 1990, 2000, 2010, 2025 }
```

Matérialise la liste des périodes de comparaison dans le modèle. Une restriction posée dans le volet Filtres serait invisible en DAX : depuis une mesure, `ALLSELECTED` restaure le contexte extérieur au visuel, lequel contient déjà la sélection du segment — impossible d'y retrouver la liste que le segment proposait.

### `rang_periode`

```dax
rang_periode = 
IF(
    'Date'[periode_reference],
    RANKX(
        FILTER('Date', 'Date'[periode_reference]),
        'Date'[annee], , ASC, DENSE
    ),
    -1
)
```

Numérote les périodes de référence de 1 à 11. Le `-1` en valeur par défaut n'est pas cosmétique : en DAX `BLANK() = 0` renvoie **VRAI**, donc un index laissé vide sur les années hors référence ferait correspondre `rang_periode = 0` à toutes ces années. Sur la première période, `index - 1` vaut 0 et capturerait alors 1901 à 2024 — le mouvement et le sous-titre afficheraient une comparaison inventée, sans aucune erreur pour prévenir.

---

## Socle

### `Naissances`

```dax
Naissances = SUM(FRANCE[valeur])
```

### `Rang`

```dax
Rang = 
IF(
    NOT ISBLANK([Naissances]) && HASONEVALUE(PRENOMS[sexe]),
    MIN(FRANCE[rang])
)
```

`MIN` n'agrège rien en pratique — dans le contexte d'un prénom, d'une période et d'un sexe il n'existe qu'une ligne. C'est le moyen de transformer une colonne en mesure.

Le garde-fou sur le sexe est indispensable : le rang de l'INSEE est calculé séparément pour les filles et les garçons. Sans lui, un prénom mixte comme Camille remonte ses deux lignes et affiche le meilleur de ses deux classements.

### `Annee actuelle`

```dax
Annee actuelle = MAX('Date'[annee])
```

`MAX` plutôt que `SELECTEDVALUE` : il respecte le contexte de filtre et couvre tous les cas d'un coup — une année épinglée, une année portée par l'axe d'un visuel, une plage dont il prend la borne haute, ou aucune sélection auquel cas il rend la dernière année plutôt qu'un écran blanc.

### `Index periode`

```dax
Index periode = MAX('Date'[rang_periode])
```

---

## Mouvement de rang

### `Rang N-1`

```dax
Rang N-1 = 
VAR IdxPrecedent = [Index periode] - 1
RETURN
    CALCULATE(
        MIN(FRANCE[rang]),
        REMOVEFILTERS('Date'),
        'Date'[rang_periode] = IdxPrecedent
    )
```

Le `REMOVEFILTERS('Date')` est obligatoire, pas prudentiel. Un filtre booléen dans `CALCULATE` ne neutralise que sa propre colonne : le segment portant sur `periode` alors que le décalage se fait sur `rang_periode`, la condition viendrait **s'ajouter** au lieu de remplacer, les deux seraient incompatibles et toute la colonne remonterait vide.

### `Mouvement`

```dax
Mouvement = 
VAR RangActuel = [Rang]
VAR RangPrec   = [Rang N-1]
RETURN
    IF(
        NOT ISBLANK(RangActuel) && NOT ISBLANK(RangPrec),
        RangPrec - RangActuel
    )
```

Sens de la soustraction : passer de 5 à 2 donne **+3**, soit trois places gagnées. C'est l'inverse de l'intuition arithmétique — le rang baisse quand on progresse — d'où l'intérêt de le fixer une fois ici, pour que tout le reste du rapport lise « positif = ça monte ».

### `Mouvement (libellé)`

```dax
Mouvement (libellé) = 
VAR IdxCourant = [Index periode]
VAR RangActuel = [Rang]
VAR RangPrec   = [Rang N-1]
VAR EtaitTop10 = NOT ISBLANK(RangPrec) && RangPrec <= 10
VAR Delta      = RangPrec - RangActuel
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(RangActuel), BLANK(),
        IdxCourant = 1,      "—",
        NOT EtaitTop10,      "entrée",
        Delta > 0,           "▲ " & Delta,
        Delta < 0,           "▼ " & ABS(Delta),
                             "="
    )
```

L'ordre des branches porte la logique. La première période ne peut pas avoir de mouvement : elle rend un tiret, pas « entrée », sinon 1900 déclarerait que dix prénoms viennent d'apparaître.

`EtaitTop10` teste `RangPrec <= 10` et non `ISBLANK` : la table contenant le rang de tous les prénoms, un prénom 47ᵉ à la période précédente a bien un rang antérieur, et sans ce test le libellé annoncerait « ▲ 37 » au lieu de « entrée ».

### `Mouvement (couleur)`

```dax
Mouvement (couleur) = 
SWITCH(
    TRUE(),
    [Mouvement] > 0, "#217552",
    [Mouvement] < 0, "#B23D3D",
                     "#8A8DA6"
)
```

À brancher sur *Étiquettes de données → Couleur → fx → Valeur du champ*.

### `Periode comparee`

```dax
Periode comparee = 
VAR IdxCourant = [Index periode]
RETURN
    IF(
        IdxCourant > 1,
        CALCULATE(
            MAX('Date'[periode]),
            REMOVEFILTERS('Date'),
            'Date'[rang_periode] = IdxCourant - 1
        )
    )
```

### `Détail mouvement`

```dax
Détail mouvement = 
VAR Prec       = [Periode comparee]
VAR RangActuel = [Rang]
VAR RangPrec   = [Rang N-1]
VAR Delta      = RangPrec - RangActuel
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(RangActuel), BLANK(),
        ISBLANK(Prec),       "Première période disponible",
        ISBLANK(RangPrec) || RangPrec > 10, "Entrée dans le top 10 depuis " & Prec,
        Delta > 0,           "Gagne " & Delta & " place(s) depuis " & Prec,
        Delta < 0,           "Perd " & ABS(Delta) & " place(s) depuis " & Prec,
                             "Rang stable depuis " & Prec
    )
```

Posée dans le puits **Info-bulles**. Elle nomme la période de comparaison, ce que les écarts irréguliers rendent nécessaire : trois places gagnées en trente ans et trois places gagnées en dix ne racontent pas la même chose, et rien à l'écran ne permettrait autrement de savoir laquelle on regarde.

---

## Titres dynamiques

### `Titre palmarès`

```dax
Titre palmarès = 
VAR Periode = SELECTEDVALUE('Date'[periode])
VAR Prec    = [Periode comparee]
RETURN
    "Top 10 — " & Periode &
    IF(
        ISBLANK(Prec),
        " · première période du rapport, pas de comparaison",
        " · évolution depuis " & Prec
    )
```

### `Titre Evolution`

```dax
Titre Evolution = 
VAR ListePrenoms = VALUES(PRENOMS[prenom])
VAR Nb     = COUNTROWS(ListePrenoms)
VAR Reste  = Nb - 3
VAR Trois  = 
    CONCATENATEX(
        TOPN(3, ListePrenoms, PRENOMS[prenom], ASC),
        PRENOMS[prenom], ", ", PRENOMS[prenom], ASC
    )
VAR Sujet = 
    SWITCH(
        TRUE(),
        NOT ISFILTERED(PRENOMS[prenom]), "de tous les prénoms",
        Nb = 1,  "du prénom "   & Trois,
        Nb <= 3, "des prénoms " & Trois,
                 "des prénoms " & Trois & " et " & Reste & IF(Reste = 1, " autre", " autres")
    )
VAR AnneeMin = MIN('Date'[annee])
VAR AnneeMax = MAX('Date'[annee])
VAR Periode = 
    IF(
        AnneeMin = AnneeMax,
        " en "   & FORMAT(AnneeMin, "0"),
        " de "   & FORMAT(AnneeMin, "0") & " à " & FORMAT(AnneeMax, "0")
    )
RETURN
    "Évolution " & Sujet & Periode
```

Trois mécanismes s'y combinent. `ISFILTERED` distingue « rien de sélectionné » de « beaucoup de sélectionné » — sans lui, `VALUES` renverrait les milliers de prénoms de la table et le titre annoncerait « et 11 997 autres ». Le `TOPN(3)` protège la mise en page, un titre sur trois lignes déplaçant tout le visuel sous lui. Et `MIN`/`MAX` sur `annee` ne peuvent jamais être vides, là où un `SELECTEDVALUE` sur la décennie laissait un trou dans la phrase dès que zéro ou deux décennies étaient cochées.

---

## Cartes

### `Prenom n1`

```dax
Prenom n1 = 
VAR Gagnant = 
    TOPN(
        1,
        CALCULATETABLE(VALUES(PRENOMS[prenom]), ALLSELECTED(PRENOMS[prenom])),
        [Naissances],
        DESC
    )
RETURN
    IF(
        HASONEVALUE(PRENOMS[sexe]),
        CONCATENATEX(Gagnant, PRENOMS[prenom], " / ")
    )
```

`CONCATENATEX` plutôt que `SELECTEDVALUE` : `TOPN` renvoie plusieurs lignes en cas d'égalité, et la carte se viderait sur une année d'ex æquo sans que rien n'explique pourquoi.

### `Naissances de l'année`

```dax
Naissances de l'année = 
CALCULATE(
    [Naissances],
    REMOVEFILTERS(PRENOMS[prenom]),
    REMOVEFILTERS(FRANCE[rang])
)
```

### `Poids du top 10`

```dax
Poids du top 10 = 
VAR Top10 = CALCULATE([Naissances], KEEPFILTERS(FRANCE[rang] <= 10))
RETURN
    DIVIDE(Top10, [Naissances de l'année])
```

Le `REMOVEFILTERS(FRANCE[rang])` du dénominateur évite le piège classique : sans lui, numérateur et dénominateur seraient tous deux restreints au top 10 et la carte afficherait un fier 100 %.

Format en pourcentage sur la mesure elle-même, une décimale — pas sur le visuel, pour qu'elle reste correcte partout où elle est réutilisée, info-bulles comprises.

---

## Matrice

### `Meilleur rang de référence`

```dax
Meilleur rang de référence = 
CALCULATE(
    MIN(FRANCE[rang]),
    REMOVEFILTERS('Date'),
    'Date'[periode_reference] = TRUE()
)
```

Filtre de lignes de la matrice, avec un seuil (`<= 3` par défaut) qui sert de curseur de densité.

C'est ce qui remplace un filtre « N premiers par naissances », lequel ne pouvait que sélectionner des prénoms anciens : Jean culmine au-dessus de 40 000 naissances en 1900, Gabriel plafonne vers 5 000 en 2025, et le top 10 pesait plus de 40 % des naissances en 1900 contre moins de 10 % aujourd'hui. Un classement par volume cumulé vidait donc entièrement les colonnes récentes.

Le `REMOVEFILTERS('Date')` la rend utilisable comme filtre de lignes : sans lui elle serait recalculée colonne par colonne et viderait des cellules au lieu de retirer des lignes.

---

## Page Evolution

### `Pic du prénom`

```dax
Pic du prénom = 
VAR Historique = 
    CALCULATETABLE(
        ADDCOLUMNS(VALUES('Date'[annee]), "@n", [Naissances]),
        REMOVEFILTERS('Date')
    )
VAR Sommet   = MAXX(Historique, [@n])
VAR AnneePic = MINX(FILTER(Historique, [@n] = Sommet), 'Date'[annee])
RETURN
    IF(
        HASONEVALUE(PRENOMS[prenom]) && Sommet > 0,
        FORMAT(AnneePic, "0") & " · " & FORMAT(Sommet, "#,##0") & " naissances",
        "Sélectionne un prénom"
    )
```

`ADDCOLUMNS` matérialise la table une fois pour deux usages — le maximum, puis la recherche de la ligne qui le porte — au lieu de recalculer `[Naissances]` sur 126 années une seconde fois. Le préfixe `@` du nom de colonne temporaire évite qu'elle masque une mesure du même nom, collision qui ne produit aucune erreur mais fausse silencieusement le résultat.

`MINX` sur les ex æquo retient l'année la plus ancienne, ce qui rend la mesure déterministe d'un rafraîchissement à l'autre.

### `Meilleur rang du prénom`

```dax
Meilleur rang du prénom = 
VAR Historique = 
    CALCULATETABLE(
        ADDCOLUMNS(VALUES('Date'[annee]), "@r", [Rang]),
        REMOVEFILTERS('Date')
    )
VAR Meilleur      = MINX(Historique, [@r])
VAR AnneeMeilleur = MINX(FILTER(Historique, [@r] = Meilleur), 'Date'[annee])
RETURN
    IF(
        HASONEVALUE(PRENOMS[prenom]) && NOT ISBLANK(Meilleur),
        "n° " & Meilleur & " en " & FORMAT(AnneeMeilleur, "0")
    )
```

Le pic de volume et le pic de popularité tombent rarement la même année : le nombre de naissances dépend autant de la natalité de l'époque que de la mode du prénom. Afficher les deux côte à côte est ce qui rend la page lisible — « Pic : 1998 · 11 000 naissances » et « Meilleur rang : n° 1 en 1996 » racontent ensemble une histoire qu'aucune des deux ne dit seule.

---

## Deux règles apprises en construisant ce rapport

**`BLANK()` s'égalise à `0` et à `""`.** Ne jamais comparer à zéro une colonne susceptible d'être vide : le test passe silencieusement et rend un résultat plausible, donc difficile à repérer. C'est ce qui a motivé le `-1` de `rang_periode`.

**Un filtre booléen dans `CALCULATE` ne remplace que sa propre colonne.** Dès que le segment et le calcul portent sur deux colonnes différentes de la même dimension, il faut un `REMOVEFILTERS` sur la table entière.
