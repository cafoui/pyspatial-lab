---
title: "Calculer différents indices spectraux avec des images Sentinel-2 gratuites dans Google Earth Engine"
date: 2026-08-14T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "google-earth-engine", "sentinel-2", "indices-spectraux", "teledetection"]
summary: "NDVI, NDWI, NDBI, SAVI, EVI : calculer plusieurs indices spectraux sur des images Sentinel-2 directement dans Google Earth Engine, sans rien télécharger, avec l'API Python."
---

L'article sur [Rasterio et le NDVI](/posts/rasterio-ndvi-tutoriel/) calculait
un indice sur une seule image, téléchargée par bandes. **Google Earth
Engine (GEE)** change d'échelle : le calcul se fait **directement sur les
serveurs de Google**, sur des pétaoctets d'archives satellite, sans jamais
télécharger une image entière — utile dès qu'on veut comparer plusieurs
indices, plusieurs dates, ou de grandes zones.

## Google Earth Engine, en bref

GEE combine un catalogue public d'imagerie satellite (Landsat, Sentinel,
MODIS, et bien d'autres) avec un moteur de calcul distribué : on écrit
une expression décrivant le traitement voulu, GEE l'exécute sur ses
serveurs et ne renvoie que le résultat. C'est ce qui permet, par exemple,
de calculer un NDVI moyen sur un pays entier en quelques secondes, sans
jamais manipuler un fichier de plusieurs gigaoctets en local.

## Prérequis : un compte et un projet Google Cloud

Contrairement à l'approche Microsoft Planetary Computer de l'article
précédent, **Earth Engine demande une inscription** — gratuite pour un
usage non commercial, mais réelle. Depuis fin 2024, Google exige que tout
usage d'Earth Engine soit rattaché à un **projet Google Cloud** :

