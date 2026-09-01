---
title: "Résoudre l'erreur \"Invalid geometry\" dans QGIS/PostGIS"
date: 2026-08-15T10:00:00+01:00
draft: false
categories: ["PostGIS"]
tags: ["postgis", "qgis", "geometrie", "depannage", "sql"]
summary: "Comprendre pourquoi une géométrie devient invalide, la diagnostiquer précisément, et la corriger — en PostGIS avec ST_MakeValid, ou directement dans QGIS."
---

*"Invalid geometry"*, *"Self-intersection"*, *"Ring Self-intersection"* —
ces messages apparaissent tôt ou tard dans presque tout projet SIG, souvent
au moment le plus gênant : en plein calcul PostGIS, ou en pleine analyse
QGIS qui s'arrête net. Ce n'est presque jamais un bug de l'outil — c'est la
géométrie elle-même qui ne respecte pas les règles topologiques attendues.
Voici comment diagnostiquer précisément le problème, puis le corriger.

## Qu'est-ce qu'une géométrie "invalide" ?

Une géométrie est valide si elle respecte un ensemble de règles
topologiques définies par la norme OGC — par exemple, pour un polygone :
ses anneaux ne doivent pas se croiser eux-mêmes, ne doivent pas se
toucher en un point unique de façon ambiguë, et doivent être correctement
fermés. Les cas les plus fréquents :

- **Auto-intersection** (*self-intersection*) : le contour du polygone se
  croise lui-même — typiquement une forme en "nœud papillon"
- **Anneau non fermé** : le premier et le dernier point d'un polygone ne
  coïncident pas exactement
- **Points dupliqués consécutifs** : deux sommets identiques à la suite,
  qui créent un segment de longueur nulle
- **Trou mal positionné** : un anneau intérieur (trou) qui sort des
  limites de l'anneau extérieur, ou qui le touche de façon incorrecte

## D'où viennent ces erreurs ?

- **Digitalisation manuelle imprécise** : un clic mal placé en dessinant
  un polygone dans QGIS peut créer une auto-intersection invisible à
  l'œil nu au niveau de zoom normal.
- **Import de formats peu stricts** : les Shapefile et KML n'imposent pas
  toujours une validation stricte à l'écriture — un fichier peut sembler
  correct dans son logiciel d'origine et se révéler invalide une fois
  importé ailleurs.
- **Opérations géométriques ratées** : un `ST_Buffer`, une simplification
  (`ST_Simplify`) trop agressive, ou une fusion (`ST_Union`) de
  géométries elles-mêmes limites peuvent produire un résultat invalide,
  même si les entrées étaient parfaitement valides.
- **Conversion entre systèmes de coordonnées** : une reprojection near
  les pôles ou sur l'antiméridien peut, dans de rares cas, déformer une
  géométrie jusqu'à l'invalider.

## Diagnostiquer dans PostGIS

### Vérifier une géométrie précise

```sql
SELECT ST_IsValid(way) FROM planet_osm_polygon WHERE osm_id = 123456;
```

Renvoie simplement `true` ou `false`. Pour comprendre **pourquoi** c'est
invalide :

```sql
SELECT ST_IsValidReason(way) FROM planet_osm_polygon WHERE osm_id = 123456;
```

Ce qui renvoie une explication du type
`"Self-intersection at 2.391 6.370"` — avec les coordonnées exactes du
point problématique, directement localisables dans QGIS.

### Trouver toutes les géométries invalides d'une table

```sql
SELECT osm_id, ST_IsValidReason(way)
FROM planet_osm_polygon
WHERE NOT ST_IsValid(way);
```

Sur une table volumineuse (import OSM par exemple), cette requête peut
prendre un moment — un index spatial n'accélère pas ce type de
vérification ligne par ligne, seule une lecture complète le permet.

## Corriger dans PostGIS

### La solution moderne : `ST_MakeValid`

```sql
UPDATE planet_osm_polygon
SET way = ST_MakeValid(way)
WHERE NOT ST_IsValid(way);
```

`ST_MakeValid` reconstruit la géométrie en respectant les règles
topologiques, en modifiant le moins possible la forme d'origine. C'est la
méthode recommandée depuis PostGIS 2.0+ — largement plus fiable que
l'ancienne astuce du buffer nul ci-dessous.

