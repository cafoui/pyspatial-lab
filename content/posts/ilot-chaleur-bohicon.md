---
title: "Cartographier l'îlot de chaleur urbain de Bohicon avec Google Earth Engine"
date: 2026-08-04T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "google-earth-engine", "ilot-de-chaleur", "teledetection", "landsat"]
summary: "Mesurer l'intensité réelle de l'îlot de chaleur urbain de Bohicon avec Landsat et Google Earth Engine : +1,46 °C mesurés sur 5 images satellite — méthode complète, résultats et tableau de bord interactif avec export CSV."
---

Un **îlot de chaleur urbain** (ICU) est l'écart de température entre une
zone bâtie et la campagne environnante : le bitume et la tôle
emmagasinent la chaleur du jour, la végétation la dissipe par
évapotranspiration. Le mesurer demande de comparer la **température de
surface** (LST — *Land Surface Temperature*), captée par les capteurs
thermiques des satellites, entre la ville et sa périphérie immédiate.

Cet article mesure l'îlot de chaleur de **Bohicon** (département du Zou)
à partir de vraies images Landsat, avec Google Earth Engine — et publie
le résultat dans un tableau de bord permettant d'explorer la carte
thermique et d'exporter les données.

## Le résultat : +1,46 °C

<iframe src="/demos/ilot-chaleur-bohicon.html" width="100%" height="640" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/ilot-chaleur-bohicon.html).)*

| Mesure | Valeur |
|---|---|
| Température moyenne — zone urbaine (0–3 km) | **40,18 °C** |
| Température moyenne — anneau rural (6–10 km) | **38,72 °C** |
| **Intensité de l'îlot de chaleur** | **+1,46 °C** |
| Amplitude sur l'ensemble de la zone | 32,2 °C à 47,6 °C |
| Images Landsat utilisées | 5 (saison sèche 2025-2026) |

