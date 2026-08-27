---
title: "Importer des données OpenStreetMap dans PostGIS avec osm2pgsql"
date: 2026-01-28T10:00:00+01:00
draft: false
categories: ["PostGIS"]
tags: ["postgis", "osm2pgsql", "openstreetmap", "postgresql"]
summary: "Importer un extrait OpenStreetMap complet (routes, bâtiments, points d'intérêt) dans une base PostGIS avec osm2pgsql — installation, import, et premières requêtes SQL sur de vraies données."
---

OpenStreetMap contient des millions de routes, bâtiments et points
d'intérêt, en accès libre. **osm2pgsql** est l'outil de référence pour
importer ces données directement dans une base **PostGIS**, où elles
deviennent interrogeables en SQL comme n'importe quelle autre table
spatiale. C'est l'étape qui précède la plupart des projets Web-SIG
sérieux : une fois les données dans PostGIS, elles peuvent alimenter une
carte interactive, un géoportail, ou une analyse spatiale complexe.

## Pourquoi osm2pgsql plutôt qu'un import manuel ?

On pourrait télécharger un fichier GeoJSON et l'importer avec
`ogr2ogr` ou même GeoPandas — mais OpenStreetMap n'est pas structuré comme
un jeu de données classique : un même fichier `.osm.pbf` mélange des
points, des lignes et des polygones, avec des dizaines de milliers de
combinaisons de tags possibles (`amenity=restaurant`,
`highway=residential`, `building=yes`...). osm2pgsql est spécifiquement
conçu pour trier tout ça et produire des tables PostGIS propres et
directement exploitables.

## Prérequis

- **PostgreSQL** avec l'extension **PostGIS** installée
- **osm2pgsql** installé sur ta machine

Installation d'osm2pgsql selon ton système :

```bash
# Ubuntu / Debian
sudo apt install osm2pgsql

# macOS (Homebrew)
brew install osm2pgsql

# Windows
# Via OSGeo4W (https://trac.osgeo.org/osgeo4w/), qui inclut osm2pgsql
```

Vérifie l'installation :

```bash
osm2pgsql --version
```

## 1. Créer la base de données PostGIS

```bash
createdb osm_benin
psql -d osm_benin -c "CREATE EXTENSION postgis;"
psql -d osm_benin -c "CREATE EXTENSION hstore;"
```

L'extension `hstore` est optionnelle mais recommandée : elle permet de
garder **tous** les tags OpenStreetMap d'un objet dans une seule colonne
clé-valeur, plutôt que de se limiter aux colonnes prédéfinies par
osm2pgsql (utile si tu veux exploiter des tags moins courants plus tard).

## 2. Télécharger un extrait OpenStreetMap

