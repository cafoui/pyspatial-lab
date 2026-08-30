---
title: "Traiter des images satellite avec Rasterio — calcul de NDVI (indice de végétation) pas à pas"
date: 2026-08-21T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "rasterio", "ndvi", "teledetection", "sentinel-2"]
summary: "Calculer un indice de végétation (NDVI) à partir d'une vraie image Sentinel-2, sans créer de compte ni télécharger de fichier volumineux — avec Rasterio et l'API gratuite de Microsoft Planetary Computer."
---

Le **NDVI** (Normalized Difference Vegetation Index) est l'indice le plus
utilisé en télédétection pour mesurer la densité de végétation à partir
d'une image satellite — il sert aussi bien au suivi de la déforestation
qu'au monitoring agricole. Ce tutoriel calcule un NDVI de bout en bout avec
**Rasterio**, à partir d'une vraie image **Sentinel-2**, sans jamais avoir à
créer de compte ni télécharger un fichier de plusieurs gigaoctets.

## Le principe du NDVI

```
NDVI = (NIR - RED) / (NIR + RED)
```

où `RED` est la réflectance dans le rouge et `NIR` la réflectance dans le
proche infrarouge. La végétation en bonne santé réfléchit fortement le
proche infrarouge et absorbe le rouge (pour la photosynthèse) — plus cette
différence est marquée, plus le NDVI est élevé, entre **-1** et **+1**.

| Valeur NDVI | Interprétation |
|---|---|
| < 0 | Eau, nuages, neige |
| 0 – 0,2 | Sol nu, zone urbaine, roche |
| 0,2 – 0,4 | Végétation clairsemée (herbe, cultures jeunes) |
| 0,4 – 1 | Végétation dense (forêt, cultures matures) |

## Pourquoi Microsoft Planetary Computer plutôt que Copernicus ou EarthExplorer

La plupart des tutoriels NDVI demandent de créer un compte (Copernicus
Data Space, USGS EarthExplorer) puis de télécharger une archive de
plusieurs centaines de mégaoctets avant même de commencer. On utilise ici
l'**API STAC gratuite de Microsoft Planetary Computer** : aucune
inscription, et Rasterio peut lire directement les bandes nécessaires
**depuis le cloud**, sans rien télécharger en entier — seule la zone qui
nous intéresse est transférée.

## Installer les dépendances

```bash
pip install rasterio numpy matplotlib pystac-client planetary-computer
```

## 1. Rechercher une image Sentinel-2 sans nuage

```python
import pystac_client
import planetary_computer

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1",
    modifier=planetary_computer.sign_inplace,
)

# Zone d'étude : environs de Cotonou, Bénin [lon_min, lat_min, lon_max, lat_max]
bbox = [2.20, 6.30, 2.50, 6.50]

search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=bbox,
    datetime="2026-01-01/2026-03-31",
    query={"eo:cloud_cover": {"lt": 10}},
)

items = list(search.items())
print(f"{len(items)} image(s) trouvée(s)")

item = items[0]
print(item.datetime, "— nuages :", item.properties["eo:cloud_cover"], "%")
```

Le filtre `"eo:cloud_cover": {"lt": 10}` ne garde que les scènes couvertes
à moins de 10 % par les nuages — indispensable pour un NDVI exploitable,
un pixel nuageux n'a aucune signification en végétation.

## 2. Lire les bandes rouge et proche infrarouge

Sur Sentinel-2, la bande **B04** correspond au rouge et **B08** au proche
infrarouge (résolution 10 m pour les deux).

```python
import rasterio
import numpy as np

red_href = item.assets["B04"].href
nir_href = item.assets["B08"].href

with rasterio.open(red_href) as src:
    red = src.read(1).astype("float32")
    profile = src.profile   # on garde les métadonnées pour l'export final

with rasterio.open(nir_href) as src:
    nir = src.read(1).astype("float32")
```

`planetary_computer.sign_inplace` a déjà signé les URLs des bandes lors de
la recherche — Rasterio peut donc les ouvrir directement en HTTPS, comme
s'il s'agissait de fichiers locaux, grâce au support GDAL des flux réseau.

## 3. Calculer le NDVI