Ces chiffres sont issus d'un vrai calcul sur des images satellite, pas
d'une estimation. Un écart de **+1,46 °C** est modéré comparé aux grandes
métropoles (où l'ICU dépasse souvent 3 à 5 °C), ce qui est cohérent avec
une ville moyenne d'environ 30 km², au bâti peu dense et encore
largement entrecoupé de végétation.

L'amplitude interne est bien plus parlante que la moyenne : **15 °C
d'écart** entre le point le plus frais et le point le plus chaud de la
zone. C'est cette variation locale — visible sur la carte du tableau de
bord — qui intéresse vraiment l'aménageur, bien plus que la moyenne
globale.

## Méthodologie : comparer la ville à sa propre périphérie

```
Intensité ICU = LST(zone urbaine) − LST(anneau rural immédiat)
```

Comparer Bohicon à un anneau situé à 6–10 km seulement — même latitude,
même saison, même masse d'air — élimine l'essentiel des biais
climatiques de fond. Ce qui reste après soustraction est presque
uniquement l'effet de la surface bâtie.

```python
import ee
ee.Initialize(project="ton-id-de-projet")

bohicon = ee.Geometry.Point([2.0667, 7.1781])

zone_urbaine = bohicon.buffer(3000)
zone_rurale = bohicon.buffer(10000).difference(bohicon.buffer(6000))
```

`zone_rurale` est un **anneau** (`.difference()` soustrait le petit
cercle du grand), pas un simple grand cercle — sinon la zone urbaine
serait incluse dans sa propre référence rurale, ce qui écraserait
artificiellement l'écart mesuré.

## 1. Charger Landsat et masquer les nuages

```python
def masquer_nuages(image):
    qa = image.select("QA_PIXEL")
    masque = qa.bitwiseAnd(1 << 3).eq(0).And(qa.bitwiseAnd(1 << 4).eq(0))
    return image.updateMask(masque)

collection = (
    ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
    .merge(ee.ImageCollection("LANDSAT/LC09/C02/T1_L2"))
    .filterBounds(bohicon.buffer(10000))
    .filterDate("2025-11-01", "2026-03-31")
    .filter(ee.Filter.lt("CLOUD_COVER", 20))
    .map(masquer_nuages)
)
```

Trois choix qui comptent :

- **Landsat plutôt que MODIS** : 30 m de résolution contre 1 km. À
  l'échelle d'une ville moyenne comme Bohicon, MODIS ne distinguerait
  tout simplement pas le tissu urbain de la campagne — quelques pixels
  couvriraient toute la ville.
- **Landsat 8 *et* 9 fusionnés** (`.merge()`) : deux satellites
  identiques en orbite décalée, ce qui double les chances de trouver des
  images sans nuages. Ici, 5 images exploitables au lieu de 2 ou 3.
- **La saison sèche** (novembre à mars) : ciel dégagé, et contraste
  thermique maximal entre ville et campagne — en saison des pluies, la
  campagne reverdit et l'écart s'atténue fortement.

{{< pub slot="2121212121" >}}

## 2. Convertir en degrés Celsius

```python
lst = collection.select("ST_B10").mean().multiply(0.00341802).add(149.0).subtract(273.15)
```

Les facteurs `0.00341802` et l'offset `149.0` sont **spécifiques au
produit Landsat Collection 2 Level 2** — ce ne sont pas les mêmes que
pour MODIS. Se tromper de facteurs donne des températures qui semblent
plausibles au premier regard mais sont fausses de plusieurs degrés :
une erreur silencieuse, donc particulièrement traître.

Le `.subtract(273.15)` convertit ensuite des Kelvin en Celsius.

## 3. Calculer les moyennes de chaque zone

```python
def moyenne_zone(geom):
    return lst.reduceRegion(
        reducer=ee.Reducer.mean(), geometry=geom, scale=30, maxPixels=1e9
    ).get("ST_B10").getInfo()

t_urbain = moyenne_zone(zone_urbaine)   # 40.18 °C
t_rural = moyenne_zone(zone_rurale)     # 38.72 °C
print(f"Intensité ICU : +{t_urbain - t_rural:.2f} °C")
```

## 4. Télécharger la carte thermique complète

Une moyenne par zone masque toute la variation interne — un marché en
plein centre n'a pas la même température qu'un quartier arboré. Pour une
vraie carte, on récupère le raster complet :

```python
import requests

url = lst.clip(bohicon.buffer(10000)).getDownloadURL({
    "scale": 30, "crs": "EPSG:4326",
    "region": bohicon.buffer(10000), "format": "GEO_TIFF"
})
with open("lst_bohicon.tif", "wb") as f:
    f.write(requests.get(url, timeout=300).content)
```

`getDownloadURL` télécharge directement, contrairement à
`ee.batch.Export.image.toDrive` qui passe par Google Drive en mode
asynchrone — plus simple pour une zone de cette taille.

## 5. Préparer la grille pour le Web-SIG

```python
import numpy as np, rasterio, json
from PIL import Image

with rasterio.open("lst_bohicon.tif") as src:
    temp = src.read(1).astype("float64")
    bounds = src.bounds

temp[temp == 0] = np.nan
step = max(1, temp.shape[1] // 150)
grid = temp[::step, ::step]
grid = np.where(np.isnan(grid), np.nanmean(grid), grid)

tmin, tmax = float(np.nanmin(grid)), float(np.nanmax(grid))
normalized = ((grid - tmin) / (tmax - tmin) * 255).astype(np.uint8)
Image.fromarray(normalized, mode="L").save("lst_bohicon_grid.png")

with open("lst_bohicon_meta.json", "w") as f:
    json.dump({
        "width": int(grid.shape[1]), "height": int(grid.shape[0]),
        "temp_min": tmin, "temp_max": tmax,
        "bounds": [bounds.left, bounds.bottom, bounds.right, bounds.top],
        "intensite_icu": round(t_urbain - t_rural, 2),
    }, f)
```

Encoder la température dans une image en niveaux de gris — la même
technique de heightmap déjà utilisée sur ce blog pour les MNT — reste le
moyen le plus compact de transmettre une grille de valeurs au navigateur.
Le JSON associé porte les bornes réelles, indispensables pour
reconstituer les vraies températures côté client.

## 6. Un piège à la lecture : les coins de la grille

Le raster exporté est un **rectangle**, alors que la zone analysée est un
**cercle** de 10 km de rayon. Les coins du rectangle, hors du cercle, ont
été comblés par la valeur moyenne à l'étape précédente — les afficher
tels quels donnerait de fausses températures sur le pourtour.

Le tableau de bord les masque explicitement :

```js
function haversineKm(lat1, lon1, lat2, lon2) {
  const R = 6371, toRad = d => d*Math.PI/180;
  const dLat = toRad(lat2-lat1), dLon = toRad(lon2-lon1);
  const a = Math.sin(dLat/2)**2 + Math.cos(toRad(lat1))*Math.cos(toRad(lat2))*Math.sin(dLon/2)**2;
  return 2*R*Math.asin(Math.sqrt(a));
}

// à la lecture de chaque pixel :
if (haversineKm(CENTER[0], CENTER[1], lat, lon) > 10) continue;
```

C'est le genre de détail facile à négliger, et qui produirait pourtant
une carte visuellement crédible mais partiellement fausse.

## 7. L'export CSV, côté navigateur

```js
function downloadCSV(rows, filename) {
  const header = 'latitude,longitude,temperature_c\n';
  const body = rows.map(p => `${p.lat.toFixed(5)},${p.lon.toFixed(5)},${p.temp.toFixed(2)}`).join('\n');
  const blob = new Blob([header + body], { type:'text/csv;charset=utf-8;' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = filename;
  link.click();
}
```

`Blob` + `URL.createObjectURL` construit le fichier à la volée dans le
navigateur, sans aucun serveur — le tableau de bord propose ainsi
d'exporter soit la grille complète, soit uniquement les points au-dessus
du seuil de température réglé par l'utilisateur, pour analyser ensuite
les zones les plus chaudes dans un tableur ou un autre SIG.

## Limites de cette mesure

Trois réserves à garder en tête avant d'utiliser ces chiffres :

1. **Température de surface ≠ température de l'air.** Landsat mesure ce
   que "voit" le capteur thermique : la surface du sol, des toits, du
   bitume. L'air à hauteur d'homme est généralement plus frais et moins
   contrasté — un ICU mesuré par satellite est systématiquement plus
   marqué que celui mesuré par des stations météo au sol.
2. **Une moyenne de 5 images n'est pas un climat.** Ces images couvrent
   une seule saison sèche ; une vraie étude s'appuierait sur plusieurs
   années pour lisser les variations d'une année à l'autre.
3. **La délimitation urbain/rural est géométrique, pas réelle.** Les
   rayons de 3 km et 6–10 km sont des approximations ; croiser avec une
   vraie couche d'occupation du sol donnerait une séparation plus fidèle
   au tissu bâti réel.

## Pour aller plus loin

- **Croiser avec l'occupation du sol** : superposer la grille de
  température à une couche Sentinel-2 permettrait de vérifier
  statistiquement que les points les plus chauds correspondent bien aux
  surfaces bâties, et pas à un sol nu agricole.
- **Suivre l'évolution saisonnière** : relancer le calcul en saison des
  pluies montrerait de combien l'écart s'atténue quand la campagne
  reverdit.
- **Étendre à d'autres communes** : la même fonction (zone urbaine +
  anneau rural + différence) s'applique telle quelle à n'importe quelle
  ville — Parakou, Abomey-Calavi, Porto-Novo — pour construire une
  comparaison inter-urbaine à l'échelle du pays.
