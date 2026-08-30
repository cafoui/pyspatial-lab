---
title: "PostGIS pour ceux qui viennent de PostgreSQL — les 10 fonctions spatiales à connaître"
date: 2026-08-18T10:00:00+01:00
draft: false
categories: ["PostGIS"]
tags: ["postgis", "postgresql", "sql", "gis"]
summary: "Dix fonctions PostGIS qui couvrent 90 % des besoins spatiaux du quotidien, expliquées pour quelqu'un qui connaît déjà SQL et PostgreSQL mais découvre la partie géospatiale."
---

PostGIS n'est pas un autre langage à apprendre — c'est PostgreSQL, avec un
type de donnée `geometry` en plus et une bibliothèque de fonctions `ST_*`
(*Spatial Type*) pour le manipuler. Si SQL et PostgreSQL sont déjà
familiers, l'essentiel du chemin restant tient dans une dizaine de
fonctions. Les voici, avec des exemples sur une vraie base — celle
importée dans l'article sur [osm2pgsql](/posts/osm2pgsql-import-postgis/).

## D'abord, deux différences avec PostgreSQL "classique"

**Le type de colonne.** Une colonne géométrique n'est ni un `TEXT` ni un
`JSON` — c'est un type dédié, `geometry(Point, 4326)` par exemple, où
`4326` est le **SRID** (l'identifiant du système de coordonnées, ici
WGS84, le standard GPS en degrés). Deux géométries avec des SRID
différents ne se comparent pas directement — il faut les ramener au même
système avec `ST_Transform` (fonction n°9 plus bas).

**L'index spatial.** Un index classique (B-tree) ne sait pas accélérer une
requête "quels points sont proches de celui-ci ?". PostGIS utilise un
index **GiST** :

```sql
CREATE INDEX idx_pepinieres_geom ON pepinieres USING GIST (geometry);
```

Sans cet index, toute requête spatiale sur une table volumineuse
parcourt l'intégralité des lignes — la différence de performance est
souvent de plusieurs ordres de grandeur.

## Les 10 fonctions

### 1. `ST_MakePoint` + `ST_SetSRID` — créer une géométrie

```sql
SELECT ST_SetSRID(ST_MakePoint(2.3912, 6.3703), 4326);
```

Construit un point à partir de coordonnées `(longitude, latitude)` — dans
cet ordre, comme en GeoJSON. `ST_SetSRID` précise ensuite le système de
coordonnées ; sans cette précision, PostGIS ne sait pas si les nombres
représentent des degrés ou des mètres.

### 2. `ST_Distance` — distance entre deux géométries

```sql
SELECT ST_Distance(
  ST_Transform(a.way, 3857),
  ST_Transform(b.way, 3857)
) AS distance_metres
FROM planet_osm_point a, planet_osm_point b
WHERE a.name = 'Pépinière Vert Avenir' AND b.amenity = 'restaurant'
ORDER BY distance_metres LIMIT 5;
```

⚠️ En SRID 4326 (degrés), `ST_Distance` renvoie un résultat en **degrés**,
pas en mètres — un chiffre inutilisable directement. Toujours reprojeter
en un système métrique (ici 3857, Web Mercator) avant de mesurer une
distance réelle.

### 3. `ST_DWithin` — tester une proximité (et utiliser l'index)

```sql
SELECT name FROM planet_osm_point
WHERE ST_DWithin(
  ST_Transform(way, 3857),
  ST_Transform(ST_SetSRID(ST_MakePoint(2.3912, 6.3703), 4326), 3857),
  1000
);
```

Pour filtrer "tout ce qui est à moins de X mètres", `ST_DWithin` est
**presque toujours préférable à `ST_Distance(...) < X`** : PostGIS peut
utiliser l'index spatial pour éliminer rapidement les géométries hors
de portée, alors que `ST_Distance` doit calculer la distance exacte pour
chaque ligne avant de filtrer.

### 4. `ST_Intersects` — deux géométries se touchent-elles ?

```sql
SELECT b.osm_id, b.name
FROM planet_osm_polygon b
WHERE ST_Intersects(b.way, ST_Buffer(ST_Transform(...), 500));
```

La fonction la plus utilisée pour croiser deux couches — quelles
parcelles touchent une route, quels bâtiments touchent une zone
inondable, etc. Fonctionne aussi bien pour repérer un simple contact
qu'un vrai chevauchement.

### 5. `ST_Contains` — une géométrie en contient-elle une autre ?

```sql
SELECT p.name AS commune, COUNT(r.osm_id) AS nb_restaurants
FROM communes p
JOIN planet_osm_point r ON ST_Contains(p.geometry, r.way) AND r.amenity = 'restaurant'
GROUP BY p.name;
```

