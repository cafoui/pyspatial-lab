---
title: "Extraire les bâtiments de Cotonou avec Python et les afficher dans un tableau de bord de gestion urbaine"
date: 2026-08-06T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "osm", "overpass", "urbanisme", "web-sig", "geopandas"]
summary: "Extraire l'ensemble des empreintes de bâtiments d'une ville avec Python et OpenStreetMap, les agréger en densité par zone, et les publier dans un tableau de bord de gestion urbaine — à l'échelle d'une ville entière, pas d'un seul quartier."
---

L'article précédent sur la [modélisation 3D d'un quartier](/posts/quartier-3d-anime-threejs/)
travaillait à l'échelle d'un îlot — quelques centaines de bâtiments. À
l'échelle d'une ville entière comme Cotonou, la même approche (extruder
chaque bâtiment en 3D) devient impraticable : des dizaines de milliers de
volumes à charger feraient ramer n'importe quel navigateur. La bonne
échelle pour un **tableau de bord de gestion urbaine** n'est pas le
bâtiment individuel, mais la **densité bâtie par zone** — exactement
l'approche de cet article.

## Voir le résultat

<iframe src="/demos/gestion-urbaine-cotonou.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/gestion-urbaine-cotonou.html).)* Ajuste le curseur de
densité minimale pour isoler les zones les plus construites.

⚠️ **Sur les données** : la grille de densité affichée est **générée pour
illustrer la méthode** (des pôles de densité placés aux emplacements
plausibles de Cotonou — Dantokpa, Akpakpa, Cadjehoun...), pas mesurée sur
un vrai export OpenStreetMap. La suite de l'article donne le code réel
pour la remplacer par de vraies données.

## Pourquoi agréger plutôt qu'afficher bâtiment par bâtiment

Trois raisons concrètes, pas seulement une question de performance :

1. **Volume** : une ville de la taille de Cotonou compte probablement plus
   de 100 000 bâtiments — même en 2D, afficher chaque empreinte individuellement
   ralentit fortement un navigateur.
2. **Lisibilité** : à l'échelle d'une ville entière, l'œil ne distingue de
   toute façon plus les bâtiments un par un — une carte de densité
   communique l'information utile bien plus vite.
3. **C'est ce que font les vrais outils de gestion urbaine** : la plupart
   des SIG municipaux professionnels raisonnent en zones, îlots ou
   grilles statistiques, pas en objets individuels, dès qu'il s'agit de
   piloter une politique à l'échelle de la ville.

## 1. Extraire les bâtiments d'une ville entière avec Overpass

Contrairement au quartier de l'article précédent, une requête Overpass
sur la ville entière risque de dépasser le temps limite du serveur. La
bonne pratique : **découper la zone en tuiles** et interroger chacune
séparément.

```python
import requests
import numpy as np
import time

LAT_MIN, LAT_MAX = 6.320, 6.400
LON_MIN, LON_MAX = 2.370, 2.470
TILE_SIZE = 0.02   # ~2,2 km de côté par tuile

def query_tile(lat0, lon0, lat1, lon1):
    query = f"""
    [out:json][timeout:60];
    way["building"]({lat0},{lon0},{lat1},{lon1});
    out geom;
    """
    r = requests.post("https://overpass-api.de/api/interpreter", data={"data": query})
    r.raise_for_status()
    return r.json()["elements"]

all_buildings = []
lat_tiles = np.arange(LAT_MIN, LAT_MAX, TILE_SIZE)
lon_tiles = np.arange(LON_MIN, LON_MAX, TILE_SIZE)

for lat0 in lat_tiles:
    for lon0 in lon_tiles:
        elements = query_tile(lat0, lon0, lat0+TILE_SIZE, lon0+TILE_SIZE)
        all_buildings.extend(elements)
        time.sleep(1.5)  # laisser respirer le serveur public, partagé par tout le monde

print(f"{len(all_buildings)} bâtiments récupérés")
```

