---
title: "Étudier la dynamique de l'occupation du sol en la drapant sur un relief 3D (2017 vs 2025)"
date: 2026-08-08T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "threejs", "occupation-du-sol", "sentinel-2", "3d", "teledetection"]
summary: "Draper deux cartes d'occupation du sol (2017 et 2025) sur le relief 3D de Natitingou pour voir où et comment le paysage a changé — avec une reprojection UTM faite de zéro, sans GDAL."
---

Le [relief 3D de Natitingou](/posts/modelisation-3d-relief-python-threejs/)
construit précédemment ne montrait que l'altitude. Cet article y drape
deux cartes réelles d'**occupation du sol** (Sentinel-2, 10 m de
résolution, 2017 et 2025) pour voir directement, sur le relief, où la
végétation a reculé, où le bâti a progressé — un changement d'année
souvent difficile à percevoir sur une carte plate devient immédiatement
lisible une fois posé sur le vrai terrain.

## Voir le résultat

<iframe src="/demos/occupation-sol-3d-natitingou.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/occupation-sol-3d-natitingou.html).)* Bascule entre
2017, 2025 et le mode "Changements" pour comparer, ou repasse en mode
Altitude pour retrouver le relief seul.

## Ce que révèlent les vraies données

Sur cette zone, **26,2 % des cellules ont changé de classe** entre 2017 et
2025 :

| Classe | 2017 | 2025 | Évolution |
|---|---|---|---|
| Arbres | 24,8 % | 8,5 % | **-16,3 points** |
| Cultures | 5,2 % | 0,5 % | -4,7 points |
| Surface bâtie | 1,8 % | 2,6 % | +0,8 point |
| Prairies | 68,1 % | 88,4 % | **+20,3 points** |

Le recul du couvert arboré est net, avec une conversion apparente
massive vers la classe "Prairies". ⚠️ **Cette lecture appelle une vraie
prudence méthodologique** : une partie de ce changement reflète
probablement une évolution réelle du paysage (déforestation, extension
agricole puis mise en jachère), mais une partie peut aussi venir de
**la confusion classique entre "Arbres" et "Prairies"** dans les
classifications automatiques Sentinel-2 d'une année sur l'autre — la
végétation arbustive et la savane arborée typique de l'Atacora se
classent de façon instable selon la saison et la qualité des images
disponibles cette année-là. Le chiffre exact ne doit pas être pris au
pied de la lettre ; la tendance générale (recul de la classe "Arbres"),
répétée sur plusieurs années, serait un signal plus solide qu'une seule
comparaison à deux dates.

## Le vrai défi technique : deux systèmes de coordonnées différents

Le MNT (article précédent) est en **coordonnées géographiques** (WGS84,
degrés). Les rasters d'occupation du sol téléchargés depuis Living Atlas
sont en revanche en **UTM zone 31N** (mètres) — la projection standard
pour les données Sentinel-2 à cette longitude. Il faut donc convertir
chaque cellule du MNT vers son équivalent en mètres UTM avant de pouvoir
lire la bonne valeur dans les rasters d'occupation du sol.

{{< pub slot="1717171717" >}}

## 1. Reprojeter sans GDAL ni pyproj

GDAL et pyproj n'étaient pas disponibles dans l'environnement utilisé pour
cet article. Plutôt qu'une approximation, la projection **UTM directe**
(formules de Snyder) a été implémentée en NumPy pur :

```python
import numpy as np

a = 6378137.0                    # rayon équatorial WGS84
f = 1/298.257223563               # aplatissement WGS84
e2 = f*(2-f)
ep2 = e2/(1-e2)
k0 = 0.9996                       # facteur d'échelle UTM
lon0 = np.radians(3.0)            # méridien central, zone 31N
FE = 500000.0                     # fausse origine Est

def latlon_to_utm(lat_deg, lon_deg):
    phi, lam = np.radians(lat_deg), np.radians(lon_deg)
    N = a / np.sqrt(1 - e2*np.sin(phi)**2)
    T, C = np.tan(phi)**2, ep2*np.cos(phi)**2
    A = np.cos(phi)*(lam - lon0)
    M = a*((1 - e2/4 - 3*e2**2/64 - 5*e2**3/256)*phi
           - (3*e2/8 + 3*e2**2/32 + 45*e2**3/1024)*np.sin(2*phi)
           + (15*e2**2/256 + 45*e2**3/1024)*np.sin(4*phi)
           - (35*e2**3/3072)*np.sin(6*phi))
    x = FE + k0*N*(A + (1-T+C)*A**3/6 + (5-18*T+T**2+72*C-58*ep2)*A**5/120)
    y = k0*(M + N*np.tan(phi)*(A**2/2 + (5-T+9*C+4*C**2)*A**4/24
             + (61-58*T+T**2+600*C-330*ep2)*A**6/720))
    return x, y
```

