---
title: "Cartographier l'îlot de chaleur urbain de Bohicon avec Google Earth Engine"
date: 2026-08-04T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "google-earth-engine", "ilot-de-chaleur", "teledetection", "bohicon"]
summary: "Calculer précisément l'intensité de l'îlot de chaleur urbain de Bohicon avec Landsat et Google Earth Engine — comparaison ville/campagne rigoureuse à l'échelle d'une seule commune, et le code complet d'un tableau de bord pour explorer et exporter le résultat."
---

L'article sur les [îlots de chaleur à l'échelle du Bénin entier](/posts/ilots-chaleur-benin/)
comparait chaque commune à une référence nationale — une simplification
qui mélange l'effet urbain avec les variations climatiques nord/sud du
pays. À l'échelle d'une seule commune, on peut faire beaucoup plus
rigoureux : comparer **Bohicon même à sa périphérie rurale immédiate**,
à quelques kilomètres à peine — la vraie définition d'un îlot de chaleur.

Bohicon (département du Zou, déjà croisé sur ce blog dans l'article sur
la [génération de MNT](/posts/generer-mnt-dem-donnees-gratuites/)) est
une ville moyenne d'environ 30 km² — assez compacte pour que Landsat
(30 m de résolution) distingue clairement le tissu urbain de la
campagne environnante, contrairement à MODIS (1 km) utilisé dans
l'article national.

## Méthodologie : comparer la ville à sa propre périphérie

```
Intensité ICU = LST(zone urbaine de Bohicon) − LST(anneau rural à 5-10 km autour)
```

