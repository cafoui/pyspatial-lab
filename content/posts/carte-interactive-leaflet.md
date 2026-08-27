---
title: "Créer un tableau de bord cartographique interactif avec des filtres dynamiques (Leaflet + JS)"
date: 2026-08-20T10:00:00+01:00
draft: false
categories: ["Web-SIG"]
tags: ["leaflet", "web-sig", "javascript", "geojson", "opendata"]
summary: "Construire, de zéro, un dashboard Web-SIG en temps réel avec Leaflet : filtres dynamiques, légende, statistiques — sans backend, avec des données publiques gratuites."
---

Un bon dashboard cartographique n'a pas besoin d'un backend complexe ni d'un
abonnement à une API payante. Dans ce tutoriel, on construit un **tableau de
bord sismique mondial en temps réel**, entièrement côté client (HTML +
CSS + JavaScript, sans framework), avec des filtres dynamiques et une carte
Leaflet — à partir d'une source de données **publique et gratuite** : le
flux GeoJSON de l'[USGS Earthquake Hazards Program](https://earthquake.usgs.gov/earthquake/feed/v1.0/geojson.php).

{{< pub slot="1111111111" >}}

## Voir le résultat

Le dashboard final ressemble à ceci : une carte sombre, une barre latérale
avec un sélecteur de période, un curseur de magnitude minimale, un compteur
de séismes affichés, et une légende. Chaque point représente un séisme réel
des dernières 24 heures (par défaut), coloré selon son intensité.

## Pourquoi cette donnée ?

Pour un premier dashboard, il faut une source qui coche trois cases :

1. **Aucune clé d'API** à demander ni à configurer
2. **Format GeoJSON natif** — pas de conversion, pas de géocodage à faire
3. **Données qui bougent** — un vrai dashboard prend tout son sens quand la
   donnée change dans le temps

Le flux USGS remplit exactement ces trois critères : il est mis à jour en
continu, et propose plusieurs granularités (dernière heure, jour, semaine,
mois) via des URLs prévisibles :

```
https://earthquake.usgs.gov/earthquake/feed/v1.0/summary/all_day.geojson
https://earthquake.usgs.gov/earthquake/feed/v1.0/summary/all_week.geojson
https://earthquake.usgs.gov/earthquake/feed/v1.0/summary/all_month.geojson
```

Chaque `feature` du GeoJSON contient une géométrie `[longitude, latitude,
profondeur]` et des propriétés utiles : `mag` (magnitude), `place` (lieu),
`time` (timestamp), `url` (fiche USGS détaillée).

## 1. Structure HTML

Le dashboard est composé de trois zones : un en-tête, une barre latérale de
filtres, et la carte. Pas de framework — juste une mise en page flexbox.

```html
<div class="psl-app">
  <header class="psl-bar">
    <h1>Tableau de bord sismique mondial</h1>
  </header>

  <div class="psl-body">
    <aside class="psl-panel">
      <select id="period">
        <option value="hour">Dernière heure</option>
        <option value="day" selected>Dernières 24 heures</option>
        <option value="week">Dernière semaine</option>
        <option value="month">Dernier mois</option>
      </select>

      <input type="range" id="magFilter" min="0" max="7" step="0.5" value="0">

      <div id="statCount">–</div>
    </aside>

    <div id="map"></div>
  </div>
</div>
```

L'essentiel à retenir : le `<select>` de période et l'`<input type="range">`
de magnitude sont les deux seuls contrôles nécessaires pour un dashboard
utile — inutile de multiplier les filtres dès la première version.

## 2. Initialiser la carte Leaflet

```js
const map = L.map('map', { worldCopyJump: true }).setView([15, 10], 2);

L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
  attribution: '&copy; OpenStreetMap &copy; CARTO',
  maxZoom: 18
}).addTo(map);

const markersLayer = L.layerGroup().addTo(map);
```

Deux choix méritent d'être expliqués :

- **`worldCopyJump: true`** évite qu'un marqueur près de l'antiméridien
  (±180°) semble "disparaître" quand on fait défiler la carte à
  l'horizontale.
- **`markersLayer`** est un `L.layerGroup()` vide qu'on va vider et
  remplir à chaque filtrage, plutôt que de recréer la carte entière. C'est
  la clé pour que les filtres soient réactifs sans recharger quoi que ce
  soit.

## 3. Charger les données (sans backend)

```js
const FEED_BASE = 'https://earthquake.usgs.gov/earthquake/feed/v1.0/summary/all_';
let rawFeatures = [];

async function loadData(period) {
  const res = await fetch(`${FEED_BASE}${period}.geojson`);
  const data = await res.json();
  rawFeatures = data.features || [];
  render();
}
```

