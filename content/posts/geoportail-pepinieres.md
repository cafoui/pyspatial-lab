---
title: "Créer un géoportail pour gérer des pépinières — carte interactive, fiches détaillées et filtres, avec vos propres données"
date: 2026-08-25T10:00:00+01:00
draft: false
categories: ["Web-SIG"]
tags: ["web-sig", "leaflet", "geojson", "geoportail", "gestion-de-donnees"]
summary: "Construire un géoportail complet pour gérer des pépinières — carte, fiches avec photo, recherche et filtres — avec des données fictives sur le Bénin, et un guide pas à pas pour les remplacer par de vraies données terrain."
---

Un **géoportail** n'est rien d'autre qu'une carte interactive connectée à
une base de données géographique — un outil que n'importe quelle
organisation peut utiliser pour visualiser et gérer un réseau de sites :
pépinières, points de collecte, infrastructures, parcelles agricoles...

Dans cet article, on construit un géoportail complet pour **gérer un réseau
de pépinières** — avec recherche, filtres, fiches détaillées avec photo — à
partir de données fictives sur le Bénin. La dernière section explique
précisément comment remplacer ces données fictives par de vraies données
terrain, pour que ce projet serve de base réutilisable pour n'importe quel
réseau de sites à gérer.

## Voir le résultat

Le géoportail final affiche une carte du Bénin avec chaque pépinière
positionnée précisément, colorée selon son statut (active ou en pause). Une
barre latérale permet de rechercher par nom ou commune, de filtrer par
commune, par espèce cultivée et par statut, avec un compteur en temps réel
du nombre de pépinières et du stock total de plants. Cliquer sur une fiche
de la liste centre la carte sur le site correspondant et ouvre sa fiche
complète (photo, responsable, contact, espèces, capacité).

{{< pub slot="3333333333" >}}

## Pourquoi cette architecture

Un géoportail n'a pas besoin d'un serveur ni d'une base de données
classique pour être utile à une petite ou moyenne organisation. Ici, tout
repose sur **un seul fichier de données** (`pepinieres-benin.geojson`) que
la page HTML charge et affiche — la même architecture que le [tableau de
bord sismique](/posts/carte-interactive-leaflet/) publié précédemment sur
ce blog, mais appliquée à un cas de gestion plutôt qu'à un flux temps réel.

L'avantage pour un usage terrain : **modifier les données ne demande
aucune compétence en programmation**. Il suffit d'éditer un fichier texte
structuré (le GeoJSON), pas de toucher au code.

## 1. Concevoir le modèle de données

Avant d'écrire une ligne de code, il faut définir précisément quelles
informations chaque pépinière doit porter. C'est l'étape la plus
importante — un mauvais modèle de données coûte cher à corriger plus tard.

Pour ce projet, chaque pépinière est un **point géographique** (coordonnées
GPS) avec les propriétés suivantes :

| Champ | Exemple | Description |
|---|---|---|
| `nom` | "Pépinière Vert Avenir" | Nom de la pépinière |
| `commune` | "Cotonou" | Commune où elle se situe |
| `departement` | "Littoral" | Département |
| `adresse` | "Quartier Fidjrossè..." | Adresse ou repère local |
| `responsable` | "Julienne AHOUANSOU" | Nom du/de la responsable |
| `telephone` | "+229 97 00 11 22" | Contact téléphonique |
| `email` | "contact@..." | Contact email |
| `especes` | `["Teck", "Manguier"]` | Liste des espèces produites |
| `capacite_plants` | 4500 | Nombre de plants en stock |
| `statut` | "Active" ou "En pause" | État actuel de la pépinière |
| `date_creation` | "2021" | Année de création |
| `photo` | URL de l'image | Photo de la pépinière |
| `description` | texte libre | Note descriptive courte |

Ce modèle se traduit directement en GeoJSON — le format standard pour
échanger des données géographiques sur le web :

