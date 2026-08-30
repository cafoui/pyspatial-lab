---
title: "Installer et publier ses premières couches avec GeoServer ou QGIS Server"
date: 2026-08-19T10:00:00+01:00
draft: false
categories: ["Web-SIG"]
tags: ["web-sig", "geoserver", "qgis-server", "wms", "wfs", "docker"]
summary: "Publier une couche géographique en WMS/WFS avec GeoServer ou QGIS Server, installés en quelques minutes via Docker — et la consommer depuis une carte Leaflet."
---

Tous les dashboards Web-SIG publiés jusqu'ici sur ce blog chargent un
fichier GeoJSON statique. C'est parfait pour un petit jeu de données figé,
mais dès que les données sont **volumineuses**, **mises à jour côté
serveur**, ou doivent être consommées par **plusieurs clients différents**
(QGIS, un autre site web, une application mobile), il faut un vrai serveur
de cartes. C'est le rôle de **GeoServer** et **QGIS Server** : publier des
couches géographiques selon les standards ouverts **WMS** (image de carte)
et **WFS** (données vectorielles brutes), consommables par n'importe quel
client compatible.

## GeoServer ou QGIS Server ?

| Critère | GeoServer | QGIS Server |
|---|---|---|
| Technologie | Java, moteur propre | Réutilise le moteur de rendu de QGIS Desktop |
| Interface d'administration | Interface web complète et riche | Minimale — se pilote surtout par fichiers de projet `.qgs` |
| Définir un style | Fichiers SLD (XML), ou éditeur intégré | Directement dans QGIS Desktop (styles `.qml`), puis publié tel quel |
| Idéal pour | Une équipe qui gère beaucoup de couches, avec autonomie via l'interface web | Une équipe qui travaille déjà sous QGIS Desktop et veut publier ses projets tels quels |
| Poids / complexité | Plus lourd, plus de fonctionnalités (cache intégré, sécurité fine) | Plus léger, plus simple à démarrer si QGIS est déjà l'outil du quotidien |

**En résumé** : si l'équipe connaît déjà QGIS Desktop et veut publier des
projets existants sans réapprendre un nouvel outil, QGIS Server est plus
direct. Si le besoin est de gérer un catalogue de couches avec une
interface d'administration complète (utilisateurs, styles avancés, cache),
GeoServer est plus complet.

## Installer avec Docker (recommandé)

Compiler GeoServer ou QGIS Server à la main implique de gérer des
dépendances Java ou QGIS complexes selon l'OS. Docker évite tout ça — une
seule commande, identique sur Windows, Mac et Linux.

### GeoServer

```bash
docker run -d \
  --name geoserver \
  -p 8080:8080 \
  -e GEOSERVER_ADMIN_PASSWORD=changezmoi \
  kartoza/geoserver:latest
```

Une fois démarré (le premier lancement prend une minute ou deux),
l'interface d'administration est accessible sur
**http://localhost:8080/geoserver**, avec l'identifiant `admin` et le mot
de passe défini ci-dessus.

### QGIS Server

```bash
docker run -d \
  --name qgis-server \
  -p 8081:80 \
  -v $(pwd)/projets:/io/data \
  kartoza/qgis-server:latest
```

Le dossier local `projets/` (créé avant de lancer la commande) contiendra
les fichiers `.qgs` à publier — QGIS Server sert directement les projets
qu'on y dépose.

{{< pub slot="7777777777" >}}

## Publier une couche avec GeoServer

On réutilise le GeoJSON des pépinières du [géoportail déjà publié sur ce
blog](/posts/geoportail-pepinieres/).

1. Dans l'interface GeoServer, **Data → Workspaces → Add new workspace**
   — nommer par exemple `pyspatial`, avec un URI (`http://pyspatial.local`
   suffit, il n'a pas besoin d'être un vrai domaine).
2. **Data → Stores → Add new Store → GeoJSON** (ou déposer le fichier
   dans un dossier monté dans le conteneur), sélectionner le workspace
   `pyspatial`, et pointer vers `pepinieres-benin.geojson`.
3. **Publish** apparaît automatiquement à côté de la couche détectée —
   cliquer dessus, vérifier que l'étendue géographique (bounding box) est
   calculée correctement (bouton "Compute from data"), puis **Save**.

La couche est maintenant publiée. Son URL **WMS GetCapabilities** (la
liste de tout ce que le serveur expose) est accessible à :

