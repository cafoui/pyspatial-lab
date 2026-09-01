---
title: "ArcGIS Online : explorer et télécharger des données avec Sentinel-2 Land Cover Explorer"
date: 2026-08-12T10:00:00+01:00
draft: false
categories: ["Web-SIG"]
tags: ["arcgis-online", "esri", "web-sig", "occupation-du-sol", "sentinel-2"]
summary: "Découvrir ArcGIS Online à travers une application concrète de Living Atlas — explorer l'occupation du sol du Bénin de 2017 à 2025, comparer deux dates, et télécharger les données en GeoTIFF."
---

Ce blog a jusqu'ici exploré des outils Web-SIG open source — GeoServer,
QGIS Server, PostGIS. **ArcGIS Online**, la plateforme cloud d'Esri, est
l'alternative commerciale la plus utilisée dans le monde professionnel de
la géomatique. Plutôt qu'une présentation abstraite, ce tutoriel prend une
application concrète et gratuite construite avec ArcGIS Online —
**Sentinel-2 Land Cover Explorer** — pour montrer ce que la plateforme
permet, et surtout **clarifier ce qui est réellement gratuit et ce qui ne
l'est pas**, une confusion fréquente chez qui découvre Esri.

## Ce qui est gratuit — et ce qui ne l'est pas

C'est le point le plus important à comprendre avant toute chose :

- **Explorer une application déjà publiée** (comme celle de ce tutoriel) :
  **gratuit, sans aucun compte**. N'importe qui peut ouvrir le lien et
  l'utiliser.
- **Créer un compte "public" ArcGIS Online** : gratuit, mais très limité
  — impossible d'y faire de l'analyse spatiale, de publier ses propres
  couches, ou d'accéder au contenu premium de Living Atlas.
- **Créer et publier ses propres cartes et applications** : nécessite un
  **abonnement payant** (avec un essai gratuit de 21 jours, 400 crédits
  inclus) — contrairement à QGIS, GeoServer ou PostGIS, déjà couverts sur
  ce blog, qui restent gratuits sans limite de durée.

Ce tutoriel se concentre sur le premier cas — un usage 100 % gratuit,
accessible immédiatement.

## Sentinel-2 Land Cover Explorer

L'application est accessible directement à
[livingatlas.arcgis.com/landcoverexplorer](https://livingatlas.arcgis.com/landcoverexplorer/) —
elle affiche une carte mondiale de l'occupation du sol, dérivée d'images
Sentinel-2, à 10 m de résolution, année par année de 2017 à 2025.

### Naviguer et lire la légende

La carte s'ouvre sur une vue globale, avec des outils de navigation
classiques (zoom, recherche de lieu). Neuf classes de couverture du sol
sont représentées par couleur :

| Couleur | Classe |
|---|---|
| Bleu | Eau |
| Vert foncé | Arbres |
| Vert clair | Végétation inondée |
| Jaune | Cultures |
| Rouge | Surface bâtie |
| Blanc/gris clair | Terrain nu |
| Blanc | Neige/Glace |
| Gris | Nuages |
| Beige | Prairies |

### Comparer deux dates

Deux modes de comparaison temporelle sont proposés :

- **Animer** : fait défiler automatiquement les années 2017 à 2025 sur la
  carte, pour visualiser une tendance en continu.
- **Swipe** : affiche un curseur séparant l'écran en deux, une année de
  chaque côté — la méthode la plus efficace pour repérer précisément un
  changement entre deux dates données.

Sur la zone du **Bénin**, on observe par exemple une progression nette des
surfaces bâties (en rouge) autour des grandes villes entre 2017 et 2025,
et un recul localisé du couvert arboré dans certaines zones.

### Télécharger les données brutes

Le bouton **"Télécharger GeoTIFF"** permet de récupérer une tuile
individuelle pour une année précise, en cliquant directement sur la
carte. Pour un usage plus large, l'application propose aussi un
**téléchargement en masse par année** (un fichier `.zip` par année,
2017 à 2025) — à noter que Esri indique environ **60 Go par année** pour
un téléchargement mondial complet, donc à réserver aux zones d'étude
précises plutôt qu'au monde entier.

Les données sont publiées sous licence **Creative Commons Attribution
(CC BY 4.0)** — réutilisables librement, y compris à des fins
commerciales, à condition de citer la source (Esri, Impact Observatory,
et Sentinel-2 / Copernicus).

## Étude de cas : le Borgou, 2017 vs 2025

En comparant le département du Borgou (Bénin) entre 2017 et 2025 avec le
mode Swipe, la progression des zones bâties (points rouges) autour des
centres urbains est immédiatement visible, de même qu'une évolution des
surfaces agricoles (jaune) en périphérie. C'est exactement le type
d'analyse qu'un MNT ou un NDVI (voir les articles précédents de ce blog)
permet de creuser plus finement une fois une zone d'intérêt identifiée
visuellement de cette façon.

## ArcGIS Online face aux outils déjà vus sur ce blog

| Critère | ArcGIS Online | QGIS + GeoServer + PostGIS |
|---|---|---|
| Coût | Payant (essai 21 jours) | Gratuit, illimité dans le temps |
| Mise en route | Immédiate, aucune installation | Installation et configuration requises |
| Catalogue de données prêtes à l'emploi | Très riche (Living Atlas) | À assembler soi-même (Geofabrik, Copernicus, etc.) |
| Personnalisation et contrôle | Limité par la plateforme | Total (open source) |
| Support | Support commercial Esri | Communauté |

**En résumé** : ArcGIS Online excelle pour explorer rapidement un
catalogue de données déjà prêtes, sans rien installer — comme le montre
cet exemple. Pour construire une infrastructure Web-SIG sur mesure et
maîtriser durablement les coûts, la pile open source déjà utilisée sur ce
blog reste la référence.

## Pour aller plus loin

- **Explorer d'autres applications Living Atlas** : Esri en publie des
  dizaines d'autres, en accès libre, sur des thématiques variées
  (démographie, climat, infrastructures).
- **Tester un compte d'essai** : les 21 jours et 400 crédits suffisent
  pour évaluer concrètement la création de cartes et l'analyse spatiale
  côté ArcGIS, avant de décider si l'investissement se justifie pour un
  projet donné.
- **Croiser les deux mondes** : rien n'empêche de télécharger un GeoTIFF
  depuis Living Atlas (gratuitement, comme montré ici) puis de le traiter
  ensuite avec Rasterio ou de le publier via GeoServer — les formats
  standards ouverts (GeoTIFF, GeoJSON) circulent librement entre les deux
  écosystèmes.

Que l'outil soit commercial ou open source, le réflexe reste le même :
comprendre précisément ce qui est gratuit, ce qui ne l'est pas, avant de
construire un projet dessus.
