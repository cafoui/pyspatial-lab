---
title: "GeoPandas pour les débutants — charger, filtrer et visualiser un shapefile en 10 lignes de code"
date: 2026-08-24T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "geopandas", "shapefile", "cartographie"]
summary: "L'essentiel de GeoPandas en un seul exemple concret : charger un shapefile mondial, filtrer par continent et population, puis visualiser le résultat — avec une vraie donnée publique, sans jeu de données factice."
---

GeoPandas est la porte d'entrée la plus naturelle en Python pour manipuler
des données géospatiales : il combine la puissance de **pandas** (tableaux,
filtres, agrégations) avec la notion de **géométrie** (points, lignes,
polygones). Dans ce tutoriel, on charge un shapefile mondial réel, on le
filtre selon deux critères, et on le visualise — le tout en une dizaine de
lignes de code utile.

## Installer GeoPandas

```bash
pip install geopandas matplotlib
```

GeoPandas s'appuie sur des bibliothèques géospatiales de bas niveau (GDAL,
GEOS, PROJ) — l'installation via `pip` fonctionne dans la grande majorité
des cas aujourd'hui, mais si tu rencontres des erreurs de compilation,
`conda install -c conda-forge geopandas` reste l'option la plus fiable.

## Une remarque avant de commencer

Beaucoup de tutoriels GeoPandas encore en ligne utilisent
`gpd.datasets.get_path("naturalearth_lowres")`. Ce jeu de données intégré a
été **supprimé de GeoPandas** (retiré définitivement en version 1.0) — si tu
vois cette ligne quelque part, elle ne fonctionnera plus. On charge donc ici
les données directement depuis leur source, ce qui est de toute façon une
meilleure habitude à prendre.

## Le code complet

```python
import geopandas as gpd
import matplotlib.pyplot as plt

# 1. Charger un shapefile mondial (pays), directement depuis son URL
url = "https://naturalearth.s3.amazonaws.com/110m_cultural/ne_110m_admin_0_countries.zip"
world = gpd.read_file(url)

# 2. Filtrer : pays africains de plus de 50 millions d'habitants
africa_large = world[(world["CONTINENT"] == "Africa") & (world["POP_EST"] > 50_000_000)]

# 3. Visualiser le résultat, coloré selon la population
africa_large.plot(column="POP_EST", legend=True, cmap="YlOrRd", figsize=(9, 7))
plt.title("Pays africains de plus de 50 millions d'habitants")
plt.show()
```

Voilà : chargement, filtrage et visualisation en une poignée de lignes.
Détaillons chaque étape.

{{< pub slot="2222222222" >}}

## 1. Charger : `gpd.read_file()`

```python
world = gpd.read_file(url)
```

GeoPandas peut lire un shapefile directement **depuis une URL**, y compris
un fichier `.zip` contenant les 4-5 fichiers qui composent un shapefile
(`.shp`, `.shx`, `.dbf`, `.prj`...) — aucun téléchargement manuel n'est
nécessaire. Cette même fonction lit aussi le GeoJSON, le GeoPackage, et une
dizaine d'autres formats : le format est détecté automatiquement.

Le résultat, `world`, est un **GeoDataFrame** : un tableau pandas classique,
avec une colonne spéciale nommée `geometry` qui contient la forme de chaque
pays (un polygone, ou un ensemble de polygones pour les pays avec des îles).

```python
world.head()
world.crs        # système de coordonnées (ici EPSG:4326 — WGS84)
len(world)        # 177 pays
```

## 2. Filtrer : comme un DataFrame pandas classique

```python
africa_large = world[(world["CONTINENT"] == "Africa") & (world["POP_EST"] > 50_000_000)]
```

C'est là toute la force de GeoPandas : puisqu'un GeoDataFrame **est** un
DataFrame, tous les filtres pandas habituels fonctionnent tels quels — la
géométrie suit automatiquement le filtre, sans rien faire de spécial.

Quelques variantes utiles :

```python
# Un seul critère
africa = world[world["CONTINENT"] == "Africa"]

# Avec .query(), souvent plus lisible pour plusieurs conditions
africa_large = world.query("CONTINENT == 'Africa' and POP_EST > 50_000_000")

# Sélectionner uniquement certaines colonnes (toujours garder 'geometry' !)
africa_simple = africa[["NAME", "POP_EST", "geometry"]]
```

## 3. Visualiser : `.plot()`

```python
africa_large.plot(column="POP_EST", legend=True, cmap="YlOrRd", figsize=(9, 7))
```

- `column="POP_EST"` colore chaque pays selon la valeur de cette colonne
  (une carte choroplèthe)
- `cmap="YlOrRd"` choisit la palette de couleurs (jaune → orange → rouge) —
  GeoPandas utilise les palettes de Matplotlib, donc `"viridis"`,
  `"Blues"`, `"terrain"` fonctionnent aussi
- `legend=True` ajoute l'échelle de couleurs

Pour superposer deux couches (par exemple tous les pays en gris clair, puis
l'Afrique en couleur par-dessus) :

```python
fig, ax = plt.subplots(figsize=(10, 8))
world.plot(ax=ax, color="#EDEFE9", edgecolor="#8FA0AE")   # fond : tous les pays
africa_large.plot(ax=ax, column="POP_EST", cmap="YlOrRd", legend=True)
plt.title("Pays africains de plus de 50 millions d'habitants")
plt.axis("off")
plt.show()
```

Le paramètre `ax` est la clé pour combiner plusieurs couches sur une même
carte — c'est le même principe qu'on utilisera plus tard pour construire des
cartes multi-couches plus complexes.

## Sauvegarder le résultat filtré

Une fois le filtre appliqué, on peut réexporter le résultat dans n'importe
quel format géospatial :

```python
africa_large.to_file("afrique_grands_pays.geojson", driver="GeoJSON")
# ou, pour rester en shapefile :
africa_large.to_file("afrique_grands_pays.shp")
```

## Pour aller plus loin

- **Reprojection** : `world.to_crs("EPSG:3857")` change le système de
  coordonnées — indispensable avant de calculer des surfaces ou des
  distances en mètres (le CRS géographique WGS84 travaille en degrés, pas
  en mètres).
- **Jointure spatiale** : `gpd.sjoin()` permet de croiser deux couches
  géographiques (par exemple : quels points de collecte KoboToolbox
  tombent dans quelle commune ?) — sujet d'un prochain article.
- **Calculs géométriques** : `.area`, `.length`, `.centroid`,
  `.buffer(distance)` s'utilisent directement sur la colonne `geometry`.
- **D'autres jeux de données Natural Earth** prêts à l'emploi : villes
  (`ne_110m_populated_places`), rivières, réseaux routiers — même principe
  de chargement par URL, en changeant simplement le nom du fichier dans
  l'URL.

La logique à retenir : un GeoDataFrame se manipule exactement comme un
DataFrame pandas, avec en plus une colonne géométrie qui suit tous les
filtres — c'est ce qui rend GeoPandas aussi rapide à prendre en main pour
qui connaît déjà pandas.