```python
np.seterr(divide="ignore", invalid="ignore")   # ignorer les avertissements 0/0

ndvi = (nir - red) / (nir + red)
ndvi = np.clip(ndvi, -1, 1)   # sécurité : borner strictement entre -1 et 1
```

La division par zéro se produit sur les pixels où `nir + red = 0`
(généralement des zones sans données) — `np.seterr` évite juste
l'avertissement dans la console, le résultat `NaN` reste correct et sera
géré naturellement par la visualisation et l'export.

## 4. Visualiser le résultat

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(9, 9))
plt.imshow(ndvi, cmap="RdYlGn", vmin=-1, vmax=1)
plt.colorbar(label="NDVI")
plt.title(f"NDVI — Cotonou, {item.datetime.date()}")
plt.axis("off")
plt.show()
```

La palette `"RdYlGn"` (rouge → jaune → vert) est la convention standard
pour représenter le NDVI : rouge pour le sol nu ou l'eau, vert pour la
végétation dense.

{{< pub slot="6666666666" >}}

## 5. Exporter le résultat en GeoTIFF

Pour réutiliser ce NDVI dans QGIS, PostGIS, ou un Web-SIG, on l'exporte en
GeoTIFF géoréférencé, en réutilisant le `profile` (CRS, transformation
géographique) capturé à l'étape 2 :

```python
profile.update(dtype=rasterio.float32, count=1, nodata=None)

with rasterio.open("ndvi_cotonou.tif", "w", **profile) as dst:
    dst.write(ndvi, 1)
```

Le fichier obtenu s'ouvre directement dans QGIS avec le bon géoréférencement
— aucune manipulation supplémentaire nécessaire.

## Le script complet

```python
import pystac_client
import planetary_computer
import rasterio
import numpy as np
import matplotlib.pyplot as plt

# 1. Recherche
catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1",
    modifier=planetary_computer.sign_inplace,
)
bbox = [2.20, 6.30, 2.50, 6.50]
search = catalog.search(
    collections=["sentinel-2-l2a"],
    bbox=bbox,
    datetime="2026-01-01/2026-03-31",
    query={"eo:cloud_cover": {"lt": 10}},
)
item = list(search.items())[0]

# 2. Lecture des bandes
with rasterio.open(item.assets["B04"].href) as src:
    red = src.read(1).astype("float32")
    profile = src.profile
with rasterio.open(item.assets["B08"].href) as src:
    nir = src.read(1).astype("float32")

# 3. Calcul du NDVI
np.seterr(divide="ignore", invalid="ignore")
ndvi = np.clip((nir - red) / (nir + red), -1, 1)

# 4. Visualisation
plt.imshow(ndvi, cmap="RdYlGn", vmin=-1, vmax=1)
plt.colorbar(label="NDVI")
plt.axis("off")
plt.savefig("ndvi_apercu.png", dpi=150, bbox_inches="tight")

# 5. Export géoréférencé
profile.update(dtype=rasterio.float32, count=1, nodata=None)
with rasterio.open("ndvi_cotonou.tif", "w", **profile) as dst:
    dst.write(ndvi, 1)
```

## Pour aller plus loin

- **Suivi temporel** : relancer la recherche STAC sur plusieurs fenêtres
  de dates (une par mois, par exemple) et empiler les NDVI obtenus permet
  de suivre l'évolution de la végétation dans le temps — utile pour
  détecter une déforestation ou une saison agricole.
- **D'autres indices** : le même principe s'applique au **NDWI**
  (indice d'eau, bandes verte et NIR) ou au **NDBI** (indice de zones
  bâties, bandes SWIR et NIR) — seule la formule et les bandes changent.
- **Classification par seuils** : transformer le NDVI continu en classes
  discrètes (`np.digitize` ou `np.select`) pour produire une carte
  "végétation dense / clairsemée / sol nu" plus simple à lire.
- **Publier sur un Web-SIG** : reprojeter le GeoTIFF en Web Mercator
  (`gdalwarp` ou `rasterio.warp.reproject`) et le convertir en PNG géoréférencé
  permet de le superposer directement sur une carte Leaflet, avec la même
  approche que les dashboards déjà publiés sur ce blog.

Le NDVI n'est qu'un point de départ : une fois qu'on sait lire deux bandes
et faire une opération pixel par pixel avec NumPy, la même mécanique
s'applique à des dizaines d'autres indices de télédétection.