```
http://localhost:8080/geoserver/pyspatial/wms?service=WMS&version=1.3.0&request=GetCapabilities
```

### Si les données viennent déjà de PostGIS

Pour une base déjà en place (voir l'article sur
[osm2pgsql](/posts/osm2pgsql-import-postgis/)), le principe est identique,
mais à l'étape 2 on choisit **Add new Store → PostGIS**, avec les
identifiants de connexion à la base — GeoServer interroge alors
directement PostGIS à chaque requête, sans dupliquer les données.

## Publier un projet avec QGIS Server

1. Dans **QGIS Desktop**, ouvrir ou créer un projet, y ajouter la couche
   des pépinières (`Couche → Ajouter une couche → Ajouter une couche
   vecteur`), et appliquer un style (couleur, symboles) comme pour
   n'importe quelle carte QGIS classique.
2. **Projet → Enregistrer sous...**, et sauvegarder dans le dossier
   `projets/` monté dans le conteneur Docker (ex.
   `pepinieres.qgs`).
3. Le projet est immédiatement servi, accessible via :

```
http://localhost:8081/?MAP=/io/data/pepinieres.qgs&SERVICE=WMS&REQUEST=GetCapabilities
```

Avec QGIS Server, **le style défini dans QGIS Desktop est directement
celui utilisé pour le rendu WMS** — aucune conversion de style
supplémentaire à faire, contrairement à GeoServer où il faut écrire ou
générer un fichier SLD séparé.

## Tester le résultat

Le plus simple pour vérifier que tout fonctionne : ouvrir **QGIS Desktop**
sur sa propre machine, **Couche → Ajouter une couche → Ajouter une couche
WMS/WMTS**, créer une nouvelle connexion avec l'URL GetCapabilities
ci-dessus, et charger la couche — elle doit s'afficher exactement comme
dans le projet source.

## Consommer la couche WMS depuis une carte Leaflet

Les dashboards Leaflet publiés sur ce blog jusqu'ici chargeaient un
GeoJSON complet en une fois. Avec un flux WMS, c'est le serveur qui génère
l'image de la carte à la demande, à la bonne échelle :

```js
const wmsLayer = L.tileLayer.wms('http://localhost:8080/geoserver/pyspatial/wms', {
  layers: 'pyspatial:pepinieres-benin',
  format: 'image/png',
  transparent: true,
  version: '1.3.0'
}).addTo(map);
```

Pour récupérer les données vectorielles brutes plutôt qu'une image (afin
de garder des popups cliquables comme dans les dashboards précédents), on
utilise le **WFS** à la place, qui renvoie du GeoJSON directement
interrogeable :

```js
fetch('http://localhost:8080/geoserver/pyspatial/ows?service=WFS&version=2.0.0&request=GetFeature&typeName=pyspatial:pepinieres-benin&outputFormat=application/json')
  .then(res => res.json())
  .then(geojson => {
    L.geoJSON(geojson).addTo(map);
  });
```

C'est la même logique de `fetch()` utilisée dans les dashboards
précédents — seule la source change : un fichier statique devient une
requête vers un vrai serveur de cartes.

## Pour aller plus loin

- **Sécuriser l'accès** : GeoServer permet de restreindre certaines
  couches par utilisateur ou par rôle (**Security → Users, Roles**) —
  indispensable avant toute mise en production avec des données
  sensibles.
- **Mettre en cache les tuiles** : GeoServer embarque **GeoWebCache**, qui
  pré-génère et met en cache les tuiles WMS aux échelles courantes —
  accélère fortement le chargement pour un site à fort trafic.
- **HTTPS et nom de domaine réel** : en production, placer GeoServer ou
  QGIS Server derrière un reverse proxy (nginx, Caddy) avec un certificat
  HTTPS, plutôt que de l'exposer directement sur le port Docker.
- **Automatiser le déploiement** : une fois la configuration stabilisée,
  un fichier `docker-compose.yml` regroupant GeoServer (ou QGIS Server) et
  PostGIS permet de retrouver l'environnement complet en une seule
  commande, sur n'importe quelle machine.

Publier via WMS/WFS plutôt qu'un simple fichier statique est l'étape qui
transforme un dashboard ponctuel en une vraie infrastructure Web-SIG,
réutilisable par n'importe quel client — QGIS, un autre site, une
application mobile — sans dupliquer les données à chaque fois.