Différence avec `ST_Intersects` : `ST_Contains` exige que la géométrie
soit **entièrement à l'intérieur**, pas seulement en contact. C'est la
fonction naturelle pour un comptage "combien de points dans chaque
zone".

{{< pub slot="8888888888" >}}

### 6. `ST_Buffer` — créer une zone tampon

```sql
SELECT ST_Buffer(ST_Transform(way, 3857), 200) AS zone_200m
FROM planet_osm_point
WHERE name = 'Pépinière Vert Avenir';
```

Génère un polygone représentant tout ce qui se trouve à une distance
donnée (ici 200 m) d'une géométrie — la base de toute analyse "zone
d'influence" ou "périmètre de sécurité". Toujours reprojeter en mètres
avant d'appeler `ST_Buffer` avec une distance réelle, pour la même raison
que `ST_Distance`.

### 7. `ST_Area` — calculer une surface

```sql
SELECT osm_id, building, ST_Area(ST_Transform(way, 3857)) AS surface_m2
FROM planet_osm_polygon
WHERE building IS NOT NULL
ORDER BY surface_m2 DESC LIMIT 10;
```

### 8. `ST_Length` — calculer une longueur

```sql
SELECT name, highway, ST_Length(ST_Transform(way, 3857)) AS longueur_m
FROM planet_osm_line
WHERE highway = 'primary';
```

Même principe que `ST_Area` : sans reprojection préalable en système
métrique, le résultat est en degrés et n'a pas de sens physique direct.

### 9. `ST_Transform` — changer de système de coordonnées

```sql
SELECT ST_Transform(way, 3857) FROM planet_osm_point LIMIT 1;
```

La fonction qui revient dans presque tous les exemples ci-dessus.
Règle simple à retenir : **EPSG:4326** (degrés, WGS84) pour stocker et
échanger des données (GeoJSON, GPS), **un système métrique local**
(3857 pour du web, ou un système national comme l'UTM approprié) pour
tout calcul de distance, surface ou buffer.

### 10. `ST_AsGeoJSON` — exporter vers le web

```sql
SELECT json_build_object(
  'type', 'FeatureCollection',
  'features', json_agg(
    json_build_object(
      'type', 'Feature',
      'geometry', ST_AsGeoJSON(way)::json,
      'properties', json_build_object('nom', name, 'amenity', amenity)
    )
  )
)
FROM planet_osm_point
WHERE amenity = 'restaurant';
```

Construit un GeoJSON complet directement en SQL, prêt à être renvoyé par
une API ou chargé dans un dashboard Leaflet — la même logique que
`ST_AsText` (vu dans l'article sur [psycopg2 et
SQLAlchemy](/posts/postgis-python-psycopg2-sqlalchemy/)), mais qui
produit ici directement un format exploitable côté web, sans conversion
supplémentaire en Python.

## Un piège fréquent : `geometry` vs `geography`

PostGIS propose deux types pour stocker des coordonnées géographiques :
`geometry` (plan cartésien, rapide, nécessite une reprojection manuelle
pour des mesures exactes) et `geography` (sphère terrestre, mesures
directement en mètres, mais plus lent). Pour la plupart des projets —
tout ce qui précède dans cet article — `geometry` avec des reprojections
explicites via `ST_Transform` reste le choix le plus courant et le plus
performant.

## Récapitulatif

| Fonction | Usage |
|---|---|
| `ST_MakePoint` / `ST_SetSRID` | Créer une géométrie |
| `ST_Distance` | Mesurer une distance exacte |
| `ST_DWithin` | Filtrer par proximité (utilise l'index) |
| `ST_Intersects` | Tester un contact/chevauchement |
| `ST_Contains` | Tester une inclusion complète |
| `ST_Buffer` | Créer une zone tampon |
| `ST_Area` | Calculer une surface |
| `ST_Length` | Calculer une longueur |
| `ST_Transform` | Changer de système de coordonnées |
| `ST_AsGeoJSON` | Exporter vers le web |

## Pour aller plus loin

- **`ST_Union`** et **`ST_Difference`** : fusionner ou soustraire des
  géométries entre elles — utiles dès qu'on manipule plusieurs couches
  qui se recoupent.
- **`ST_Simplify`** : réduire le nombre de points d'une géométrie
  complexe, pour accélérer l'affichage web de couches très détaillées.
- La documentation officielle PostGIS (postgis.net/docs) reste la
  référence complète — mais ces dix fonctions couvrent la grande
  majorité des besoins d'un projet Web-SIG classique.

Une fois ces dix fonctions maîtrisées, l'essentiel des requêtes spatiales
du quotidien — filtrer par proximité, croiser deux couches, mesurer,
exporter — se construit en combinant simplement ce vocabulaire de base.
