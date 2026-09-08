---
title: "Modélisation 3D d'un quartier urbain avec Python et Three.js — Web-SIG animé (cycle jour/nuit, circulation)"
date: 2026-08-07T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "threejs", "osm", "3d", "urbanisme", "web-sig"]
summary: "Extruder des empreintes de bâtiments en volumes 3D pour modéliser un quartier urbain, avec un cycle jour/nuit animé et de la circulation — et comment brancher de vraies données OpenStreetMap à la place de l'exemple illustratif."
---

Les articles 3D précédents de ce blog modélisaient un relief naturel. La
même logique de base — géométrie 2D + attribut de hauteur → volume 3D —
s'applique tout aussi bien à un **quartier urbain** : chaque bâtiment
devient une empreinte au sol extrudée à la hauteur voulue. Ce tutoriel
construit un quartier 3D animé (cycle jour/nuit, circulation) et explique
comment remplacer l'exemple par de vrais bâtiments OpenStreetMap.

## Voir le résultat

<iframe src="/demos/quartier-3d-anime.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/quartier-3d-anime.html).)* Le cycle jour/nuit tourne
en continu (curseur de vitesse dans le panneau), les fenêtres s'allument
la nuit, et des véhicules circulent sur le réseau de rues.

⚠️ **Sur les données** : le quartier affiché est **généré pour illustrer
la technique**, pas un relevé réel de Cotonou — grille de rues, parcelles
et hauteurs de bâtiments plausibles, mais fictives. La section finale
explique précisément comment le remplacer par de vrais bâtiments
OpenStreetMap.

## Le principe : extrusion d'empreintes

Un bâtiment, vu du dessus, est un polygone (son empreinte au sol). Lui
donner du volume ne demande qu'une opération : **l'extrusion** — répéter
ce contour à toutes les hauteurs entre 0 et la hauteur du bâtiment, puis
fermer le haut et le bas. Three.js fait exactement ça avec
`ExtrudeGeometry`.

```js
const shape = new THREE.Shape();
building.footprint.forEach(([x, z], i) => {
  if (i === 0) shape.moveTo(x, z); else shape.lineTo(x, z);
});
shape.closePath();

const geometry = new THREE.ExtrudeGeometry(shape, { depth: building.height, bevelEnabled: false });
geometry.rotateX(-Math.PI / 2);   // l'extrusion se fait sur Z -> on la redresse sur Y (la hauteur)
```

`ExtrudeGeometry` extrude par défaut le long de l'axe Z — d'où la
rotation de -90° après coup, pour que la hauteur pointe bien vers le haut
(Y) plutôt que vers l'avant de la scène.

## 1. Le modèle de données d'un bâtiment

```json
{
  "footprint": [[12.4, 8.1], [24.6, 8.1], [24.6, 19.3], [12.4, 19.3]],
  "height": 9.6,
  "floors": 3,
  "type": "residentiel"
}
```

`footprint` est une liste de points `[x, z]` en mètres, dans un repère
local (pas des coordonnées GPS directement) — c'est ce repère local,
centré sur le quartier, que Three.js utilise pour positionner tout le
monde de façon cohérente.

{{< pub slot="1818181818" >}}

## 2. D'où vient la hauteur, avec de vraies données OSM

OpenStreetMap ne donne quasiment jamais une empreinte de bâtiment sans
information de hauteur exploitable — mais sous deux formes différentes
selon les contributeurs :