1. Se rendre sur la [page d'inscription Earth Engine](https://code.earthengine.google.com/register)
2. Créer (ou choisir) un projet Google Cloud
3. Sélectionner le tier **"Community"** (gratuit, sans carte bancaire ni
   compte de facturation, suffisant pour ce tutoriel)

Aucune carte bancaire n'est demandée pour ce tier — mais l'étape
d'inscription elle-même est incontournable, contrairement à la méthode
Planetary Computer déjà utilisée sur ce blog.

## Installer et authentifier l'API Python

```bash
pip install earthengine-api geemap
```

```python
import ee

ee.Authenticate()   # ouvre une fenêtre de connexion Google, une seule fois
ee.Initialize(project="ton-id-de-projet")   # remplacer par l'ID de ton projet Cloud
```

`ee.Authenticate()` n'est nécessaire qu'une fois par machine — les
exécutions suivantes réutilisent le jeton enregistré localement.

## 1. Charger et filtrer une collection Sentinel-2

```python
bbox = ee.Geometry.Rectangle([2.20, 6.30, 2.50, 6.50])  # environs de Cotonou

collection = (
    ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
    .filterBounds(bbox)
    .filterDate("2026-01-01", "2026-03-31")
    .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 10))
)

print("Nombre d'images trouvées :", collection.size().getInfo())

image = collection.median()   # composite médian, réduit le bruit résiduel
```

`.median()` combine toutes les images filtrées en une seule, pixel par
pixel — une façon simple d'obtenir une image propre même si aucune date
unique n'est parfaitement sans nuage sur toute la zone.

## 2. Calculer plusieurs indices

GEE fournit une méthode dédiée, `.normalizedDifference()`, pour tout
indice de la forme `(A - B) / (A + B)` :

```python
ndvi = image.normalizedDifference(["B8", "B4"]).rename("NDVI")   # végétation
ndwi = image.normalizedDifference(["B3", "B8"]).rename("NDWI")   # eau
ndbi = image.normalizedDifference(["B11", "B8"]).rename("NDBI")  # bâti
```

| Indice | Bandes utilisées | Détecte |
|---|---|---|
| NDVI | NIR (B8), Rouge (B4) | Végétation |
| NDWI | Vert (B3), NIR (B8) | Surfaces en eau |
| NDBI | SWIR1 (B11), NIR (B8) | Zones bâties |

Pour un indice à la formule plus complexe, `.expression()` permet
d'écrire directement la formule mathématique :

```python
# SAVI (Soil-Adjusted Vegetation Index) — corrige l'influence du sol nu
savi = image.expression(
    "((NIR - RED) / (NIR + RED + 0.5)) * 1.5",
    {"NIR": image.select("B8"), "RED": image.select("B4")}
).rename("SAVI")

# EVI (Enhanced Vegetation Index) — moins sensible à la saturation en forêt dense
evi = image.expression(
    "2.5 * ((NIR - RED) / (NIR + 6 * RED - 7.5 * BLUE + 1))",
    {
        "NIR": image.select("B8"),
        "RED": image.select("B4"),
        "BLUE": image.select("B2"),
    }
).rename("EVI")
```

{{< pub slot="1212121212" >}}

## 3. Visualiser avec geemap

`geemap` affiche les résultats sur une carte interactive directement dans
un notebook Jupyter — sans exporter la moindre image :

```python
import geemap

Map = geemap.Map(center=[6.40, 2.35], zoom=10)

vis_ndvi = {"min": -1, "max": 1, "palette": ["#B33951", "#EDEFE9", "#5C7A66"]}
vis_ndwi = {"min": -1, "max": 1, "palette": ["#EDEFE9", "#486074"]}

Map.addLayer(ndvi, vis_ndvi, "NDVI")
Map.addLayer(ndwi, vis_ndwi, "NDWI")
Map.addLayer(ndbi, {"min": -1, "max": 1, "palette": ["#EDEFE9", "#C08A3E"]}, "NDBI")

Map
```

Chaque couche peut être activée/désactivée indépendamment dans le panneau
de contrôle généré par `geemap` — pratique pour comparer visuellement
plusieurs indices sur la même zone.

## 4. Exporter le résultat

Pour récupérer un indice en GeoTIFF, utilisable ensuite dans QGIS ou
Rasterio :

```python
task = ee.batch.Export.image.toDrive(
    image=ndvi,
    description="ndvi_cotonou",
    folder="earth_engine_exports",
    region=bbox,
    scale=10,
    crs="EPSG:4326"
)
task.start()
```

L'export s'exécute de façon asynchrone sur les serveurs Google — le
fichier apparaît dans le Google Drive du compte utilisé, généralement en
quelques minutes selon la taille de la zone.

## Earth Engine ou Rasterio/Planetary Computer : lequel choisir ?

| Critère | Earth Engine | Rasterio + Planetary Computer |
|---|---|---|
| Inscription requise | Oui (gratuite, non-commercial) | Non |
| Calcul | Côté serveur (cloud) | En local, sur ta machine |
| Idéal pour | Grandes zones, séries temporelles longues, comparaisons multiples | Une image ponctuelle, contrôle fin du traitement |
| Export de fichiers | Asynchrone, via Drive/Cloud Storage | Immédiat, en local |
| Écosystème Python | API dédiée (`earthengine-api`) | Bibliothèques standards (rasterio, numpy) |

**En résumé** : pour une analyse ponctuelle sur une image, l'approche
Rasterio déjà publiée sur ce blog reste plus simple et plus rapide à
mettre en place. Earth Engine devient nettement plus avantageux dès qu'il
s'agit de comparer plusieurs dates, de couvrir de grandes surfaces, ou de
répéter un calcul sur des années d'archives.

## Pour aller plus loin

- **Séries temporelles** : `ImageCollection.map()` applique le calcul
  d'indice à chaque image d'une collection plutôt qu'à un seul composite
  — la base pour suivre l'évolution d'un indice dans le temps.
- **Earth Engine Apps** : un script peut être publié comme mini
  application web interactive, partageable par simple lien, sans que le
  destinataire ait besoin d'un compte Earth Engine.
- **Classification supervisée** : GEE inclut des algorithmes de
  classification (Random Forest, SVM) directement utilisables sur les
  images de la collection, pour aller au-delà des indices simples vers
  une vraie carte d'occupation du sol.

Entre Rasterio pour le contrôle fin en local et Earth Engine pour le
calcul à grande échelle, les deux approches se complètent plus qu'elles
ne s'opposent — le choix dépend surtout de la taille de la zone et du
nombre de dates à traiter.