### L'ancienne astuce : buffer de distance zéro

```sql
UPDATE planet_osm_polygon
SET way = ST_Buffer(way, 0)
WHERE NOT ST_IsValid(way);
```

Cette technique fonctionnait avant l'existence de `ST_MakeValid` — un
buffer de distance 0 force PostGIS à recalculer la géométrie et corrige
au passage certaines invalidités. Elle reste mentionnée ici parce
qu'elle apparaît encore dans beaucoup de tutoriels en ligne, mais
`ST_MakeValid` est aujourd'hui préférable dans presque tous les cas :
plus prévisible, et pensée spécifiquement pour ce problème.

{{< pub slot="1010101010" >}}

### Empêcher les géométries invalides d'entrer en base

Une contrainte `CHECK` bloque toute insertion de géométrie invalide dès
la source, plutôt que de devoir nettoyer après coup :

```sql
ALTER TABLE pepinieres
ADD CONSTRAINT geom_valide CHECK (ST_IsValid(geometry));
```

## Diagnostiquer et corriger directement dans QGIS

Pas besoin de PostGIS pour un fichier local (Shapefile, GeoPackage) — QGIS
intègre ses propres outils, dans la boîte à outils de traitement
(**Traitement → Boîte à outils**) :

1. **Vérifier la validité** (*Check Validity*) : analyse une couche et
   produit trois couches en sortie — géométries valides, invalides, et un
   rapport d'erreurs avec la raison précise pour chacune (identique dans
   l'esprit à `ST_IsValidReason`).
2. **Réparer les géométries** (*Fix geometries*) : corrige
   automatiquement les géométries invalides d'une couche — l'équivalent
   QGIS de `ST_MakeValid`, utilisable sans jamais toucher à PostGIS.

Pour repérer une géométrie invalide **pendant la digitalisation** plutôt
qu'après coup : **Préférences → Numérisation → Vérification de la
topologie**, qui alerte en temps réel dès qu'une géométrie tracée devient
invalide.

## Cas particuliers

- **`ST_IsEmpty`** : une géométrie peut être *valide* mais *vide* (par
  exemple le résultat d'une intersection entre deux polygones qui ne se
  touchent pas) — un cas différent d'une géométrie invalide, à tester
  séparément (`SELECT ST_IsEmpty(way) FROM ...`).
- **Coordonnées `NaN` ou infinies** : provoquent des erreurs plus graves
  qu'une simple invalidité topologique, souvent causées par une division
  par zéro en amont dans un calcul — `ST_MakeValid` ne les corrige pas,
  il faut filtrer ces lignes explicitement avant tout traitement.
- **`GEOMETRYCOLLECTION` inattendue en sortie** : `ST_MakeValid` peut,
  dans certains cas complexes (polygone qui s'auto-intersecte au point de
  se scinder en deux formes disjointes), renvoyer une collection
  hétérogène plutôt qu'un simple polygone — un `CASE` ou un filtre sur
  `ST_GeometryType()` après coup permet de ne garder que les types de
  géométrie attendus.

## Pour aller plus loin

- **`ST_CollectionExtract`** : utile juste après un `ST_MakeValid` qui a
  produit une `GEOMETRYCOLLECTION`, pour n'en extraire que les polygones
  (ou lignes, ou points) et ignorer le reste.
- **`ST_SimplifyPreserveTopology`** : une alternative à `ST_Simplify`
  quand la simplification elle-même est la cause de géométries devenues
  invalides.
- **Valider systématiquement après tout import** : ajouter une étape de
  vérification (`ST_IsValid`) directement dans un pipeline d'import
  (comme celui utilisant [osm2pgsql](/posts/osm2pgsql-import-postgis/))
  évite de découvrir le problème bien plus tard, au moment d'une analyse
  spatiale qui s'arrête sans prévenir.

La règle la plus utile à retenir : ne jamais essayer de deviner où se
situe le problème à l'œil — `ST_IsValidReason` (ou "Check Validity" dans
QGIS) donne toujours la localisation exacte, ce qui transforme un
dépannage qui pourrait prendre une heure en une correction de quelques
minutes.
