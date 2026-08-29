---
title: "Automatiser le nettoyage des données KoboToolbox avec Python (pandas + géolocalisation)"
date: 2026-08-22T10:00:00+01:00
draft: false
categories: ["KoboToolbox"]
tags: ["python", "pandas", "kobotoolbox", "nettoyage-donnees", "geolocalisation"]
summary: "Nettoyer automatiquement un export KoboToolbox avec pandas : extraire les coordonnées GPS, supprimer les doublons, filtrer les soumissions invalides, et produire un fichier prêt pour un Web-SIG."
---

Un export KoboToolbox brut n'est presque jamais directement exploitable :
colonnes préfixées par le nom des groupes du formulaire, coordonnées GPS
stockées dans un seul champ texte, doublons dus aux resoumissions sur le
terrain, soumissions de test mélangées aux vraies données... Ce tutoriel
construit un script pandas qui automatise tout ce nettoyage, à partir d'un
export type d'un formulaire de suivi de pépinières — la suite logique de
l'article sur le [géoportail des pépinières](/posts/geoportail-pepinieres/)
publié précédemment sur ce blog.

## Exporter les données depuis KoboToolbox

Dans l'interface KoboToolbox : ouvrir le projet → onglet **Data** →
**Downloads** → choisir le format **CSV**, avec l'option **"XML values and
headers"** décochée (pour obtenir des en-têtes lisibles plutôt que des
codes internes). Le fichier téléchargé sert de point de départ à ce
tutoriel.

## À quoi ressemble un export brut

Voici un extrait représentatif de ce que produit KoboToolbox pour un
formulaire avec un groupe `groupe_infos` et un champ GPS
`gps_pepiniere` :

```
_uuid,_submission_time,groupe_infos/nom_pepiniere,groupe_infos/commune,groupe_infos/responsable,groupe_infos/telephone,groupe_infos/capacite_plants,groupe_localisation/gps_pepiniere,test
a1b2c3,2026-08-10T09:15:00,Pépinière Vert Avenir,cotonou,Julienne Ahouansou,97001122,4500,"6.370300 2.391200 15 3.2",0
a1b2c3,2026-08-10T09:17:00,Pépinière Vert Avenir,cotonou,Julienne Ahouansou,97001122,4500,"6.370300 2.391200 12 2.8",0
d4e5f6,2026-08-11T14:02:00,Pépinière du Golfe,Porto-Novo,,96123456,,"0.000000 0.000000 0 99",0
g7h8i9,2026-08-12T08:30:00,TEST FORMULAIRE,test,Test,00000000,1,"6.449000 2.356000 10 4.0",1
```

On y voit déjà les quatre problèmes classiques : une **soumission
dupliquée** (même `_uuid`, resoumise deux minutes plus tard), une
**géolocalisation invalide** (`0.000000 0.000000` — l'appareil n'a pas
réussi à capter le GPS), une **soumission de test**, et des **champs
vides**.

## 1. Charger et repérer les colonnes utiles

```python
import pandas as pd

df = pd.read_csv("export_pepinieres.csv")

# Retirer le préfixe de groupe pour des noms de colonnes plus lisibles
df.columns = [col.split("/")[-1] for col in df.columns]

print(df.columns.tolist())
```

`col.split("/")[-1]` ne garde que la partie après le dernier `/` — ainsi
`groupe_infos/nom_pepiniere` devient simplement `nom_pepiniere`. Attention
si deux groupes différents ont un champ du même nom : dans ce cas, il faut
renommer manuellement pour éviter les collisions.

## 2. Extraire latitude, longitude, altitude et précision

Le champ GPS de Kobo est une seule chaîne de texte contenant 4 valeurs
séparées par des espaces : `latitude longitude altitude précision`.

```python
gps_split = df["gps_pepiniere"].str.split(" ", expand=True)
gps_split.columns = ["latitude", "longitude", "altitude", "precision_gps"]
gps_split = gps_split.astype(float)

df = pd.concat([df, gps_split], axis=1)
```

## 3. Éliminer les soumissions de test

```python
df = df[df["test"] != 1]
```

Un champ dédié (`test`, souvent une case à cocher cachée dans le
formulaire) est la méthode la plus fiable — si ton formulaire n'en a pas,
un filtre sur le nom (`~df["nom_pepiniere"].str.contains("test", case=False, na=False)`)
peut dépanner, mais reste moins rigoureux.

## 4. Supprimer les doublons — garder la soumission la plus récente

```python
df = df.sort_values("_submission_time").drop_duplicates(subset="_uuid", keep="last")
```

Trier par date de soumission avant de dédupliquer garantit qu'on garde la
**dernière version** de chaque enregistrement (utile si l'agent de terrain
a corrigé une erreur en resoumettant).

## 5. Valider les coordonnées GPS

Deux vérifications essentielles : des coordonnées à `(0, 0)` (échec de
géolocalisation, très fréquent) et des coordonnées hors de la zone
d'étude.

```python
# Rejeter les points (0, 0) — signe d'un échec de captation GPS
df = df[(df["latitude"] != 0) | (df["longitude"] != 0)]

# Rejeter les points hors du Bénin (bounding box approximative)
BENIN_BBOX = {"lat_min": 6.0, "lat_max": 12.5, "lon_min": 0.5, "lon_max": 3.9}
df = df[
    df["latitude"].between(BENIN_BBOX["lat_min"], BENIN_BBOX["lat_max"]) &
    df["longitude"].between(BENIN_BBOX["lon_min"], BENIN_BBOX["lon_max"])
]
```

