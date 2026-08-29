---
title: "PostGIS + Python : requêtes spatiales depuis un script avec psycopg2 ou SQLAlchemy/GeoAlchemy2"
date: 2026-08-23T10:00:00+01:00
draft: false
categories: ["PostGIS"]
tags: ["python", "postgis", "psycopg2", "sqlalchemy", "geoalchemy2"]
summary: "Connecter un script Python à une base PostGIS pour exécuter des requêtes spatiales : comparaison pratique entre psycopg2 (SQL brut) et SQLAlchemy/GeoAlchemy2 (ORM), avec des exemples sur de vraies données."
---

Une fois des données dans PostGIS — par exemple après un [import
OpenStreetMap avec osm2pgsql](/posts/osm2pgsql-import-postgis/) — l'étape
suivante consiste souvent à les exploiter depuis un script Python : générer
un export, automatiser une analyse, ou alimenter un dashboard. Deux
approches dominent en Python pour ça : **psycopg2** (SQL brut, léger) et
**SQLAlchemy + GeoAlchemy2** (ORM, plus structuré). Ce tutoriel compare les
deux avec des exemples concrets sur la même base de données.

## Installer les dépendances

```bash
pip install psycopg2-binary sqlalchemy geoalchemy2 geopandas
```

`psycopg2-binary` évite d'avoir à compiler la bibliothèque C sous-jacente —
recommandé pour le développement (la version non-binaire est préférable en
production, mais ce détail ne change rien pour ce tutoriel).

## Base de données utilisée dans les exemples