Le `time.sleep(1.5)` entre chaque tuile n'est pas cosmétique : Overpass
est un service public gratuit, partagé par des milliers d'utilisateurs —
l'enchaîner sans pause expose à un blocage temporaire de ton adresse IP.
Pour un usage répété ou une zone très large, héberger sa propre instance
Overpass (ou utiliser [Geofabrik](https://download.geofabrik.de), déjà
présenté dans l'article sur [osm2pgsql](/posts/osm2pgsql-import-postgis/))
reste préférable.

{{< pub slot="1919191919" >}}

## 2. Convertir en GeoDataFrame

```python
import geopandas as gpd
from shapely.geometry import Polygon

def to_geodataframe(elements):
    rows = []
    for way in elements:
        if "geometry" not in way or len(way["geometry"]) < 3:
            continue
        coords = [(pt["lon"], pt["lat"]) for pt in way["geometry"]]
        rows.append({
            "geometry": Polygon(coords),
            "levels": way.get("tags", {}).get("building:levels"),
        })
    return gpd.GeoDataFrame(rows, crs="EPSG:4326")

buildings = to_geodataframe(all_buildings)
print(f"{len(buildings)} empreintes valides")
```

## 3. Agréger en grille de densité

C'est l'étape qui transforme des dizaines de milliers de polygones
individuels en une donnée exploitable à l'échelle de la ville :

```python
import numpy as np

CELL_DEG = 0.0025  # ~275 m de côté, ajustable selon le niveau de détail voulu

buildings["cell_lon"] = (buildings.geometry.centroid.x // CELL_DEG) * CELL_DEG
buildings["cell_lat"] = (buildings.geometry.centroid.y // CELL_DEG) * CELL_DEG

grille = buildings.groupby(["cell_lon", "cell_lat"]).size().reset_index(name="n_buildings")
print(grille.sort_values("n_buildings", ascending=False).head(10))
```

`buildings.geometry.centroid.x // CELL_DEG` arrondit chaque bâtiment à sa
cellule de grille — un `groupby` suffit ensuite à compter combien de
bâtiments tombent dans chaque cellule. C'est littéralement la même
opération qu'un histogramme 2D, appliquée à des coordonnées géographiques.

## 4. Exporter en GeoJSON pour le Web-SIG

```python
import json

features = []
max_count = grille["n_buildings"].max()

for _, row in grille.iterrows():
    lon0, lat0 = row["cell_lon"], row["cell_lat"]
    features.append({
        "type": "Feature",
        "properties": {
            "density": round(row["n_buildings"] / max_count, 3),
            "n_buildings_est": int(row["n_buildings"]),
        },
        "geometry": {
            "type": "Polygon",
            "coordinates": [[
                [lon0, lat0], [lon0+CELL_DEG, lat0],
                [lon0+CELL_DEG, lat0+CELL_DEG], [lon0, lat0+CELL_DEG], [lon0, lat0]
            ]]
        }
    })

with open("cotonou_densite.json", "w") as f:
    json.dump({"type": "FeatureCollection", "features": features,
               "meta": {"total_buildings_est": int(len(buildings))}}, f)
```

Ce fichier a exactement la structure attendue par le tableau de bord
ci-dessus — le déposer dans `data/cotonou_densite.json` (en écrasant le
fichier d'exemple) suffit à remplacer la démonstration par de vraies
données.

## 5. Le tableau de bord (Leaflet)

```js
fetch('./data/cotonou_densite.json')
  .then(r => r.json())
  .then(geojson => {
    geojson.features.forEach(f => {
      L.geoJSON(f, {
        style: { fillColor: colorForDensity(f.properties.density), fillOpacity: 0.75 }
      }).bindPopup(`~${f.properties.n_buildings_est} bâtiments estimés`).addTo(map);
    });
  });
```

Même logique `fetch()` + `L.geoJSON()` que tous les autres dashboards déjà
publiés sur ce blog — la seule différence est qu'on affiche des cellules
de densité plutôt que des points ou des lignes individuelles.

## Pour aller plus loin

- **Croiser avec les limites administratives** : une jointure spatiale
  (`gpd.sjoin`) entre la grille de densité et les limites des
  arrondissements de Cotonou donnerait une densité moyenne **par
  arrondissement** plutôt que par cellule arbitraire — plus parlant pour
  une vraie décision d'urbanisme.
- **Suivre l'évolution dans le temps** : en répétant cette extraction à
  intervalles réguliers (tous les 6 mois, par exemple), la même grille
  permettrait de suivre la progression de l'urbanisation, comme dans
  l'article sur la [dynamique de l'occupation du sol](/posts/occupation-sol-3d-dynamique-natitingou/).
- **Publier en base plutôt qu'en fichier statique** : pour une vraie
  application de gestion urbaine mise à jour régulièrement, stocker cette
  grille dans PostGIS (voir les articles dédiés) plutôt que dans un
  fichier GeoJSON statique permettrait des requêtes dynamiques côté
  serveur.

Ce passage du bâtiment individuel à la grille agrégée est une des
transitions les plus utiles à maîtriser en géomatique urbaine — la bonne
échelle d'analyse n'est pas toujours la plus détaillée, c'est celle qui
répond effectivement à la question posée.