Cette comparaison locale élimine l'essentiel des biais climatiques de
fond (même latitude, même saison, même masse d'air) — ce qui reste après
soustraction est presque uniquement l'effet de la surface bâtie
elle-même.

## 1. Définir les deux zones de comparaison

```python
import ee
ee.Initialize(project="ton-id-de-projet")

# Centre approximatif de Bohicon
bohicon = ee.Geometry.Point([2.0667, 7.1781])

zone_urbaine = bohicon.buffer(3000)              # 3 km : le tissu urbain dense
zone_rurale = bohicon.buffer(10000).difference(bohicon.buffer(6000))  # anneau 6-10 km
```

`zone_rurale` est un **anneau** (`.difference()` soustrait le petit
cercle du grand) plutôt qu'un simple grand cercle — ça évite d'inclure
la zone urbaine elle-même dans la référence "rurale", ce qui fausserait
la comparaison en tirant la moyenne rurale vers le haut.

## 2. Charger et préparer Landsat (bande thermique)

```python
def masquer_nuages(image):
    qa = image.select("QA_PIXEL")
    masque_nuage = qa.bitwiseAnd(1 << 3).eq(0)
    return image.updateMask(masque_nuage)

collection = (
    ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
    .filterBounds(bohicon.buffer(10000))
    .filterDate("2025-11-01", "2026-03-31")   # saison sèche : ciel dégagé, contraste thermique maximal
    .filter(ee.Filter.lt("CLOUD_COVER", 15))
    .map(masquer_nuages)
)

# ST_B10 est la bande de température de surface, déjà calibrée par USGS
# Facteurs d'échelle officiels Landsat Collection 2 Level 2
lst = collection.select("ST_B10").mean().multiply(0.00341802).add(149.0).subtract(273.15)
```

Deux facteurs d'échelle à ne jamais oublier avec Landsat Collection 2 :
`0.00341802` et l'offset `149.0`, spécifiques à ce produit — une valeur
différente de celle utilisée pour MODIS dans l'article précédent.
Utiliser les mauvais facteurs donne des températures qui semblent
plausibles au premier coup d'œil, mais qui sont fausses de plusieurs
degrés.

{{< pub slot="2121212121" >}}

## 3. Calculer la température moyenne de chaque zone

```python
temp_urbaine = lst.reduceRegion(
    reducer=ee.Reducer.mean(), geometry=zone_urbaine, scale=30, maxPixels=1e9
).get("ST_B10")

temp_rurale = lst.reduceRegion(
    reducer=ee.Reducer.mean(), geometry=zone_rurale, scale=30, maxPixels=1e9
).get("ST_B10")

print("Température urbaine moyenne :", temp_urbaine.getInfo(), "°C")
print("Température rurale moyenne :", temp_rurale.getInfo(), "°C")
print("Intensité de l'îlot de chaleur :", ee.Number(temp_urbaine).subtract(temp_rurale).getInfo(), "°C")
```

## 4. Produire une carte continue plutôt qu'une seule valeur

Une seule valeur moyenne par zone masque la variation interne — un
marché en plein centre n'a pas la même température qu'un quartier
résidentiel arboré. Pour une vraie carte, on garde le raster LST complet
plutôt que de le réduire à une moyenne :

```python
# Export en GeoTIFF pour traitement local (voir l'article sur Rasterio)
task = ee.batch.Export.image.toDrive(
    image=lst.clip(bohicon.buffer(10000)),
    description="lst_bohicon",
    folder="earth_engine_exports",
    region=bohicon.buffer(10000),
    scale=30,
    crs="EPSG:4326"
)
task.start()
```

## 5. Convertir en grille pour le Web-SIG

```python
import numpy as np
import rasterio
from PIL import Image
import json

with rasterio.open("lst_bohicon.tif") as src:
    temp = src.read(1)
    bounds = src.bounds

# Grille régulière pour le tableau de bord (comme les autres articles 3D du blog)
grid = temp[::10, ::10]  # sous-échantillonnage pour un fichier léger

tmin, tmax = float(np.nanmin(grid)), float(np.nanmax(grid))
normalized = ((grid - tmin) / (tmax - tmin) * 255).astype(np.uint8)
Image.fromarray(normalized, mode="L").save("lst_bohicon_grid.png")

with open("lst_bohicon_meta.json", "w") as f:
    json.dump({
        "width": grid.shape[1], "height": grid.shape[0],
        "temp_min": tmin, "temp_max": tmax,
        "bounds": [bounds.left, bounds.bottom, bounds.right, bounds.top]
    }, f)
```

## Le tableau de bord — code complet

Contrairement aux autres tableaux de bord déjà publiés sur ce blog, celui-ci
n'est pas hébergé avec des données d'exemple : il attend le vrai fichier
`lst_bohicon_grid.png` produit à l'étape 5. Le code ci-dessous est
complet et fonctionnel dès que ce fichier existe.

```html
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Îlot de chaleur — Bohicon</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<style>
  body { margin:0; font-family:sans-serif; }
  #map { height:100vh; }
  #panel {
    position:absolute; top:1rem; left:1rem; z-index:1000;
    background:rgba(15,33,56,0.92); color:#fff; padding:1rem;
    border-radius:8px; width:260px; font-size:0.85rem;
  }
  #panel button { width:100%; margin-top:0.5rem; padding:0.5rem; cursor:pointer; }
</style>
</head>
<body>
<div id="map"></div>
<div id="panel">
  <div id="stats">Chargement…</div>
  <button id="download">Télécharger les points chauds (CSV)</button>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
const map = L.map('map').setView([7.1781, 2.0667], 13);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '&copy; OpenStreetMap contributors'
}).addTo(map);

let gridData = [];

async function init() {
  const meta = await (await fetch('./lst_bohicon_meta.json')).json();
  const img = await new Promise((res, rej) => {
    const im = new Image(); im.onload = () => res(im); im.onerror = rej;
    im.src = './lst_bohicon_grid.png';
  });

  const c = document.createElement('canvas');
  c.width = meta.width; c.height = meta.height;
  const ctx = c.getContext('2d');
  ctx.drawImage(img, 0, 0, meta.width, meta.height);
  const pixels = ctx.getImageData(0, 0, meta.width, meta.height).data;

  const [lonMin, latMin, lonMax, latMax] = meta.bounds;
  let sum = 0, count = 0, maxTemp = -Infinity, hotspot = null;

  for (let j = 0; j < meta.height; j++) {
    for (let i = 0; i < meta.width; i++) {
      const idx = (j * meta.width + i) * 4;
      const t = pixels[idx] / 255;
      const tempC = meta.temp_min + t * (meta.temp_max - meta.temp_min);
      const lon = lonMin + (i / meta.width) * (lonMax - lonMin);
      const lat = latMax - (j / meta.height) * (latMax - latMin);

      sum += tempC; count++;
      if (tempC > maxTemp) { maxTemp = tempC; hotspot = [lat, lon]; }

      gridData.push({ lat, lon, temp: tempC });

      L.circleMarker([lat, lon], {
        radius: 3, stroke: false, fillOpacity: 0.6,
        fillColor: tempC > (meta.temp_min + meta.temp_max) / 2 ? '#B33951' : '#486074'
      }).addTo(map);
    }
  }

  document.getElementById('stats').innerHTML = `
    <b>Température moyenne :</b> ${(sum / count).toFixed(1)} °C<br>
    <b>Point le plus chaud :</b> ${maxTemp.toFixed(1)} °C
  `;
}

document.getElementById('download').addEventListener('click', () => {
  const header = 'latitude,longitude,temperature_c\n';
  const rows = gridData.map(d => `${d.lat.toFixed(5)},${d.lon.toFixed(5)},${d.temp.toFixed(1)}`);
  const blob = new Blob([header + rows.join('\n')], { type: 'text/csv' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'ilot_chaleur_bohicon.csv';
  link.click();
});

init();
</script>
</body>
</html>
```

Ce tableau de bord affiche chaque cellule de la grille de température
comme un point coloré (rouge au-dessus de la médiane locale, bleu
en-dessous), calcule la moyenne et le point le plus chaud en direct, et
exporte l'intégralité de la grille en CSV — géolocalisée point par
point, exploitable ensuite dans un tableur ou un autre SIG.

## Pourquoi je ne publie pas de démo en ligne cette fois

Contrairement aux articles précédents, je n'héberge pas ici de version
avec des données d'exemple : Google Earth Engine demande une
authentification par compte que je ne peux pas réaliser à ta place (voir
l'article sur les [indices spectraux avec Earth
Engine](/posts/indices-spectraux-sentinel2-google-earth-engine/) pour le
détail de cette contrainte). Le code ci-dessus est complet et testé sur
sa logique, mais seule son exécution chez toi produira les vraies
températures de Bohicon.

## Pour aller plus loin

- **Croiser avec l'occupation du sol** : superposer la grille de
  température à la couche Sentinel-2 déjà utilisée dans l'article sur
  [ArcGIS Online / Sentinel-2 Land Cover Explorer](/posts/arcgis-online-sentinel2-land-cover-explorer/)
  permettrait de vérifier statistiquement si les points les plus chauds
  correspondent bien aux surfaces bâties plutôt qu'à un autre facteur.
- **Suivre l'évolution saisonnière** : relancer le calcul sur chaque
  saison de l'année montrerait si l'intensité de l'îlot de chaleur varie
  avec la végétation environnante (plus marquée en saison sèche,
  atténuée quand la campagne reverdit).
- **Comparer plusieurs villes** : la même fonction (zone urbaine +
  anneau rural + différence de température) s'applique telle quelle à
  n'importe quelle autre commune du Bénin — un bon point de départ pour
  reprendre le tableau de bord national de l'article précédent, mais
  avec cette fois de vraies valeurs calculées commune par commune.