Les exemples ci-dessous supposent une base `osm_benin` déjà peuplée (voir
l'article sur [osm2pgsql](/posts/osm2pgsql-import-postgis/)), avec une
table `planet_osm_point` contenant, entre autres, tous les restaurants
(`amenity = 'restaurant'`) du Bénin.

## Option A — psycopg2 : SQL brut, contrôle total

C'est l'approche la plus directe : on écrit du SQL, psycopg2 l'exécute.

```python
import psycopg2

conn = psycopg2.connect(
    dbname="osm_benin",
    user="postgres",
    password="motdepasse",
    host="localhost",
    port=5432
)
cur = conn.cursor()

cur.execute("""
    SELECT name, ST_AsText(way) AS geometry
    FROM planet_osm_point
    WHERE amenity = 'restaurant'
    LIMIT 10;
""")

for name, geometry in cur.fetchall():
    print(name, geometry)

cur.close()
conn.close()
```

`ST_AsText(way)` convertit la géométrie binaire stockée en base vers une
représentation texte lisible (WKT — Well-Known Text). Sans cette
conversion, psycopg2 renverrait des octets bruts difficilement
exploitables directement.

### Requêtes paramétrées — toujours

Ne jamais insérer une valeur utilisateur directement dans la chaîne SQL
(risque d'injection SQL). Toujours passer les paramètres séparément :

```python
# ❌ À ne jamais faire
cur.execute(f"SELECT * FROM planet_osm_point WHERE name = '{nom_saisi}'")

# ✅ Requête paramétrée
cur.execute("SELECT * FROM planet_osm_point WHERE name = %s", (nom_saisi,))
```

### Le raccourci le plus utile : passer directement par GeoPandas

Dans la pratique, pour la plupart des usages, il est plus simple de
laisser GeoPandas gérer la conversion géométrique automatiquement :

```python
import geopandas as gpd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:motdepasse@localhost:5432/osm_benin")

restaurants = gpd.read_postgis(
    "SELECT name, amenity, way AS geometry FROM planet_osm_point WHERE amenity = 'restaurant'",
    engine,
    geom_col="geometry"
)

restaurants.plot()
```

`gpd.read_postgis()` exécute la requête et renvoie directement un
**GeoDataFrame** exploitable — les mêmes filtres, exports et
visualisations que dans l'article sur GeoPandas s'appliquent ensuite
directement, sans manipulation manuelle de géométrie.

{{< pub slot="4444444444" >}}

## Option B — SQLAlchemy + GeoAlchemy2 : approche ORM

SQLAlchemy permet de représenter les tables comme des classes Python, et
GeoAlchemy2 ajoute le support des colonnes géométriques à SQLAlchemy.

### Définir le modèle

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import declarative_base, sessionmaker
from geoalchemy2 import Geometry

Base = declarative_base()

class Point(Base):
    __tablename__ = "planet_osm_point"
    osm_id = Column(Integer, primary_key=True)
    name = Column(String)
    amenity = Column(String)
    way = Column(Geometry("POINT", srid=3857))

engine = create_engine("postgresql://postgres:motdepasse@localhost:5432/osm_benin")
Session = sessionmaker(bind=engine)
session = session = Session()
```

### Interroger avec l'ORM

```python
restaurants = session.query(Point).filter(Point.amenity == "restaurant").limit(10).all()

for r in restaurants:
    print(r.name)
```

### Requêtes spatiales avec les fonctions PostGIS

GeoAlchemy2 expose les fonctions spatiales de PostGIS via `func` de
SQLAlchemy — la syntaxe reste proche du SQL, mais intégrée au reste du
code Python :

```python
from sqlalchemy import func

# Restaurants dans un rayon de 1 km autour d'un point donné
point_reference = func.ST_SetSRID(func.ST_MakePoint(2.3912, 6.3703), 4326)

proches = session.query(Point).filter(
    func.ST_DWithin(
        func.ST_Transform(Point.way, 3857),
        func.ST_Transform(point_reference, 3857),
        1000
    )
).all()
```

## psycopg2 ou SQLAlchemy/GeoAlchemy2 : lequel choisir ?

| Critère | psycopg2 | SQLAlchemy + GeoAlchemy2 |
|---|---|---|
| Courbe d'apprentissage | Faible — c'est du SQL direct | Plus élevée — nouveau vocabulaire (modèles, sessions) |
| Contrôle sur le SQL généré | Total | Indirect (l'ORM génère le SQL) |
| Adapté aux scripts ponctuels | Oui, idéal | Plutôt excessif |
| Adapté aux applications qui durent (API, backend) | Fonctionne, mais devient vite verbeux | Beaucoup plus maintenable à long terme |
| Risque d'erreur de syntaxe SQL | Plus élevé (SQL à la main) | Réduit (erreurs Python, pas SQL) |
| Interopérabilité avec GeoPandas | Excellente (`gpd.read_postgis`) | Bonne, mais un détour par SQL brut reste souvent plus simple |

**En résumé** : pour un script d'analyse ou d'export ponctuel, `psycopg2`
combiné à `gpd.read_postgis()` est le chemin le plus court. Pour une
application qui va grandir — un backend qui sert un géoportail dynamique,
par exemple — SQLAlchemy + GeoAlchemy2 apporte une structure qui paie sur
la durée.

## Exemple complet : de PostGIS à un fichier GeoJSON prêt pour un Web-SIG

En combinant tout ce qui précède, voici comment produire un fichier
GeoJSON directement exploitable dans un dashboard Leaflet comme ceux déjà
publiés sur ce blog :

```python
import geopandas as gpd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:motdepasse@localhost:5432/osm_benin")

restaurants = gpd.read_postgis(
    "SELECT name, amenity, way AS geometry FROM planet_osm_point WHERE amenity = 'restaurant'",
    engine,
    geom_col="geometry"
)

restaurants = restaurants.to_crs(epsg=4326)  # s'assurer du bon CRS pour le web
restaurants.to_file("restaurants_benin.geojson", driver="GeoJSON")
```

Ce fichier peut ensuite être chargé directement par n'importe lequel des
dashboards Leaflet déjà construits sur ce blog, en remplaçant simplement
le fichier de données chargé par `fetch()`.

## Sécuriser les identifiants de connexion

Ne jamais écrire le mot de passe en dur dans le script. Utiliser des
variables d'environnement :

```python
import os
from sqlalchemy import create_engine

url = f"postgresql://{os.environ['DB_USER']}:{os.environ['DB_PASS']}@{os.environ['DB_HOST']}/osm_benin"
engine = create_engine(url)
```

(avec un fichier `.env` chargé via `python-dotenv`, ou des variables
définies directement dans l'environnement d'exécution).

## Pour aller plus loin

- **Construire une API** : exposer ces requêtes via **FastAPI** ou
  **Flask** transforme ce script en backend capable de servir un
  géoportail dynamique, avec des filtres calculés côté serveur plutôt que
  chargés entièrement dans le navigateur.
- **Pool de connexions** : pour une application qui reçoit plusieurs
  requêtes simultanées, `create_engine(..., pool_size=10)` évite d'ouvrir
  une nouvelle connexion à chaque appel.
- **Migrations de schéma** : si le modèle de données évolue,
  **Alembic** (l'outil de migration de SQLAlchemy) permet de faire évoluer
  le schéma PostGIS de façon versionnée plutôt qu'à la main.

Connecter Python à PostGIS ferme la boucle entre les trois piliers de ce
blog : les données arrivent en base via osm2pgsql (ou KoboToolbox), sont
interrogées et transformées en Python, puis publiées sur une carte
Web-SIG — un pipeline complet, du terrain jusqu'au navigateur.
