---
title: "Corriger l'erreur GEOSException ou géométries invalides avec GeoPandas"
date: 2026-08-13T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "geopandas", "shapely", "geometrie", "depannage"]
summary: "Diagnostiquer et corriger les géométries invalides dans un GeoDataFrame — is_valid, explain_validity, make_valid — avant qu'une GEOSException n'interrompe un traitement GeoPandas."
---

`shapely.errors.GEOSException`, `TopologyException`, ou simplement un
résultat vide là où une intersection était attendue — ces erreurs
apparaissent généralement au moment d'une opération géométrique
(`overlay`, `union`, `intersection`) sur des données qui semblaient
pourtant correctes. Ce blog a déjà traité ce problème
[côté PostGIS](/posts/erreur-invalid-geometry-qgis-postgis/) — voici
l'équivalent côté Python, avec GeoPandas et Shapely.

## Pourquoi ça arrive avec GeoPandas spécifiquement

GeoPandas délègue tout calcul géométrique à **GEOS**, la même
bibliothèque C++ utilisée par PostGIS et QGIS en coulisses. Les mêmes
règles de validité s'appliquent donc (voir l'article PostGIS pour le
détail des cas : auto-intersection, anneau non fermé, points dupliqués).
La différence : quand une géométrie invalide déclenche une erreur côté
PostGIS, la requête SQL échoue avec un message clair. Côté GeoPandas,
l'erreur peut apparaître **plus tard**, au moment d'une opération
combinée comme `overlay()`, avec un message parfois moins direct
(`GEOSException: TopologyException: found non-noded intersection...`).

## 1. Identifier les géométries invalides

```python
import geopandas as gpd

gdf = gpd.read_file("pepinieres.geojson")

invalides = gdf[~gdf.geometry.is_valid]
print(f"{len(invalides)} géométrie(s) invalide(s) sur {len(gdf)}")
```

`is_valid` est une colonne booléenne calculée à la volée pour chaque
géométrie — le point de départ systématique avant toute opération
géométrique un peu complexe (jointure spatiale, overlay, dissolve).

## 2. Comprendre pourquoi

```python
from shapely.validation import explain_validity

for idx, geom in invalides.geometry.items():
    print(idx, "→", explain_validity(geom))
```

`explain_validity()` renvoie une explication précise — par exemple
`"Self-intersection[2.391 6.370]"` — avec les coordonnées exactes du
point problématique, exactement comme `ST_IsValidReason` côté PostGIS.

## 3. Corriger : `make_valid()`

```python
gdf["geometry"] = gdf.geometry.make_valid()
```

Depuis GeoPandas 0.13 / Shapely 2.0, `.make_valid()` est disponible
directement comme méthode sur la série de géométries — c'est la méthode
recommandée aujourd'hui, l'équivalent direct du `ST_MakeValid` de
PostGIS.

### L'ancienne astuce, encore répandue dans les tutoriels

```python
gdf["geometry"] = gdf.geometry.buffer(0)
```

Cette technique fonctionnait avant l'existence de `.make_valid()` — un
buffer de distance nulle force Shapely à recalculer la géométrie, ce qui
corrige au passage de nombreux cas d'invalidité. Elle reste largement
utilisée par habitude, mais `.make_valid()` est aujourd'hui préférable :
plus explicite sur l'intention, et plus prévisible sur les cas complexes.

{{< pub slot="1313131313" >}}

## 4. Attention aux géométries vides et nulles — un piège différent

Une géométrie peut être *valide* mais poser problème pour d'autres
raisons :

```python
# Géométries vides (ex. résultat d'une intersection qui ne se touche pas)
vides = gdf[gdf.geometry.is_empty]

# Géométries manquantes (valeur NULL/None)
manquantes = gdf[gdf.geometry.isna()]

# Retirer les deux avant un traitement plus loin
gdf = gdf[~gdf.geometry.is_empty & gdf.geometry.notna()]
```

`.make_valid()` ne corrige ni les géométries vides ni les valeurs `None`
— ce sont des cas à traiter séparément, généralement en excluant ces
lignes plutôt qu'en cherchant à les "réparer".

## 5. Le cas fréquent : `GEOSException` pendant un `overlay()`

```python
import geopandas as gpd

communes = gpd.read_file("communes.geojson")
pepinieres = gpd.read_file("pepinieres.geojson")

# Nettoyage préventif AVANT l'opération, sur les deux couches
communes["geometry"] = communes.geometry.make_valid()
pepinieres["geometry"] = pepinieres.geometry.make_valid()

resultat = gpd.overlay(communes, pepinieres, how="intersection")
```

**La règle la plus importante de cet article** : dans une opération qui
combine deux couches (`overlay`, jointure spatiale), **les deux** doivent
être valides — une seule géométrie invalide dans l'une des deux couches
suffit à faire échouer (ou fausser silencieusement) le résultat complet.

## Fonction de nettoyage réutilisable

```python
from shapely.validation import explain_validity

def nettoyer_geometries(gdf, verbose=True):
    invalides = gdf[~gdf.geometry.is_valid]
    if verbose and len(invalides) > 0:
        print(f"{len(invalides)} géométrie(s) invalide(s) corrigée(s) :")
        for idx, geom in invalides.geometry.items():
            print(f"  ligne {idx} : {explain_validity(geom)}")

    gdf = gdf.copy()
    gdf["geometry"] = gdf.geometry.make_valid()
    gdf = gdf[~gdf.geometry.is_empty & gdf.geometry.notna()]
    return gdf


pepinieres = nettoyer_geometries(gpd.read_file("pepinieres.geojson"))
```

Appeler cette fonction juste après chaque `gpd.read_file()` évite de
découvrir le problème bien plus tard, au milieu d'un traitement plus
long.

## Cas particulier : `GEOMETRYCOLLECTION` après `make_valid()`

Sur une géométrie très dégradée (auto-intersection sévère qui scinde
un polygone en plusieurs morceaux disjoints), `.make_valid()` peut
renvoyer une `GEOMETRYCOLLECTION` mélangeant plusieurs types de
géométrie plutôt qu'un simple polygone :

```python
def garder_polygones_seulement(geom):
    if geom.geom_type == "GeometryCollection":
        polygones = [g for g in geom.geoms if g.geom_type in ("Polygon", "MultiPolygon")]
        if polygones:
            from shapely.ops import unary_union
            return unary_union(polygones)
        return None
    return geom

gdf["geometry"] = gdf.geometry.apply(garder_polygones_seulement)
```

## Pour aller plus loin

- **Précision numérique** : `shapely.set_precision()` peut résoudre des
  invalidités causées par des coordonnées avec une précision excessive
  (des différences de l'ordre de 10⁻¹⁵ qui créent des intersections
  fantômes) — un cas plus rare, mais qui résiste parfois à
  `make_valid()` seul.
- **`ST_MakeValid` côté base de données** : si les mêmes données
  transitent aussi par PostGIS (voir l'article dédié), corriger dès
  l'entrée en base évite de refaire le même nettoyage à chaque script
  Python qui lit ensuite ces données.
- **Valider à l'écriture, pas seulement à la lecture** : après une
  opération de simplification (`gdf.simplify(tolerance)`), revérifier
  `is_valid` — la simplification elle-même peut, dans de rares cas,
  produire une géométrie invalide à partir d'une géométrie initialement
  correcte.

Le réflexe à garder : `is_valid` avant toute opération géométrique un peu
sérieuse, `explain_validity()` pour comprendre plutôt que deviner, et
`make_valid()` pour corriger — dans cet ordre, systématiquement.
