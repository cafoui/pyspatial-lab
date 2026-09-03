---
title: "Modélisation et visualisation Web-SIG des écoulements d'eau avec Python (GeoPandas, RichDEM & Leaflet)"
date: 2026-08-11T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "richdem", "pysheds", "hydrologie", "geopandas", "web-sig"]
summary: "Extraire un réseau hydrographique et un bassin versant à partir d'un MNT avec RichDEM et pysheds, puis les publier sur une carte Leaflet interactive — la chaîne complète, du relief brut jusqu'au Web-SIG."
---

Un MNT — déjà téléchargé dans un [article précédent](/posts/generer-mnt-dem-donnees-gratuites/)
— ne se limite pas à l'ombrage et au calcul de pente. Il permet aussi de
**simuler où l'eau s'écoule** : quelle direction elle prend à chaque
pixel, où elle s'accumule, et quel territoire draine vers un point donné
(le bassin versant). Ce tutoriel construit cette chaîne complète, du
relief brut jusqu'à une carte Web-SIG interactive.

## Voir le résultat

Le Web-SIG final affiche un réseau hydrographique coloré selon l'**ordre
de Strahler** (plus le trait est épais, plus le cours d'eau est
important), superposé à son bassin versant, avec des couches activables
indépendamment et les statistiques de superficie et de longueur totale.

<iframe src="/demos/reseau-hydrographique.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/reseau-hydrographique.html).)*

## Le principe, en trois étapes

1. **Combler les dépressions** du MNT — un relief brut contient souvent
   des creux artificiels (bruit du capteur, résolution) qui bloqueraient
   artificiellement l'écoulement simulé.
2. **Calculer la direction d'écoulement** — pour chaque pixel, vers lequel
   de ses 8 voisins l'eau s'écoulerait (algorithme D8, le plus courant).
3. **Accumuler les flux** — compter, pour chaque pixel, combien de pixels
   en amont s'écoulent vers lui. Un flux accumulé élevé signale un cours
   d'eau ; un flux faible, un simple ruissellement diffus.

## Installer les dépendances

```bash
pip install richdem pysheds geopandas rasterio
```

Le tutoriel combine deux bibliothèques complémentaires : **RichDEM** pour
le remplissage des dépressions et le calcul d'accumulation de flux (rapide,
optimisé), et **pysheds** pour la délimitation de bassin versant et
l'extraction directe du réseau en géométries vectorielles — un point où
pysheds propose des fonctions prêtes à l'emploi plus pratiques que RichDEM.

## 1. Combler les dépressions et calculer l'accumulation avec RichDEM

```python
import richdem as rd

dem = rd.LoadGDAL("mnt_bohicon.tif")   # le MNT de l'article précédent

rd.FillDepressions(dem, epsilon=False, in_place=True)

accumulation = rd.FlowAccumulation(dem, method='D8')

rd.SaveGDAL("accumulation_bohicon.tif", accumulation)
```

`FillDepressions` modifie le MNT en place pour garantir qu'aucun pixel
n'est un cul-de-sac artificiel. `FlowAccumulation` avec la méthode
**D8** (chaque pixel s'écoule vers un seul de ses 8 voisins, celui de plus
forte pente) est l'algorithme le plus simple et le plus répandu — des
méthodes plus fines existent (D-infinity, MFD) mais D8 suffit largement
pour un premier réseau hydrographique.

{{< pub slot="1414141414" >}}

## 2. Délimiter le bassin versant avec pysheds

```python
from pysheds.grid import Grid

grid = Grid.from_raster("mnt_bohicon.tif")
dem_ps = grid.read_raster("mnt_bohicon.tif")

pit_filled = grid.fill_pits(dem_ps)
flooded = grid.fill_depressions(pit_filled)
inflated = grid.resolve_flats(flooded)

fdir = grid.flowdir(inflated)
acc = grid.accumulation(fdir)

# Coordonnées approximatives de l'exutoire (le point en aval duquel
# on veut connaître tout le territoire qui s'y draine)
x, y = 2.095, 7.285

# "Accrocher" l'exutoire au cours d'eau le plus proche dans les données
# (évite de rater le réseau si le point cliqué est à quelques mètres du lit réel)
x_snap, y_snap = grid.snap_to_mask(acc > 500, (x, y))

bassin = grid.catchment(x=x_snap, y=y_snap, fdir=fdir, xytype='coordinate')
```

`snap_to_mask` est l'étape la plus facile à oublier, et la plus
importante : un exutoire cliqué à la main tombe rarement exactement sur
le pixel du cours d'eau dans la donnée raster — sans ce recalage, le
bassin versant calculé peut être complètement erroné.