- Le tag `height` (en mètres, directement utilisable)
- Le tag `building:levels` (nombre d'étages — à multiplier par une
  hauteur d'étage estimée, 3 à 3,5 m selon le contexte)

```python
def estimer_hauteur(tags):
    if "height" in tags:
        return float(tags["height"].replace("m", "").strip())
    if "building:levels" in tags:
        return float(tags["building:levels"]) * 3.2
    return 3.2  # valeur par défaut : un seul niveau
```

De nombreux bâtiments n'ont ni l'un ni l'autre — une valeur par défaut
raisonnable (un seul niveau) reste nécessaire pour ne pas les exclure de
la scène.

## 3. Récupérer de vrais bâtiments avec l'API Overpass

```python
import requests

query = """
[out:json][timeout:25];
(
  way["building"](6.360,2.425,6.370,2.440);
);
out geom;
"""
response = requests.post("https://overpass-api.de/api/interpreter", data={"data": query})
data = response.json()
```

Les quatre nombres `(6.360,2.425,6.370,2.440)` définissent la zone
d'intérêt (`lat_min, lon_min, lat_max, lon_max`) — ici un quartier de
Cotonou, à ajuster pour n'importe quelle ville. `out geom` demande à
Overpass de renvoyer directement la géométrie de chaque bâtiment, pas
seulement ses identifiants.

## 4. Convertir en repère local (mètres, centré sur le quartier)

Les coordonnées OSM sont en latitude/longitude — il faut les convertir en
mètres, centrées sur le quartier, avant de les donner à Three.js :

```python
import numpy as np

lat0 = np.mean([pt["lat"] for way in data["elements"] for pt in way["geometry"]])
lon0 = np.mean([pt["lon"] for way in data["elements"] for pt in way["geometry"]])

def to_local_xy(lat, lon):
    x = (lon - lon0) * 111320 * np.cos(np.radians(lat0))
    z = (lat - lat0) * 111320
    return x, z

buildings = []
for way in data["elements"]:
    if "geometry" not in way:
        continue
    footprint = [to_local_xy(pt["lat"], pt["lon"]) for pt in way["geometry"]]
    height = estimer_hauteur(way.get("tags", {}))
    buildings.append({"footprint": footprint, "height": height, "type": "residentiel"})
```

Cette projection locale (équirectangulaire simplifiée) est suffisamment
précise pour un quartier de quelques centaines de mètres — inutile d'une
vraie projection UTM à cette échelle, contrairement à l'article sur
l'[occupation du sol en 3D](/posts/occupation-sol-3d-dynamique-natitingou/)
qui couvrait une zone bien plus large.

## 5. Le cycle jour/nuit

```js
function skyColorAt(t) {
  // t: 0..1 sur 24h — un dégradé de couleurs à des moments clés
  const stops = [
    [0.00,[8,14,30]],   // minuit
    [0.27,[214,140,90]], // lever du soleil
    [0.50,[120,175,225]], // midi
    [0.73,[214,120,80]], // coucher du soleil
    [1.00,[8,14,30]],   // minuit
  ];
  // interpolation entre les paliers les plus proches...
}

const angle = dayT * Math.PI * 2 - Math.PI/2;
sun.position.set(Math.cos(angle)*300, Math.max(Math.sin(angle),0.05)*300, 120);
sun.intensity = 0.15 + Math.max(Math.sin(angle), 0) * 1.3;
```

Le soleil est une simple `DirectionalLight` dont la position orbite en
cercle — son intensité suit le sinus de sa hauteur, nulle (ou presque)
sous l'horizon, maximale à midi. C'est ce qui fait automatiquement
varier les ombres portées des bâtiments au fil du cycle, sans code
supplémentaire.

## 6. Les fenêtres qui s'allument la nuit

```js
const isNight = sunHeight < 0.08;
buildingMeshes.forEach(m => {
  m.material.emissiveIntensity = isNight ? 0.35 : 0;
});
```

Plutôt que de modéliser des fenêtres individuelles (beaucoup plus de
géométrie, pour un effet marginal à cette échelle), chaque bâtiment
utilise sa propre couleur comme couleur émissive la nuit — un raccourci
visuel efficace, qui donne une impression de vie sans complexifier la
scène.

## 7. La circulation

```js
function updateVehicle(v) {
  v.t += v.speed * 0.01;
  if (v.t > 1) v.t -= 1;
  const [x1,z1] = v.street.path[0], [x2,z2] = v.street.path[1];
  const x = x1 + (x2-x1)*v.t, z = z1 + (z2-z1)*v.t;
  v.mesh.position.set(x, 1.1, z);
}
```

Chaque véhicule avance simplement le long d'un segment de rue
(interpolation linéaire), boucle une fois arrivé au bout, et repart —
aucune détection de collision ni règle de circulation, volontairement :
l'objectif est une impression de vie dans la scène, pas une simulation de
trafic.

## Pour aller plus loin

- **Un vrai socle topographique sous les bâtiments** : la version en
  ligne de ce visualiseur utilise désormais un vrai MNT de Cotonou
  (Copernicus GLO-30, obtenu via l'API OpenTopography — mesures radar
  satellite réelles, mission TanDEM-X). Sur la fenêtre utilisée
  (360 m de large), le relief varie de **2,5 m à 8,5 m** — cohérent avec
  une ville côtière très plate. ⚠️ Honnêteté nécessaire : ce MNT réel
  vient d'un point du littoral de Cotonou choisi arbitrairement, **il ne
  correspond pas géographiquement** au quartier fictif qui y est posé
  (celui-ci reste un quartier généré, sans adresse réelle) — le socle
  sert à donner une idée juste de la topographie locale, pas une
  localisation précise.

```python
from PIL import Image
import numpy as np
from scipy import ndimage

dem = np.array(Image.open("cotonou_dem.tif")).astype(np.float64)
H, W = dem.shape

# Découper une fenêtre à l'échelle réelle du quartier (ici ~360 m,
# soit 12 pixels à 30 m de résolution) plutôt que d'étirer tout le
# MNT sur la zone — étirer aurait faussé l'amplitude du relief
half = 6
cy, cx = H//2, W//2
window = dem[cy-half:cy+half, cx-half:cx+half]

# Lissage doux pour l'affichage, sans prétendre ajouter du vrai détail
smooth = ndimage.zoom(window, 80/window.shape[0], order=3)
```

- **Vraie extraction depuis un shapefile local** : si les bâtiments
  viennent d'un shapefile plutôt que d'Overpass, GeoPandas
  (`gpd.read_file(...)`) remplace directement l'appel à l'API — le reste
  du pipeline (projection locale, extrusion) ne change pas.
- **Toits en pente** : `ExtrudeGeometry` ne produit que des toits plats ;
  un vrai toit à deux pans demanderait de construire la géométrie du toit
  séparément et de l'assembler au volume extrudé du bâtiment.
- **Densité de circulation réaliste** : faire varier le nombre de
  véhicules actifs selon l'heure simulée (peu la nuit, pic aux heures de
  pointe) rendrait le cycle jour/nuit encore plus parlant.
- **Étude d'ensoleillement réelle** : ce même principe (soleil qui orbite,
  ombres portées calculées automatiquement par Three.js) est la base des
  études d'ensoleillement utilisées en urbanisme pour vérifier l'accès
  à la lumière naturelle d'un projet — un vrai cas d'usage professionnel
  au-delà de l'aspect purement visuel.

Extruder des empreintes 2D en volumes 3D est une technique simple, mais
qui ouvre la porte à énormément d'analyses urbaines une fois combinée à
de vraies données — ombres portées, densité bâtie, visibilité depuis un
point donné.
