---
title: "Simuler la pluie et le ruissellement sur un relief 3D avec Python et Three.js"
date: 2026-08-09T10:00:00+01:00
draft: false
categories: ["Python"]
tags: ["python", "threejs", "hydrologie", "3d", "webgl", "mnt"]
summary: "Faire tomber la pluie sur un vrai relief 3D et la faire ruisseler vers l'aval, en combinant le champ d'écoulement calculé en Python avec une scène Three.js animée — ciel, gouttes et réseau hydrographique."
---

Les deux articles précédents ont construit les deux moitiés de ce projet
séparément : un [relief 3D interactif](/posts/modelisation-3d-relief-python-threejs/)
de la chaîne de l'Atacora, et un [champ d'écoulement calculé en Python](/posts/modelisation-ecoulements-eau-richdem-webgis/)
à partir d'un MNT. Cet article les assemble : de la pluie tombe du ciel,
touche le relief, puis **suit exactement le chemin que l'eau emprunterait
réellement** — parce que sa direction, à chaque instant, vient du même
calcul de direction d'écoulement (D8) que celui utilisé pour extraire un
réseau hydrographique.

## Voir le résultat

<iframe src="/demos/pluie-ruissellement-natitingou.html" width="100%" height="600" style="border:0; border-radius:8px;"></iframe>

*(Si l'aperçu ne s'affiche pas dans ton lecteur de flux, [ouvre-le
directement](/demos/pluie-ruissellement-natitingou.html).)* Les gouttes
claires tombent depuis le ciel ; une fois qu'elles touchent le relief,
elles s'assombrissent et se mettent à ruisseler le long de la pente la
plus forte — exactement le principe D8 déjà expliqué sur ce blog, mais
visible en mouvement plutôt que comme une carte statique.

## Le principe : une seule source de vérité pour la direction d'écoulement

Le piège évident de ce genre de projet serait de faire tomber des
particules "au hasard" vers le bas avec un effet visuel vaguement
convaincant, sans lien avec le vrai relief. Ici, la direction que suit
chaque goutte au sol vient d'une **grille de direction D8 précalculée en
Python**, exactement la même méthode que dans l'article sur les
écoulements d'eau (remplissage des dépressions, résolution des plateaux,
direction de plus forte pente) — simplement appliquée à la résolution
plus légère utilisée pour le rendu 3D, et exportée comme une image que le
navigateur peut lire directement.

## 1. Calculer le champ d'écoulement (rappel condensé)

```python
import numpy as np
from scipy import ndimage
import heapq

# ... remplissage des dépressions, résolution des plateaux,
# direction D8 : voir l'article dédié pour le détail complet ...

# Le résultat : flowdir, une grille où chaque cellule contient un code
# de 0 à 7 (direction parmi les 8 voisins) ou -1 (aucun écoulement,
# bord du domaine).
```

Le point important pour cet article : ce calcul tourne sur la **grille
déjà sous-échantillonnée** pour le relief 3D (quelques centaines de
cellules de large), pas sur le MNT complet — largement suffisant pour un
rendu visuel, et rapide à calculer (moins de 2 secondes ici).

## 2. Exporter la direction d'écoulement comme une image

```python
from PIL import Image

# flowdir contient des valeurs de -1 à 7 ; on décale pour rester en 8 bits
dir_encoded = np.where(flowdir == -1, 8, flowdir).astype(np.uint8)
Image.fromarray(dir_encoded, mode="L").save("flowdir.png")
```

Exactement le même principe que la heightmap de l'article précédent :
encoder une grille de valeurs dans une image en niveaux de gris, pour que
le navigateur la lise sans backend ni traitement supplémentaire.

{{< pub slot="1616161616" >}}

## 3. Lire la grille de direction côté navigateur

```js
function readChannel(img, w, h) {
  const c = document.createElement('canvas');
  c.width = w; c.height = h;
  const ctx = c.getContext('2d');
  ctx.drawImage(img, 0, 0, w, h);
  const data = ctx.getImageData(0, 0, w, h).data;
  const arr = new Uint8Array(w * h);
  for (let i = 0; i < w*h; i++) arr[i] = data[i*4];  // canal rouge = valeur
  return arr;
}
```

