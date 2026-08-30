---
title: "Comment générer un MNT/DEM (Modèle Numérique de Terrain) à partir de données gratuites"
date: 2026-08-16T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "rasterio", "mnt", "dem", "teledetection"]
summary: "Télécharger et exploiter un Modèle Numérique de Terrain gratuit, sans création de compte, avec Rasterio — visualisation, calcul de pente et d'ombrage, export pour QGIS."
---

Un **MNT** (Modèle Numérique de Terrain, ou *DEM — Digital Elevation
Model* en anglais) est une image où chaque pixel représente une
altitude plutôt qu'une couleur. Il sert de base à énormément d'analyses
spatiales : calcul de pente pour un projet agricole, délimitation de
bassins versants, simulation d'inondation, ombrage pour une carte plus
lisible. Ce tutoriel télécharge un MNT gratuit sur une zone du Bénin,
sans création de compte, et en tire une carte de pente et un ombrage.

## MNT, MNS, MNE : de quoi parle-t-on exactement ?

Avant de choisir une source de données, une nuance importante :

- **MNS** (Modèle Numérique de Surface / *DSM*) : altitude de **tout ce
  qui est visible depuis le ciel** — sol, bâtiments, canopée forestière
  comprise.
- **MNT** (Modèle Numérique de Terrain / *DTM*) : altitude du **sol nu
  uniquement**, bâtiments et végétation retirés par traitement.

Le jeu de données utilisé dans ce tutoriel, **Copernicus DEM GLO-30**, est
techniquement un **MNS** (il inclut la canopée et le bâti) — comme la
grande majorité des MNT gratuits à couverture mondiale. Pour un vrai MNT
"sol nu" sur une zone précise, il faut généralement se tourner vers des
données LiDAR aéroportées, rarement disponibles gratuitement en dehors de
quelques pays (l'IGN français propose par exemple RGE ALTI en accès
libre).

## Où trouver des données gratuites

| Source | Résolution | Compte requis ? | Couverture |
|---|---|---|---|
| **Copernicus DEM GLO-30** (utilisé ici) | 30 m | Non | Mondiale |
| SRTM (via OpenTopography) | 30 m | Oui (gratuit) | Quasi mondiale (60°N–56°S) |
| USGS 3DEP | jusqu'à 1 m | Oui (gratuit) | États-Unis uniquement |
| IGN RGE ALTI | 1 à 5 m | Non | France uniquement |

On utilise ici **Copernicus DEM GLO-30**, accessible via l'API STAC de
Microsoft Planetary Computer — la même méthode déjà utilisée dans
l'article sur le [calcul de NDVI avec Rasterio](/posts/rasterio-ndvi-tutoriel/),
pour la même raison : zéro inscription, lecture directe depuis le cloud.

## Installer les dépendances

```bash
pip install rasterio numpy matplotlib pystac-client planetary-computer
```

## 1. Rechercher la tuile MNT

```python
import pystac_client
import planetary_computer

catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1",
    modifier=planetary_computer.sign_inplace,
)

# Zone d'étude : environs de Bohicon, Bénin
bbox = [1.95, 7.05, 2.20, 7.30]

search = catalog.search(collections=["cop-dem-glo-30"], bbox=bbox)
items = list(search.items())
print(f"{len(items)} tuile(s) trouvée(s)")

item = items[0]
```

Contrairement à la recherche Sentinel-2 de l'article NDVI, il n'y a ici
aucun filtre de nuages à appliquer — un MNT n'est pas une image optique, la
couverture nuageuse n'entre pas en jeu.

## 2. Ouvrir et visualiser le MNT

```python
import rasterio
import matplotlib.pyplot as plt

with rasterio.open(item.assets["data"].href) as src:
    dem = src.read(1).astype("float32")
    profile = src.profile

plt.figure(figsize=(9, 8))
plt.imshow(dem, cmap="terrain")
plt.colorbar(label="Altitude (m)")
plt.title("MNT — région de Bohicon, Bénin")
plt.axis("off")
plt.show()
```