```json
{
  "type": "Feature",
  "geometry": { "type": "Point", "coordinates": [2.3912, 6.3703] },
  "properties": {
    "nom": "Pépinière Vert Avenir",
    "commune": "Cotonou",
    "departement": "Littoral",
    "adresse": "Quartier Fidjrossè, non loin du carrefour",
    "responsable": "Julienne AHOUANSOU",
    "telephone": "+229 97 00 11 22",
    "email": "contact@vertavenir.bj",
    "especes": ["Teck", "Manguier", "Moringa"],
    "capacite_plants": 4500,
    "statut": "Active",
    "date_creation": "2021",
    "photo": "https://placehold.co/500x350/5C7A66/FFFFFF?text=Pépinière+Vert+Avenir",
    "description": "Spécialisée dans les plants fruitiers et le reboisement communautaire."
  }
}
```

⚠️ **Important sur les coordonnées** : en GeoJSON, l'ordre est toujours
**`[longitude, latitude]`** — l'inverse de l'ordre habituel "latitude,
longitude" qu'on voit sur Google Maps. C'est l'erreur la plus fréquente
quand on démarre avec le GeoJSON.

Le jeu de données complet de démonstration (10 pépinières fictives réparties
sur plusieurs communes du Bénin) est disponible ici :
[pepinieres-benin.geojson](/demos/data/pepinieres-benin.geojson).

## 2. Charger les données

```js
async function loadData() {
  const res = await fetch('./data/pepinieres-benin.geojson');
  const geojson = await res.json();
  allFeatures = geojson.features;

  populateFilterOptions();
  render();
}
```

Le fichier GeoJSON est chargé une seule fois au démarrage et gardé en
mémoire dans `allFeatures`. Tous les filtres qui suivent retravaillent
cette même liste, sans jamais recharger le fichier.

## 3. Construire les filtres automatiquement à partir des données

Plutôt que d'écrire à la main la liste des communes ou des espèces dans le
HTML, on la déduit **automatiquement** du fichier de données :

```js
function populateFilterOptions() {
  const communes = [...new Set(allFeatures.map(f => f.properties.commune))].sort();
  communes.forEach(c => {
    const opt = document.createElement('option');
    opt.value = c; opt.textContent = c;
    els.commune.appendChild(opt);
  });

  const especes = [...new Set(allFeatures.flatMap(f => f.properties.especes))].sort();
  especes.forEach(e => {
    const opt = document.createElement('option');
    opt.value = e; opt.textContent = e;
    els.espece.appendChild(opt);
  });
}
```