Dessiner l'image sur un `<canvas>` invisible puis lire ses pixels est la
technique la plus simple pour faire passer une grille de données de
Python au navigateur sans bibliothèque supplémentaire — déjà utilisée
pour la heightmap, réutilisée ici à l'identique pour la direction
d'écoulement.

## 4. Faire tomber la pluie

Chaque goutte est un point dans un `THREE.Points`, avec un état
(`falling` ou `flowing`) suivi séparément en JavaScript :

```js
function stepRain(dt) {
  for (let k = 0; k < activeCount; k++) {
    if (dropFalling[k]) {
      positions[k*3+1] -= 42 * dt;                    // chute
      const groundY = terrainY[dropJ[k]*gridW + dropI[k]];
      if (positions[k*3+1] <= groundY) {
        dropFalling[k] = 0;                            // touche le sol
        positions[k*3+1] = groundY;
        // direction lue dans flowdirGrid, pas devinée
      }
    } else {
      // interpolation vers la cellule suivante indiquée par flowdirGrid
    }
  }
}
```

Le point essentiel : au moment où une goutte touche le relief, elle lit
**directement** `flowdirGrid[j*gridW+i]` — la vraie direction calculée en
Python pour cette cellule précise — plutôt que d'improviser une direction
à partir de la pente locale recalculée côté JavaScript. Le rendu 3D et le
calcul hydrologique utilisent la même donnée, pas deux approximations
différentes.

## 5. Superposer le réseau hydrographique déjà calculé

Le réseau vectorisé (voir l'article dédié pour le détail de son calcul)
se pose directement sur le relief 3D, légèrement surélevé pour éviter
qu'il ne soit masqué par le maillage :

```js
const pts = seg.path.map(([i,j]) =>
  new THREE.Vector3(worldX[i], terrainY[j*gridW+i] + 0.15, worldZ[j])
);
const geo = new THREE.BufferGeometry().setFromPoints(pts);
const line = new THREE.Line(geo, new THREE.LineBasicMaterial({ color: 0x5FB4E5 }));
```

Comparer visuellement où les gouttes s'accumulent en ruisselant et où se
trouve le réseau déjà extrait par seuillage de l'accumulation de flux est
une bonne vérification croisée : les deux devraient converger vers les
mêmes vallées, puisqu'ils partagent la même grille de direction
d'écoulement en amont.

## 6. Le ciel

```js
function makeSkyTexture() {
  const c = document.createElement('canvas');
  c.width = 2; c.height = 256;
  const ctx = c.getContext('2d');
  const grad = ctx.createLinearGradient(0, 0, 0, 256);
  grad.addColorStop(0, '#0B1B33');
  grad.addColorStop(0.45, '#2E5C82');
  grad.addColorStop(0.75, '#7FA8C4');
  grad.addColorStop(1, '#C7D8E2');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, 2, 256);
  return new THREE.CanvasTexture(c);
}
scene.background = makeSkyTexture();
```

Un dégradé vertical dessiné sur un `<canvas>` de 2 pixels de large suffit
comme fond de scène — inutile d'un skybox à six faces ou d'une bibliothèque
dédiée pour un simple ciel de fond.

## Pour aller plus loin

- **Accumulation visuelle** : faire grossir légèrement chaque segment du
  réseau en fonction du nombre de gouttes qui y sont réellement passées
  (plutôt que de l'ordre de Strahler précalculé) donnerait une
  accumulation "vécue" en direct plutôt que précalculée.
- **Rebond et éclaboussures** : ajouter un court effet de particules
  radiales au moment de l'impact d'une goutte rendrait la chute plus
  lisible, surtout à faible exagération verticale.
- **Débit variable** : moduler l'intensité de la pluie dans le temps
  (averses puis accalmies) et observer comment le réseau "réagit" avec un
  léger délai serait une bonne façon d'introduire intuitivement la notion
  de temps de concentration d'un bassin versant.

Ce projet est surtout une démonstration qu'une même grille de calcul —
ici, une simple direction D8 — peut servir à la fois à une analyse
rigoureuse (extraire un réseau hydrographique) et à une visualisation
pédagogique animée, sans dupliquer la logique entre les deux.