La palette `"terrain"` de Matplotlib est la convention habituelle pour un
MNT : bleu pour les basses altitudes, vert puis brun pour les altitudes
croissantes.

{{< pub slot="9999999999" >}}

## 3. Calculer la pente

La pente en chaque point se calcule à partir du gradient d'altitude — la
variation d'altitude entre pixels voisins :

```python
import numpy as np

# Taille du pixel en degrés (résolution native du MNT), à convertir en mètres
# approximatif à cette latitude — suffisant pour ce calcul illustratif
pixel_size_m = 30

dy, dx = np.gradient(dem, pixel_size_m)
pente_radians = np.arctan(np.sqrt(dx**2 + dy**2))
pente_degres = np.degrees(pente_radians)

plt.figure(figsize=(9, 8))
plt.imshow(pente_degres, cmap="YlOrRd", vmin=0, vmax=30)
plt.colorbar(label="Pente (degrés)")
plt.title("Carte de pente")
plt.axis("off")
plt.show()
```

Pour une carte de pente destinée à un vrai projet (agricole, gestion des
risques), l'outil en ligne de commande **`gdaldem slope`** (fourni avec
GDAL) donne un résultat plus rigoureux, en tenant compte correctement de
la projection :

```bash
gdaldem slope mnt_bohicon.tif pente_bohicon.tif
```

## 4. Générer un ombrage (hillshade)

L'ombrage simule l'éclairage du relief par le soleil — la technique la
plus efficace pour rendre une carte de terrain immédiatement lisible :

```bash
gdaldem hillshade mnt_bohicon.tif ombrage_bohicon.tif -az 315 -alt 45
```

`-az 315` place la source de lumière au nord-ouest (convention
cartographique standard), `-alt 45` fixe la hauteur du soleil à 45°
au-dessus de l'horizon — les valeurs par défaut qui produisent un rendu
naturel dans la plupart des cas.

## 5. Exporter le MNT

```python
profile.update(dtype=rasterio.float32, count=1)

with rasterio.open("mnt_bohicon.tif", "w", **profile) as dst:
    dst.write(dem, 1)
```

## 6. Visualiser dans QGIS

Le fichier `mnt_bohicon.tif` s'ouvre directement dans QGIS
(**Couche → Ajouter une couche → Ajouter une couche raster**). Pour un
rendu en relief immédiatement lisible : clic droit sur la couche →
**Propriétés → Symbologie → Ombrage** (*Hillshade*), qui applique
l'ombrage directement au rendu sans avoir à générer un fichier séparé.

## Fusionner plusieurs tuiles

Une zone d'étude qui déborde sur plusieurs tuiles Copernicus DEM se
fusionne avec `rasterio.merge` :

```python
from rasterio.merge import merge

datasets = [rasterio.open(it.assets["data"].href) for it in items]
mosaique, transform = merge(datasets)
```

## Pour aller plus loin

- **Délimitation de bassins versants** : des bibliothèques comme
  `richdem` ou `pysheds` calculent directement, à partir d'un MNT, le sens
  d'écoulement de l'eau et les bassins versants — la suite naturelle
  d'une carte de pente pour un projet lié à l'eau ou à l'érosion.
- **MNT vs MNS pour un vrai projet** : si le projet exige un sol nu précis
  (pas de canopée ni de bâti), vérifier si un jeu de données LiDAR
  national existe pour la zone d'étude — bien plus précis que les MNT
  mondiaux à 30 m, mais avec une couverture géographique limitée.
- **Publier en Web-SIG** : un MNT converti en tuiles raster (par exemple
  via `gdal2tiles`) peut être superposé à un dashboard Leaflet comme ceux
  déjà publiés sur ce blog, pour ajouter un fond topographique interactif.

Le MNT est une des données de base les plus polyvalentes en géomatique —
une fois téléchargé, il alimente aussi bien une simple carte d'ombrage
qu'une analyse hydrologique complète.