`new Set(...)` élimine les doublons (chaque commune n'apparaît qu'une
fois dans le menu déroulant, même si plusieurs pépinières s'y trouvent).
Résultat concret : **si tu ajoutes une pépinière dans une nouvelle
commune, elle apparaît automatiquement dans le filtre** — aucune autre
modification du code n'est nécessaire.

## 4. Filtrer : recherche texte + 3 filtres combinés

```js
function getFiltered() {
  const q = els.search.value.trim().toLowerCase();
  const commune = els.commune.value;
  const espece = els.espece.value;
  const statut = els.statut.value;

  return allFeatures.filter(f => {
    const p = f.properties;
    if (q && !(p.nom.toLowerCase().includes(q) || p.commune.toLowerCase().includes(q))) return false;
    if (commune && p.commune !== commune) return false;
    if (espece && !p.especes.includes(espece)) return false;
    if (statut && p.statut !== statut) return false;
    return true;
  });
}
```

Chaque condition ne s'applique que si le filtre correspondant est
renseigné (`if (commune && ...)`) — un filtre vide laisse passer tous les
résultats. C'est ce qui permet de combiner librement recherche texte,
commune, espèce et statut, dans n'importe quelle combinaison.

## 5. Afficher : carte + liste synchronisées

```js
function render() {
  const filtered = getFiltered();
  markersLayer.clearLayers();
  els.list.innerHTML = '';

  filtered.forEach((f, i) => {
    const p = f.properties;
    const [lon, lat] = f.geometry.coordinates;
    const color = p.statut === 'Active' ? '#5C7A66' : '#486074';

    const marker = L.circleMarker([lat, lon], {
      radius: 8, fillColor: color, fillOpacity: 0.9, color: '#0F2138', weight: 1.5
    }).bindPopup(popupHTML(p));

    markersLayer.addLayer(marker);

    const card = document.createElement('div');
    card.className = 'gp-card';
    card.innerHTML = `<h3>${p.nom}</h3><div class="meta">${p.commune}</div>`;
    card.addEventListener('click', () => {
      map.setView([lat, lon], 12, { animate: true });
      marker.openPopup();
    });
    els.list.appendChild(card);
  });
}
```

**Notez l'inversion de coordonnées** : `f.geometry.coordinates` donne
`[longitude, latitude]` (norme GeoJSON), mais Leaflet attend
`[latitude, longitude]` — d'où `const [lon, lat] = ...` suivi de
`[lat, lon]` dans `L.circleMarker(...)`. C'est la même règle que dans la
section 1, appliquée ici au code.

## 6. La fiche détaillée (popup)

```js
function popupHTML(p) {
  return `
    <div class="gp-popup">
      <img src="${p.photo}" alt="${p.nom}">
      <h3>${p.nom}</h3>
      <p class="desc">${p.description}</p>
      <table>
        <tr><td>Commune</td><td>${p.commune} (${p.departement})</td></tr>
        <tr><td>Espèces</td><td>${p.especes.join(', ')}</td></tr>
        <tr><td>Capacité</td><td>${p.capacite_plants.toLocaleString('fr-FR')} plants</td></tr>
        <tr><td>Statut</td><td>${p.statut}</td></tr>
        <tr><td>Responsable</td><td>${p.responsable}</td></tr>
        <tr><td>Téléphone</td><td>${p.telephone}</td></tr>
      </table>
    </div>
  `;
}
```

Chaque propriété du GeoJSON a sa place précise dans la fiche — c'est du
gabarit HTML classique, rien de spécifique à la cartographie ici.

## Le code complet

- **Le géoportail** (HTML + CSS + JS, un seul fichier) :
  [geoportail-pepinieres.html](/demos/geoportail-pepinieres.html)
- **Les données de démonstration** :
  [pepinieres-benin.geojson](/demos/data/pepinieres-benin.geojson)

Les deux fichiers doivent rester dans la même disposition relative
(`geoportail-pepinieres.html` à côté d'un dossier `data/` contenant le
GeoJSON) pour que le chargement fonctionne.

---

## Remplacer les données fictives par les vraies données

C'est le cœur du projet : transformer cette démo en outil réellement
utilisable. Voici la marche à suivre, étape par étape.

### Étape 1 — Récupérer les coordonnées GPS de chaque pépinière

Trois méthodes possibles, de la plus simple à la plus rigoureuse :

1. **Google Maps** : faire un clic droit sur l'emplacement exact →
   cliquer sur les coordonnées affichées (ex. `6.370300, 2.391200`) pour
   les copier. **Attention à l'ordre** : Google Maps donne
   *latitude, longitude* — il faudra les inverser pour le GeoJSON (voir
   étape 3).
2. **Un smartphone sur le terrain**, avec une application GPS (ou même
   l'appareil photo, qui enregistre souvent les coordonnées EXIF).
3. **KoboToolbox** — puisque ce blog en a déjà parlé : un formulaire
   Kobo avec une question de type "Point GPS" permet de collecter les
   coordonnées de chaque pépinière directement sur le terrain, avec
   validation immédiate. C'est la méthode la plus fiable si plusieurs
   personnes doivent contribuer à la collecte.

### Étape 2 — Organiser les photos

Deux options, selon les moyens disponibles :

- **Solution la plus simple** : héberger les photos dans le dépôt GitHub
  du site lui-même, dans un dossier `static/demos/data/photos/`, puis
  utiliser un chemin relatif dans le champ `photo` (ex.
  `"photo": "./data/photos/vert-avenir.jpg"`). Fonctionne immédiatement,
  aucun compte externe nécessaire.
- **Si beaucoup de photos** (au-delà de quelques dizaines) : un service
  d'hébergement d'images externe (Cloudinary, Imgur, ou un bucket cloud)
  évite d'alourdir le dépôt Git — coller directement l'URL publique de
  chaque image dans le champ `photo`.

### Étape 3 — Remplir le fichier GeoJSON

Ouvre `pepinieres-benin.geojson` dans un éditeur de texte (VS Code par
exemple) et, pour chaque pépinière réelle, copie ce squelette :

```json
{
  "type": "Feature",
  "geometry": { "type": "Point", "coordinates": [LONGITUDE, LATITUDE] },
  "properties": {
    "nom": "",
    "commune": "",
    "departement": "",
    "adresse": "",
    "responsable": "",
    "telephone": "",
    "email": "",
    "especes": [],
    "capacite_plants": 0,
    "statut": "Active",
    "date_creation": "",
    "photo": "",
    "description": ""
  }
}
```

Remplis chaque champ, et surtout **n'oublie pas d'inverser les
coordonnées** si tu les as copiées depuis Google Maps
(`latitude, longitude` → `[longitude, latitude]` en GeoJSON). Sépare
chaque pépinière par une virgule dans le tableau `"features": [ ... ]`.

💡 **Astuce pour éviter les erreurs de syntaxe** : colle le résultat
final dans [geojson.io](https://geojson.io) avant de l'utiliser — l'outil
affiche directement une carte et signale toute erreur de structure.

### Étape 4 — Renommer et remplacer le fichier

1. Renomme `pepinieres-benin.geojson` avec un nom qui correspond à ton
   projet (ex. `pepinieres-reelles.geojson`)
2. Dans `geoportail-pepinieres.html`, mets à jour la ligne de chargement
   en conséquence :
   ```js
   const res = await fetch('./data/pepinieres-reelles.geojson');
   ```
3. Remplace le titre "DONNÉES DE DÉMONSTRATION · BÉNIN" dans le `<header>`
   par le nom réel du projet.

### Étape 5 — Déployer

Le géoportail est un simple fichier HTML statique : il se déploie
exactement comme les articles de ce blog — dépose-le dans le dépôt GitHub
(`static/demos/`), pousse sur `main`, et il est en ligne, à l'adresse
`/demos/geoportail-pepinieres.html`.

## Pour aller plus loin

- **Ajouter/modifier les pépinières sans toucher au code** : pour une
  équipe non technique, connecter le géoportail à un Google Sheet (via
  l'API Google Sheets) ou à Airtable permettrait de modifier les données
  depuis un tableur classique plutôt qu'un fichier GeoJSON — un sujet
  pour un prochain article.
- **Formulaire de collecte terrain** : un formulaire KoboToolbox dédié,
  avec les mêmes champs que ce modèle de données, permettrait à une
  équipe de terrain d'alimenter directement le géoportail.
- **Statistiques avancées** : un graphique de répartition des espèces ou
  de l'évolution du stock de plants dans le temps, avec Chart.js,
  synchronisé aux mêmes filtres.
- **Export** : ajouter un bouton "Exporter en PDF" ou "Exporter en CSV"
  pour la liste filtrée, utile pour des rapports.

Ce modèle — un fichier de données structuré, une page qui le charge et
l'affiche avec des filtres — s'adapte à n'importe quel réseau de sites à
gérer : pépinières, points de collecte de déchets, infrastructures
scolaires, parcelles agricoles. Seul le modèle de données de la section 1
change d'un projet à l'autre.