## 3. Extraire le réseau hydrographique en vecteur

```python
reseau = grid.extract_river_network(fdir, acc > 500)
```

`extract_river_network` renvoie directement une structure GeoJSON — pas
besoin de vectoriser un raster à la main. Le seuil `acc > 500` détermine
à partir de combien de pixels amont on considère qu'il y a un cours d'eau
plutôt qu'un simple ruissellement — à ajuster selon la résolution du MNT
et la densité de réseau souhaitée (un seuil plus bas produit un réseau
plus dense, y compris de petits rus).

## 4. Convertir en GeoDataFrame et exporter

```python
import geopandas as gpd
from shapely.geometry import shape

# Le bassin versant (masque booléen) -> polygone vectoriel
from rasterio.features import shapes as rio_shapes

bassin_shapes = [
    shape(geom) for geom, val in rio_shapes(bassin.astype('uint8'), transform=grid.affine)
    if val == 1
]
gdf_bassin = gpd.GeoDataFrame(geometry=bassin_shapes, crs="EPSG:4326")
gdf_bassin["superficie_km2"] = gdf_bassin.to_crs(epsg=32631).area / 1_000_000

# Le réseau hydrographique (déjà en GeoJSON) -> GeoDataFrame
gdf_reseau = gpd.GeoDataFrame.from_features(reseau["features"], crs="EPSG:4326")

gdf_bassin.to_file("bassin_versant.geojson", driver="GeoJSON")
gdf_reseau.to_file("reseau_hydrographique.geojson", driver="GeoJSON")
```

`to_crs(epsg=32631)` reprojette temporairement en UTM zone 31N (adapté au
Bénin) avant de calculer une superficie en mètres carrés — le même
réflexe que dans l'article GeoPandas : jamais de calcul de surface
directement en degrés.

## 5. Calculer l'ordre de Strahler (optionnel, mais utile pour le rendu)

L'**ordre de Strahler** hiérarchise le réseau : un cours d'eau sans
affluent est d'ordre 1 ; quand deux cours du même ordre se rejoignent,
l'ordre augmente d'un ; sinon, le plus grand des deux ordres est conservé.
Une implémentation complète est plus longue que ce tutoriel ne peut en
donner, mais le principe permet de comprendre pourquoi le réseau affiché
sur la carte ci-dessus varie en épaisseur — c'est ce qui rend une carte
hydrographique lisible d'un coup d'œil plutôt qu'un enchevêtrement de
lignes uniformes.

## Publier sur le Web-SIG

Une fois `bassin_versant.geojson` et `reseau_hydrographique.geojson`
produits, ils remplacent directement le fichier d'exemple de la carte
ci-dessus (`reseau-hydrographique-bohicon.geojson`) — même structure de
propriétés (`type`, `strahler_order`, `longueur_km`, `superficie_km2`), à
adapter selon ta nomenclature si tu es reparti d'un nouveau calcul.

```js
fetch('./data/reseau_hydrographique.geojson')
  .then(res => res.json())
  .then(geojson => {
    L.geoJSON(geojson, {
      style: f => styleCoursEau(f.properties.strahler_order)
    }).addTo(map);
  });
```

C'est la même logique `fetch()` + `L.geoJSON()` que tous les autres
dashboards déjà publiés sur ce blog — seule la donnée en amont change,
produite ici par un vrai calcul hydrologique plutôt que directement
téléchargée.

## Pour aller plus loin

- **Comparer D8 et D-infinity** : `richdem.FlowAccumulation(dem,
  method='Dinf')` répartit le flux entre plusieurs voisins plutôt qu'un
  seul — un réseau plus réaliste sur un terrain peu marqué, mais plus
  coûteux à calculer.
- **Croiser avec l'occupation du sol** : superposer le bassin versant
  calculé ici à une couche d'occupation du sol (voir l'article sur
  [ArcGIS Online / Sentinel-2 Land Cover Explorer](/posts/arcgis-online-sentinel2-land-cover-explorer/))
  permet d'estimer, par exemple, la part de surfaces imperméabilisées
  dans un bassin versant — un indicateur classique de risque
  d'inondation.
- **Publier en WMS** : un réseau hydrographique volumineux (plusieurs
  bassins versants d'un pays entier) gagnerait à être publié via
  GeoServer (voir l'article dédié) plutôt que chargé entièrement en
  GeoJSON statique.

La modélisation hydrologique est un des domaines où le passage du MNT
brut à une carte compréhensible demande le plus d'étapes intermédiaires
— mais chacune, prise séparément, reste une fonction bien documentée de
RichDEM ou pysheds.
