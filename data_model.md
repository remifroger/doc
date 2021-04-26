[![Ask Me Anything !](https://img.shields.io/badge/Ask%20me-anything-1abc9c.svg)](http://srv-gitlab.audiar.net/rfroger/pgsql-documentation/issues/new)

# Modèle de données

Les données sont multidimensionnelles et structurées en constellation (et/ou en flocons) selon une logique faits/dimensions. Les faits partagent des dimensions communes et les dimensions sont normalisées (voir [différence entre modèle en étoile et modèle en constellation](https://fr.wikipedia.org/wiki/%C3%89toile_(mod%C3%A8le_de_donn%C3%A9es))).  

Les dimensions ont été normalisées pour faciliter les mises à jour, notamment car les territoires administratifs bougent chaque année, et la normalisation (c'est-à-dire un échelon administratif = une table) permet d'avoir plus de maniabilité.  

Une exception : la dimension temporelle qui est dénormalisée, toutes les temporalités figurent dans la même table (mois, semestre, trimestre et année) car celle-ci ne fera pas l'object de mise à jour à l'avenir (seulement de l'insertion, si par exemple la valeur maximale de l'année stockée actuellement est 2030, il faudra, en 2030, insérer les années suivantes).  

## Principe du modèle en constellation et des dimensions floconnées 

![alt text](img/mcd_1.svg "Modèle de données - échantillon")

Les tables de faits contiennent des dimensions (axes d'analyse) et des mesures. Les dimensions de la table de faits sont des clés étrangères (indexées) vers les tables dimensionnelles, les mesures sont des champs numériques (agrégeables ou non, on peut stocker des taux par exemple). L'ensemble des clés étrangères (dimensions) est une clé d'unicité (par convention on ajoute aussi une clé primaire id_f).

Le principe essentiel est de déterminer en amont les dimensions et mesures selon la donnée à intégrer, il s'agit du travail de modélisation, nous verrons un [exemple plus bas](#exemple).  

Le but est de déterminer la granularité la plus fine possible de la donnée (c'est-à-dire le plus grand niveau de détails de chaque dimension). Cette phase dépend des contraintes métiers et de la donnée (qualité, source, garantie, durabilité, etc.). On ne mélange pas des granularités différentes dans une table de faits (impossible).

## Exemple

Imaginons que nous souhaitons intégrer une donnée ayant la structure suivante :


colonne | description
--- | ---
code_insee | code Insee de la commune
nom_commune | nom de la commune
code_departement | code du département
code_region | code de la région
pop_1530_h | population 15-30 ans masculine
pop_1530_f | population 15-30 ans féminine
pop_3045_h | population 30-45 ans masculine
pop_3045_f | population 30-45 ans féminine
pop_4554_h | population 45-54 ans masculine
pop_4554_f | population 45-54 ans féminine

Le premier réflexe est de déterminer les dimensions de cette donnée et donc la granularité. Il s'agit d'une donnée mise à jour annuellement, disponible à l'échelon communal, décrivant la population par tranche d'âge et par sexe.  

Première chose à vérifier, est-ce que cette donnée existe à un niveau de détail plus fin ? On pourrait supposer que la même information pourrait exister sur une temporalité plus fine (MAJ trimestrielle par exemple ?) ou à un niveau géographique plus fin (les IRIS ?) ou sur des tranches d'âge plus fines (seulement 3 ici) ou encore sur davantage de dimensions (2 dimensions métiers ici : tranches d'âge et sexe ; peut-être que la donnée existe sur une ou plusieurs variables supplémentaires).

Imaginons que la donnée existe à un niveau géographique plus fin (les IRIS) mais avec moins de variables métiers (on aurait simplement la population par sexe, par exemple). On gagnerait en précision sur l'échelon géographique, mais avec une perte de détails sur les dimensions métiers. Dans ce cas de figure, un choix est à faire, à savoir, intégration de 2 tables de faits pour les 2 données ou intégration d'une des 2 selon les besoins exprimés. Le choix de la granularité doit être guidé par les besoins de ceux qui utiliseront et analyseront la donnée par la suite.

Nous restons avec la donnée initiale. Après ce premier exercice, on distingue 4 dimensions en tout : la géographie (échelle communale), la temporalité (MAJ annuelle), les tranches d'âge et le sexe. *La majorité des données possède, a minima, une dimension géographique et une dimension temporelle.* La mesure de cette donnée est la population. Attention à bien respecter le format initial de la mesure, on pourrait penser que la population est un nombre entier, et en réalité non, les recensements de la population sont des estimations, la population est donc un nombre décimal. Il ne faut surtout pas arrondir, ou modifier, la donnée quantitative d'origine.  

Nous avons tous les éléments pour modéliser la donnée dans notre modèle de données multidimensionnelles. On vérifie dans un premier temps que les dimensions extraites n'existent pas déjà dans notre BDD. Si non, il faut les créer en respectant la nomenclature, et en repérant si l'une des dimensions ne pourrait pas être reliée à une dimension existante (clé étrangère).  

Les cardinalités entre les faits et dimensions sont toujours du (1, n). `Expliquer pourquoi`. On obtiendrait le MLD-R avec les tables suivantes :  

faits_pop_com_annee_ta_sexe(#id_f, *id_commune*, *id_temporalite*, *id_ta3*, *id_sexe*, population)  
dimension_communes(#id_commune, #code_insee, nom_commune, *id_dep*, *id_region*)  
dimension_temporalite(#id_temporalite, numero_annee)  
dimension_tranches_age_3(#id_ta3, lib_ta3)  
dimension_sexe(#id_sexe, lib_sexe)  

Les noms de tables et de colonnes indiqués ici sont à titre d'exemple et ne respecte pas la nomenclature fixée.

Le passage de ce MLD-R vers le modèle physique peut se réaliser directement en SQL (à la main si vous êtes à l'aise) ou en passant par un "modeleur" (ex. : pgModeler). En le faisant à la main, il faut bien penser à toutes les contraintes d'unicité, aux clés étrangères (en général on utilise les options `ON UPDATE NO ACTION ON DELETE NO ACTION` par sécurité) et aux index !

Les dimensions contiennent au moins un identifiant (entier auto-incrémenté) qui sera la clé primaire + une colonne contenant les valeurs métiers (ou plusieurs colonnes, on utilise dans notre cas un libellé court, un libellé long et/ou un libellé alternatif -> voir [la nomenclature](http://srv-gitlab.audiar.net/rfroger/pgsql-documentation/blob/master/regles_nommage.md)).

Les faits contiennent un identifiant (entier auto-incrémenté, ici id_f) comme première clé primaire, les clés étrangères vers les dimensions + une contrainte d'unicité correspondant à l'ensemble des clés étrangères dimensionnelles (`UNIQUE(id_commune, id_temporalite, id_ta3, id_sexe)`) et enfin une ou plusieurs colonnes de mesure (ici population, type numeric).

Enfin, un index par clé étrangère + un index composite (multi-colonnes) si une requête utilise fréquemment une combinaison de dimensions (par exemple on requête souvent sur la commune, l'année et le sexe -> la création d'un index(id_commune, id_temporalite, id_sexe) serait judicieux ; attention à l'ordre des colonnes, voir [la documentation de PostgreSQL pour plus d'informations](https://www.postgresql.org/docs/10/indexes-multicolumn.html)).

L'insertion des données dans les dimensions peut être effectuée à la main selon le volume de données (par exemple le remplissage d'une dimension contenant 3 valeurs ne nécessite pas la mise en place d'une chaîne de traitements via un ETL). La table de faits sera en revanche alimentée via un ETL (dans notre exemple, il s'agira simplement de faire pivoter les données et de rechercher les ID correspondant aux valeurs des dimensions).  

* Ne pas modéliser seul.e, et plutôt chercher à échanger et valider une modélisation avec un.e collègue
* Être sûr.e d'être sur une granularité la plus fine possible en adéquation avec les besoins
* Stocker la donnée source (donnée brute) quelque part

## Dimensions

Les dimensions correspondent aux axes d'analyse. Elles peuvent être partagées entre plusieurs tables de faits, une dimension peut également être reliée à une autre dimension (emboîtement normalisé). Le nommage des tables dimensionnelles et de leurs colonnes est expliqué [ici](http://srv-gitlab.audiar.net/rfroger/pgsql-documentation/blob/master/regles_nommage.md).

### Dimensions métiers

Les dimensions métiers sont propres à une donnée. Certaines sont générales et utilisées par différentes tables de faits (comme le sexe, les tranches d'âge ou les valeurs booléennes (du type oui/non)), d'autres sont très spécifiques et difficiles à réutiliser.

* Respecter la nomenclature
* Vérifier que la dimension n'existe pas déjà
* Vérifier si une dimension créée ne pourrait pas être associée à une autre
* Indexer les colonnes

### Dimensions territoriales (géographiques)

Les dimensions représentant le territoire administratif français commencent toutes par `d_territoires`. Les territoires sont normalisés, c'est-à-dire qu'on associe un territoire à un autre via une relation, on ne stocke pas en dur les associations territoriales. Par exemple les communes sont reliées aux EPCI (relation (1, n)), on va associer les 2 par une clé étrangère, ce qui suppose que 2 tables existent (les communes et les EPCI). On pourrait stocker le code et le libellé EPCI directement dans la table des communes sans créer de relations, mais cette façon de faire ne permet pas de conserver un historique des EPCI et d'avoir un modèle cohérent, souple et efficace. Elle aurait été pertinente si les EPCI restaient figés dans le temps, sans évolution.

**Intégrer un schéma de l'emboîtement des territoires**

### Dimensions temporelles

La dimension temporelle est dénormalisée et groupée dans une table (`d_temporalite`), avec pour niveau le plus fin : le mois. Ce choix a été fait car aucune MAJ ne sera effectuée sur cette table, seulement de l'insertion (si besoin d'une année non présente dans les valeurs actuelles) et pour en simplifier la gestion. Seul point noir, cette dénormalisation implique de réaliser des groupements ou d'associer un mois "fictif" lorsque la granularité du fait n'est pas aussi précise que le mois. Exemple, un fait ayant pour temporalité l'année sera par défaut rattaché à l'identifiant du mois de janvier de l'année.  

Une autre table temporelle existe : `d_temporalite_bornes`, correspondant à des bornes temporelles. Certaines données n'étaient pas accessibles à un niveau plus détaillé que la borne temporelle.

## Faits

### Faits, en général

Les tables de faits, comme vu dans l'exemple contiennent des clés étrangères vers les dimensions et des mesures. La combinaison des clés étrangères est une clé primaire. Les mesures doivent être de type `numeric`. Pour plus d'informations sur la nomenclature, voir [ici](http://srv-gitlab.audiar.net/rfroger/pgsql-documentation/blob/master/regles_nommage.md). Il est primordial de bien indexé les clés étrangères !  

En résumé, voici les points clés (ce stade suppose que la donnée ait été modélisée en amont) :

* Respecter la nomenclature
* La combinaison des clés étrangères dimensionnelles est une clé primaire
* Les mesures sont de type `numeric`, bien veiller à ce que les valeurs ne soient pas arrondies lors de l'intégration
* Indexation de chaque clé étrangère dimensionnelle, et éventuellement des index multi-colonnes si besoin (en particulier si des requêtes sont fréquentes sur des dimensions ciblées)
* Une colonne `id_f` auto-incrémentée comme clé primaire
* Documenter les métadonnées en commentaire de la table (à l'aide de la fonction `_a_fonctions_globales.generate_metadata_comment_table('nom_schéma', 'nom_table')`)
* Commenter la colonne de la mesure (voir exemple sur une table existante)
* Tester les valeurs
* Demander à un collègue de vérifier la structure de la table

### Faits géographiques

Un fait peut être géographique et appartenir en plus à une dimension géographique. Il suffit simplement d'ajouter une colonne géométrique à la table de faits. C'est le cas des mutations `f_foncier_val_surf_prixm2_com_annee_marches` dont on possède la géométrie et également la commune de référence. On pourrait se dire que l'association avec la dimension communale est inutile puisque celle-ci se récupérerait par jointure spatiale ; cependant la réalité de la donnée est autre, et la précision de la géométrie d'une mutation ne garantit pas forcément une jointure spatiale juste.