Il est rarement utile de télécharger la planète entière. **Geofabrik**
propose des extraits gratuits, découpés par continent, pays, voire région :
[download.geofabrik.de](https://download.geofabrik.de).

Exemple avec le Bénin :

```bash
wget https://download.geofabrik.de/africa/benin-latest.osm.pbf
```

Le fichier `.osm.pbf` est un format binaire compressé — quelques dizaines
de mégaoctets pour un pays comme le Bénin, contre plusieurs gigaoctets
pour la planète entière.

## 3. Lancer l'import

```bash
osm2pgsql \
  --create \
  --slim \
  --hstore \
  --database osm_benin \
  --username postgres \
  --host localhost \
  --cache 1000 \
  benin-latest.osm.pbf
```

Détail des options les plus importantes :

| Option | Rôle |
|---|---|
| `--create` | Crée les tables depuis zéro (à utiliser uniquement au premier import — écrase tout import précédent) |
| `--slim` | Conserve les données brutes (nœuds, chemins) en base plutôt qu'en mémoire, indispensable dès qu'on dépasse un petit extrait, et nécessaire pour pouvoir faire des mises à jour incrémentales ensuite |
| `--hstore` | Garde tous les tags OSM dans une colonne `tags` interrogeable |
| `--cache` | Mémoire (en Mo) allouée au cache — augmente ce chiffre si tu importes un pays plus grand et que ta machine dispose de RAM disponible |

Pour un import plus rapide sur un gros extrait, le mode "flat nodes"
(`--flat-nodes chemin/vers/fichier.bin`) réduit fortement la charge sur
PostgreSQL en stockant les identifiants de nœuds dans un fichier séparé
plutôt qu'en base — utile à partir de l'échelle d'un pays de taille
moyenne à grande.

## 4. Explorer les tables générées

Une fois l'import terminé, osm2pgsql a créé plusieurs tables :

```sql
\dt planet_osm_*
```

- `planet_osm_point` — points (arbres isolés, bornes, petits POI)
- `planet_osm_line` — lignes (routes, cours d'eau, limites)
- `planet_osm_polygon` — surfaces (bâtiments, parcs, plans d'eau, zones)
- `planet_osm_roads` — sous-ensemble des routes principales (généralisé,
  pratique pour l'affichage à petite échelle)

## 5. Premières requêtes SQL

Tous les restaurants du Bénin :

```sql
SELECT name, amenity, way
FROM planet_osm_point
WHERE amenity = 'restaurant';
```

Le réseau routier principal, filtré par type de voie :

```sql
SELECT name, highway, way
FROM planet_osm_line
WHERE highway IN ('primary', 'secondary', 'trunk');
```

Tous les bâtiments dans un rayon de 500 m autour d'un point donné
(nécessite une reprojection en mètres — voir l'article GeoPandas de ce
blog pour le rappel sur les CRS) :

```sql
SELECT osm_id, building, way
FROM planet_osm_polygon
WHERE building IS NOT NULL
  AND ST_DWithin(
        way,
        ST_Transform(ST_SetSRID(ST_MakePoint(2.3912, 6.3703), 4326), 3857),
        500
      );
```

Grâce à `--hstore`, on peut aussi interroger n'importe quel tag OSM, même
ceux qui n'ont pas de colonne dédiée :

```sql
SELECT name, tags -> 'cuisine' AS type_cuisine, way
FROM planet_osm_point
WHERE tags ? 'cuisine';
```

## 6. Visualiser le résultat

- **Dans QGIS** : Couche → Ajouter une couche → Couche PostGIS, connecte-toi
  à `osm_benin`, et toutes les tables `planet_osm_*` apparaissent
  directement, prêtes à afficher.
- **Sur le web** : les données PostGIS peuvent alimenter directement un
  Web-SIG via GeoServer, ou être extraites en GeoJSON pour un dashboard
  Leaflet comme ceux déjà publiés sur ce blog — en connectant
  [psycopg2](https://www.psycopg.org/) ou GeoPandas
  (`gpd.read_postgis(...)`) à la base.

## Mettre à jour les données

Un import osm2pgsql capture les données à un instant T — OpenStreetMap
change en continu. Pour mettre à jour la base sans tout réimporter :

```bash
osm2pgsql --append --slim --hstore --database osm_benin update.osc.gz
```

Le fichier `.osc.gz` (fichier de différences) peut être généré avec des
outils comme `osmium` ou téléchargé directement depuis les serveurs de
mise à jour de Geofabrik pour les extraits qu'ils proposent.

## Pour aller plus loin

- **Styles personnalisés** : osm2pgsql accepte un fichier de style Lua
  (`--style fichier.lua`) pour contrôler précisément quels tags OSM sont
  importés et comment — utile pour ne garder que les données pertinentes
  à un projet et alléger la base.
- **osm2pgsql-themepark** : un système de thèmes prêts à l'emploi
  (bâtiments, routes, POI...) qui évite d'écrire ses propres règles de
  style à partir de zéro.
- **Comparer avec un import GeoPandas classique** : relis l'article
  [GeoPandas pour les débutants](/posts/premiers-pas-geopandas/) de ce
  blog pour voir la différence d'approche entre un import ponctuel en
  Python et un import structuré en base avec osm2pgsql.

Une fois OpenStreetMap dans PostGIS, la donnée devient une vraie base de
travail : interrogeable en SQL, connectable à QGIS, à un Web-SIG, ou à un
pipeline Python — la suite logique de tout ce qu'on a construit jusqu'ici
sur ce blog.
