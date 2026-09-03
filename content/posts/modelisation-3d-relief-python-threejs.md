---
title: "Modélisation 3D du relief avec Python et un MNT — rendu interactif sur un Web-SIG"
date: 2026-08-10T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "mnt", "3d", "threejs", "web-sig", "webgl"]
summary: "Transformer un MNT en relief 3D interactif consultable dans le navigateur — préparation des données côté Python, rendu WebGL avec Three.js, et un avertissement honnête sur l'exagération verticale."
---

Un MNT (voir l'article sur [comment en générer un gratuitement](/posts/generer-mnt-dem-donnees-gratuites/))
n'est pas fait que pour du calcul — c'est aussi la matière première d'un
relief 3D navigable dans un navigateur, sans plugin ni logiciel installé.
Ce tutoriel part d'un vrai MNT de la **chaîne de l'Atacora, autour de
Natitingou** (nord du Bénin) — la région la plus reliefée du pays — et
construit un visualiseur 3D interactif avec **Three.js**.

## Voir le résultat

<iframe src="/demos/relief-3d-natitingou.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/relief-3d-natitingou.html).)* Fais glisser pour orbiter
autour du relief, molette pour zoomer, et ajuste le curseur d'exagération
verticale dans le panneau — c'est le réglage le plus important de tout ce
tutoriel, expliqué plus bas.

## Le problème que personne ne prévient : le relief réel reste discret

Sur cette zone pourtant montagneuse pour le Bénin, l'altitude varie de
**191 m à 671 m** sur environ **61,5 km** de large — un dénivelé de 480 m
sur une telle distance reste proportionnellement modeste (moins de 1 %).
Rendu en 3D à l'échelle réelle, ce relief serait beaucoup moins spectaculaire
qu'on ne l'imagine en regardant une carte topographique colorée. **Toute
visualisation 3D de terrain applique donc une exagération verticale** —
multiplier l'altitude par un facteur (10×, 20×, 30×...) pour rendre le
relief perceptible, même sur la région la plus accidentée du pays.

Ce n'est pas un problème en soi — c'est une pratique standard en
cartographie du relief. Le problème serait de **ne jamais le préciser** :
un relief 3D sans indication d'exagération peut donner une impression
totalement fausse du terrain réel à quelqu'un qui ne connaît pas la
région. Le visualiseur ci-dessus affiche en permanence le facteur choisi,
justement pour éviter ce piège.

## 1. Préparer le MNT côté Python

Avec `rasterio` (l'outil recommandé — voir la note en fin d'article sur
l'alternative utilisée pour cette démonstration précise) :

```python
import rasterio
import numpy as np
from scipy import ndimage

with rasterio.open("mnt_natitingou.tif") as src:
    dem = src.read(1).astype("float64")
    nodata = src.nodata

valid = dem != nodata
```

## 2. Combler les zones sans données

Un MNT réel contient presque toujours des trous (bord de couverture,
artefacts du capteur). Pour un rendu 3D, il suffit de les combler avec la
valeur valide la plus proche — inutile d'une méthode plus sophistiquée,
l'objectif est visuel, pas analytique :

```python
indices = ndimage.distance_transform_edt(~valid, return_distances=False, return_indices=True)
dem_filled = dem[tuple(indices)]
```

## 3. Sous-échantillonner pour le web

Un MNT complet (souvent plusieurs millions de pixels) est inutilement
lourd pour un rendu WebGL fluide dans un navigateur — quelques centaines
de points suffisent largement à donner une impression de relief crédible :

```python
H, W = dem_filled.shape
factor = 6
target_h, target_w = H // factor, W // factor
reduced = ndimage.zoom(dem_filled, (target_h / H, target_w / W), order=1)
```

Un facteur 6 fait passer un MNT de plus de 850 000 pixels à environ
24 000 — largement suffisant pour un rendu fluide, y compris sur un
téléphone.

{{< pub slot="1515151515" >}}

## 4. Exporter en heightmap PNG

Plutôt que d'envoyer un tableau de nombres au navigateur, on encode
l'altitude dans une image en niveaux de gris — un format standard pour
échanger un relief avec un moteur 3D, beaucoup plus compact qu'un JSON de
coordonnées :

```python
from PIL import Image
import json

zmin, zmax = reduced.min(), reduced.max()
normalized = ((reduced - zmin) / (zmax - zmin) * 255).astype("uint8")

Image.fromarray(normalized, mode="L").save("heightmap.png")

with open("heightmap_meta.json", "w") as f:
    json.dump({"width": target_w, "height": target_h, "elev_min": float(zmin), "elev_max": float(zmax)}, f)
```

Le fichier `heightmap_meta.json` est indispensable : le PNG seul ne
contient que des valeurs de 0 à 255, il faut les bornes réelles
(`elev_min`, `elev_max`) pour reconstruire les vraies altitudes en mètres
côté navigateur.

## 5. Construire le relief 3D avec Three.js

Le principe : charger l'image, lire chaque pixel comme une hauteur, et
déplacer chaque sommet d'un maillage plan en conséquence.

```js
import * as THREE from 'three';

const geometry = new THREE.PlaneGeometry(100, 61.5, gridW - 1, gridH - 1);
geometry.rotateX(-Math.PI / 2);   // le plan devient horizontal

const pos = geometry.attributes.position;
for (let j = 0; j < gridH; j++) {
  for (let i = 0; i < gridW; i++) {
    const idx = j * gridW + i;
    const t = heights[idx];                     // 0..1, lu depuis le PNG
    const elevM = elevMin + t * (elevMax - elevMin);
    pos.setY(idx, elevM * exaggeration * unitsPerMeter);
  }
}
geometry.computeVertexNormals();   // indispensable pour un éclairage correct
```

`computeVertexNormals()` est l'étape la plus facile à oublier — sans elle,
l'éclairage du relief reste plat et incohérent, même si la géométrie 3D
est correcte.

## 6. Colorer par altitude (teinte hypsométrique)

```js
function colorForElevation(t) {
  // t entre 0 et 1 -> dégradé bas/moyen/haut
  const stops = [
    [0.00, [92, 122, 102]],   // vert (bas)
    [0.55, [192, 138, 62]],   // brun (moyen)
    [1.00, [237, 239, 233]],  // clair (haut)
  ];
  // interpolation linéaire entre les paliers les plus proches...
}
```

Cette teinte hypsométrique (vert → brun → clair) est la convention
cartographique la plus reconnaissable pour un relief — bien plus lisible
qu'une couleur unique, surtout sur un relief aussi subtil que celui-ci.

## 7. Naviguer avec OrbitControls

```js
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;   // un mouvement plus fluide, moins mécanique
```

`OrbitControls` gère automatiquement rotation (glisser), zoom (molette) et
translation (clic droit + glisser) — la navigation standard de tout
visualiseur 3D web, sans rien coder à la main.

## Pour aller plus loin

- **Superposer une texture réelle** : au lieu d'une teinte hypsométrique
  générée par code, une image satellite (voir l'article sur
  [Sentinel-2 Land Cover Explorer](/posts/arcgis-online-sentinel2-land-cover-explorer/))
  plaquée sur le même maillage donnerait un rendu bien plus réaliste.
- **Animer le relief** : faire varier l'exagération verticale
  progressivement au chargement (de 1× à la valeur cible) donne un effet
  d'apparition du relief assez efficace pour une page d'accueil.
- **MNT + réseau hydrographique** : superposer le réseau vectoriel de
  l'article sur les [écoulements d'eau](/posts/modelisation-ecoulements-eau-richdem-webgis/)
  directement sur ce relief 3D, en surélevant légèrement les lignes
  au-dessus du maillage pour éviter qu'elles ne soient masquées par le
  terrain.
- **Exporter en glTF** : pour réutiliser ce relief dans un logiciel de
  modélisation 3D (Blender) ou un moteur de jeu, exporter le maillage
  Three.js au format glTF plutôt que de le reconstruire à chaque fois côté
  navigateur.

> **Note technique sur cet article** : la préparation des données
> ci-dessus utilise `rasterio`, l'outil recommandé pour un usage normal.
> Cette démonstration précise a été produite avec `tifffile` + `scipy` +
> `Pillow` à la place, `rasterio` n'étant pas disponible dans
> l'environnement utilisé pour cet article — le résultat final (la
> heightmap PNG) est strictement identique quelle que soit la méthode
> utilisée pour y arriver.

Un relief 3D bien construit n'est jamais qu'une heightmap et un maillage
déformé — la vraie difficulté, comme souvent en cartographie, est de
rester honnête sur ce que la mise en scène (ici, l'exagération verticale)
change par rapport à la réalité du terrain.