C'est la même formule que celle utilisée en interne par PROJ/GDAL pour ce
type de conversion — implémenter la formule directement reste tout à fait
défendable pour un usage ponctuel, mais `pyproj` reste recommandé pour un
usage répété ou des zones proches des limites de validité de la
projection.

En appliquant cette fonction à chaque cellule de la grille du MNT
(latitude/longitude), on obtient sa position exacte en mètres UTM — donc
la ligne et la colonne correspondantes dans les rasters d'occupation du
sol :

```python
tie_x, tie_y = 297110.0, 1163480.0   # origine (coin haut-gauche) du raster OCS
res = 10.0                            # résolution du raster, en mètres

x_utm, y_utm = latlon_to_utm(lat_grid, lon_grid)
col = ((x_utm - tie_x) / res).astype(int)
row = ((tie_y - y_utm) / res).astype(int)
```

Résultat sur cette zone : **99,5 % des cellules du MNT retombent bien
dans l'emprise du raster OCS** — une confirmation que les deux jeux de
données couvrent bien la même zone, et que la reprojection est correcte.

## 2. Coloriser selon la légende officielle

```python
COLORS = {
    1:  (26,91,171),    # Eau
    2:  (53,130,33),    # Arbres
    4:  (135,209,158),  # Végétation inondée
    5:  (255,219,92),   # Cultures
    7:  (237,2,42),     # Surface bâtie
    8:  (237,233,228),  # Terrain nu
    11: (198,173,141),  # Prairies
}

sampled = ocs_raster[row, col]           # échantillonnage au plus proche voisin
rgb = np.zeros((*sampled.shape, 3), dtype=np.uint8)
for code, color in COLORS.items():
    rgb[sampled == code] = color
```

Ces couleurs sont celles de la légende officielle Esri pour ce jeu de
données — les reprendre exactement permet de rester cohérent avec
d'autres cartes utilisant la même palette (notamment celle du
[Sentinel-2 Land Cover Explorer](/posts/arcgis-online-sentinel2-land-cover-explorer/)
déjà présenté sur ce blog).

## 3. Calculer un masque de changement

```python
both_valid = (classes_2017 != 0) & (classes_2025 != 0)
changed = both_valid & (classes_2017 != classes_2025)

change_rgb = np.where(changed[...,None], [255,60,120], [40,55,70]).astype(np.uint8)
```

Un simple test d'égalité pixel par pixel — mais seulement sur les
cellules valides dans les deux dates, pour ne jamais confondre "a changé"
et "était en dehors de la zone couverte une des deux années".

## 4. Draper la texture sur le maillage 3D

Le maillage du relief 3D (voir l'article précédent) a été construit sur
la **même grille** que ces cartes d'occupation du sol — chaque sommet
correspond exactement à un pixel de la texture colorisée. Changer
l'affichage revient donc simplement à remplacer les couleurs des sommets,
sans recalculer la géométrie :

```js
function setColorSet(colorArray) {
  terrainMesh.geometry.attributes.color.set(colorArray);
  terrainMesh.geometry.attributes.color.needsUpdate = true;
}
```

C'est ce qui rend la bascule 2017 / 2025 / Changements instantanée dans le
visualiseur — aucun rechargement, juste un nouveau tableau de couleurs
appliqué au même relief.

## Pour aller plus loin

- **Plus de deux dates** : Living Atlas propose ce jeu de données pour
  chaque année de 2017 à 2025 — animer la succession complète donnerait
  une vraie vidéo de l'évolution du paysage plutôt qu'une comparaison à
  deux points.
- **Croiser avec la pente** : superposer le masque de changement à une
  carte de pente (voir l'article sur le [calcul de pente et
  d'ombrage](/posts/generer-mnt-dem-donnees-gratuites/)) permettrait de
  vérifier si le recul du couvert arboré est plus marqué sur les pentes
  fortes (érosion) ou les zones planes (extension agricole).
- **Valider le signal** : croiser ce changement de classification avec un
  NDVI calculé sur les mêmes deux dates (voir l'article dédié) donnerait
  un second indicateur indépendant — si les deux s'accordent, la
  confiance dans un vrai changement (plutôt qu'un artefact de
  classification) augmente nettement.

Ce projet illustre une idée simple mais souvent négligée : une carte
d'occupation du sol et un MNT sont rarement dans le même système de
coordonnées dès qu'ils viennent de sources différentes — la reprojection
n'est pas une formalité optionnelle, c'est l'étape qui rend la
comparaison possible.