Adapter cette *bounding box* à la zone réelle du projet — c'est un filtre
de bon sens, pas une validation géométrique précise, mais il attrape la
grande majorité des erreurs de saisie GPS.

## 6. Nettoyer les champs texte

```python
# Espaces superflus
for col in ["nom_pepiniere", "commune", "responsable"]:
    df[col] = df[col].str.strip()

# Uniformiser la casse des communes (ex. "cotonou" -> "Cotonou")
df["commune"] = df["commune"].str.title()

# Signaler les champs obligatoires manquants plutôt que les supprimer silencieusement
champs_obligatoires = ["nom_pepiniere", "commune", "responsable"]
manquants = df[df[champs_obligatoires].isna().any(axis=1)]
if len(manquants) > 0:
    print(f"⚠️ {len(manquants)} soumission(s) avec des champs obligatoires manquants :")
    print(manquants[["_uuid", "nom_pepiniere", "commune"]])
```

Signaler plutôt que supprimer automatiquement les lignes incomplètes est
une bonne pratique : une valeur manquante peut être une vraie erreur à
corriger sur le terrain, pas nécessairement une soumission à jeter.

{{< pub slot="5555555555" >}}

## 7. Convertir en GeoDataFrame et exporter

Une fois les données propres, GeoPandas les transforme en couche
géospatiale prête pour un Web-SIG :

```python
import geopandas as gpd
from shapely.geometry import Point

geometry = [Point(lon, lat) for lon, lat in zip(df["longitude"], df["latitude"])]
gdf = gpd.GeoDataFrame(df, geometry=geometry, crs="EPSG:4326")

gdf.to_file("pepinieres_nettoyees.geojson", driver="GeoJSON")
```

Ce fichier a exactement la structure attendue par le [géoportail des
pépinières](/posts/geoportail-pepinieres/) déjà publié sur ce blog — il
suffit de le déposer dans le dossier `data/` du géoportail (en adaptant les
noms de colonnes aux propriétés attendues) pour que les vraies données
terrain remplacent les données fictives.

## Le script complet, sous forme de fonction réutilisable

```python
import pandas as pd
import geopandas as gpd
from shapely.geometry import Point

def nettoyer_export_kobo(chemin_csv, bbox):
    df = pd.read_csv(chemin_csv)
    df.columns = [col.split("/")[-1] for col in df.columns]

    gps_split = df["gps_pepiniere"].str.split(" ", expand=True)
    gps_split.columns = ["latitude", "longitude", "altitude", "precision_gps"]
    df = pd.concat([df, gps_split.astype(float)], axis=1)

    df = df[df["test"] != 1]
    df = df.sort_values("_submission_time").drop_duplicates(subset="_uuid", keep="last")
    df = df[(df["latitude"] != 0) | (df["longitude"] != 0)]
    df = df[
        df["latitude"].between(bbox["lat_min"], bbox["lat_max"]) &
        df["longitude"].between(bbox["lon_min"], bbox["lon_max"])
    ]

    for col in ["nom_pepiniere", "commune", "responsable"]:
        df[col] = df[col].str.strip()
    df["commune"] = df["commune"].str.title()

    geometry = [Point(lon, lat) for lon, lat in zip(df["longitude"], df["latitude"])]
    return gpd.GeoDataFrame(df, geometry=geometry, crs="EPSG:4326")


BENIN_BBOX = {"lat_min": 6.0, "lat_max": 12.5, "lon_min": 0.5, "lon_max": 3.9}
gdf = nettoyer_export_kobo("export_pepinieres.csv", BENIN_BBOX)
gdf.to_file("pepinieres_nettoyees.geojson", driver="GeoJSON")

print(f"{len(gdf)} pépinières valides après nettoyage.")
```

Cette fonction se relance à l'identique à chaque nouvel export — c'est
elle qu'on automatiserait dans un pipeline régulier (voir plus bas).

## Pour aller plus loin

- **Automatiser complètement, sans export manuel** : l'API REST de
  KoboToolbox (`https://kf.kobotoolbox.org/api/v2/`) permet de récupérer
  les soumissions directement en JSON avec un jeton d'authentification,
  sans passer par un téléchargement manuel de CSV — un script planifié
  (tâche cron, ou GitHub Actions programmé) peut ainsi régénérer le
  GeoJSON du géoportail chaque nuit.
- **Validation plus fine des coordonnées** : croiser les points avec une
  couche des limites administratives du Bénin (via une jointure spatiale
  GeoPandas, `gpd.sjoin()`) permet de vérifier qu'un point tombe bien
  dans la bonne commune déclarée — utile pour détecter des erreurs de
  saisie sur le champ "commune" lui-même.
- **Journaliser les rejets** : plutôt que de simplement filtrer les
  lignes invalides, les écrire dans un fichier `rejets.csv` séparé permet
  à l'équipe terrain de les corriger et les resoumettre.

Ce pipeline — Kobo pour la collecte, pandas pour le nettoyage, GeoPandas
pour la géométrie, et un Web-SIG pour la visualisation — forme une chaîne
complète, du terrain jusqu'à la carte, entièrement construite avec des
outils gratuits et open source.