Point important : on charge les données **une seule fois par période**, et
on les garde en mémoire dans `rawFeatures`. Le filtre de magnitude, lui, ne
refait **aucun appel réseau** — il retraite simplement les données déjà en
main. C'est ce qui rend le curseur instantané.

## 4. Le filtrage dynamique — le cœur du dashboard

```js
function render() {
  const threshold = parseFloat(document.getElementById('magFilter').value);
  markersLayer.clearLayers();

  let visible = 0;
  rawFeatures.forEach(f => {
    const mag = f.properties.mag;
    if (mag === null || mag < threshold) return;
    visible++;

    const [lon, lat, depth] = f.geometry.coordinates;
    const marker = L.circleMarker([lat, lon], {
      radius: Math.max(4, mag * 3.2),
      fillColor: colorForMag(mag),
      fillOpacity: 0.8,
      color: '#0F2138',
      weight: 1
    });

    marker.bindPopup(`<b>${f.properties.place}</b><br>Magnitude ${mag.toFixed(1)}`);
    markersLayer.addLayer(marker);
  });

  document.getElementById('statCount').textContent = visible;
}

function colorForMag(m) {
  if (m >= 6) return '#B33951'; // fort
  if (m >= 4) return '#C08A3E'; // modéré
  if (m >= 2) return '#486074'; // léger
  return '#5C7A66';             // micro-séisme
}
```

Le principe général d'un dashboard filtrable, quelle que soit la donnée,
tient en trois étapes qu'on retrouve ici :

1. On garde les données brutes en mémoire (`rawFeatures`)
2. On vide la couche de marqueurs (`clearLayers()`)
3. On reconstruit la couche en filtrant les données brutes selon l'état
   actuel des contrôles (`threshold`)

## 5. Connecter les filtres

```js
document.getElementById('period').addEventListener('change', (e) => {
  loadData(e.target.value);
});

document.getElementById('magFilter').addEventListener('input', () => {
  render(); // pas de rechargement réseau, juste un nouveau filtrage
});
```

Remarque : `change` sur le sélecteur de période déclenche un nouvel appel
réseau (car la donnée source change), mais `input` sur le curseur de
magnitude réagit à chaque déplacement, sans réseau. C'est cette
distinction — **quel contrôle nécessite de recharger la donnée, et lequel
peut se contenter de refiltrer ce qui est déjà chargé** — qui détermine la
fluidité perçue du dashboard.

## 6. Gérer les erreurs et l'état de chargement

Un dashboard connecté à une API réelle doit prévoir le cas où la requête
échoue (réseau coupé, API en maintenance) :

```js
async function loadData(period) {
  document.getElementById('psl-status').textContent = 'Chargement…';
  try {
    const res = await fetch(`${FEED_BASE}${period}.geojson`);
    if (!res.ok) throw new Error('Réponse invalide');
    const data = await res.json();
    rawFeatures = data.features || [];
    render();
  } catch (err) {
    document.getElementById('psl-status').textContent = 'Données indisponibles.';
  }
}
```

## Code complet

Le fichier complet (HTML + CSS + JS, un seul fichier, aucune dépendance de
build) est disponible ici : [tableau-de-bord-sismique.html](/demos/tableau-de-bord-sismique.html).
Tu peux l'ouvrir directement dans un navigateur, ou l'intégrer dans une
page du blog avec une balise `<iframe>` :

```html
<iframe src="/demos/tableau-de-bord-sismique.html" width="100%" height="600" style="border:0;"></iframe>
```

## Pour aller plus loin

- **Performance** : sur la période "mois", le flux peut contenir plusieurs
  milliers de séismes. Au-delà de quelques milliers de marqueurs, il vaut
  mieux regrouper les points proches avec le plugin
  [Leaflet.markercluster](https://github.com/Leaflet/Leaflet.markercluster)
  plutôt que de tous les afficher individuellement.
- **Filtre géographique** : ajouter un `<select>` de région qui zoome la
  carte (`map.fitBounds(...)`) et filtre les séismes par zone.
- **Graphique complémentaire** : un histogramme du nombre de séismes par
  jour (avec Chart.js par exemple) au-dessus ou en dessous de la carte,
  synchronisé avec les mêmes filtres.
- **Autres jeux de données publics** à essayer avec la même méthode :
  trafic maritime (AIS), qualité de l'air (OpenAQ), incendies actifs
  (NASA FIRMS) — toutes proposent des flux GeoJSON ou équivalents,
  gratuits et sans inscription.

La logique de ce dashboard — charger une fois, filtrer en mémoire, ne
recharger que si la source change vraiment — s'applique à n'importe quel
jeu de données géospatiales, pas seulement aux séismes.
