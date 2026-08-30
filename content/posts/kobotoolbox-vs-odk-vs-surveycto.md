---
title: "KoboToolbox vs ODK vs SurveyCTO — comparatif pour choisir son outil de collecte"
date: 2026-08-17T10:00:00+01:00
draft: false
categories: ["KoboToolbox"]
tags: ["kobotoolbox", "odk", "surveycto", "collecte-de-donnees", "comparatif"]
summary: "KoboToolbox, ODK ou SurveyCTO : les trois outils de référence pour la collecte de données terrain, comparés sur le coût, l'hébergement, la facilité de prise en main et les fonctionnalités avancées."
---

Ce blog a déjà abordé KoboToolbox à plusieurs reprises — mais ce n'est pas
le seul outil de collecte de données terrain, et ce n'est pas toujours le
bon choix. **ODK** et **SurveyCTO** répondent au même besoin avec des
compromis différents. Voici un comparatif honnête pour choisir en
connaissance de cause, sans parti pris pour l'outil déjà couvert sur ce
blog.

## Le point commun aux trois : XLSForm

Avant même de comparer, un fait important : **les trois outils utilisent
le même standard de formulaire, XLSForm** — un fichier Excel structuré qui
décrit les questions, la logique de saut, les validations. C'est en
réalité **ODK** qui a défini ce standard. Un formulaire conçu pour l'un se
transfère avec peu de retouches vers un autre — ce qui limite le risque
de rester "enfermé" dans un outil.

## Vue d'ensemble

| | KoboToolbox | ODK | SurveyCTO |
|---|---|---|---|
| **Modèle** | Gratuit (ONG éligibles), payant sinon | Open source, auto-hébergé gratuitement | Commercial |
| **Prix (hors plan gratuit)** | À partir d'environ 21 $/mois (Starter) | Gratuit en auto-hébergé, ou ODK Cloud à partir de ~199 $/mois | À partir d'environ 349 $/mois |
| **Hébergement** | Géré par Kobo (cloud) | À toi de gérer (ou ODK Cloud géré) | Géré par SurveyCTO (cloud) |
| **Compétence technique requise** | Faible | Élevée si auto-hébergé | Faible à modérée |
| **Support** | Communauté, documentation | Communauté (forum actif) | Support dédié, y compris pour l'auto-hébergement |
| **Fonctionnalités de qualité de données avancées** | Basiques | Basiques (extensible) | Avancées (audit audio, détection d'anomalies, chiffrement multi-couche) |
| **Idéal pour** | ONG, petites/moyennes équipes, budget limité | Organisations avec capacité IT interne | Recherche à grande échelle, rigueur méthodologique exigée |

## KoboToolbox

Plateforme gratuite et open source, développée à l'origine par la Harvard
Humanitarian Initiative, aujourd'hui maintenue par une organisation à but
non lucratif dédiée. Interface web de création de formulaires, hébergement
cloud inclus, application Android (basée sur ODK Collect).

**Points forts** : gratuit pour les organisations humanitaires éligibles,
opérationnel en quelques minutes sans aucune infrastructure à gérer,
communauté active dans le secteur du développement et de l'humanitaire.

**Limites** : le plan gratuit est soumis à un plafond mensuel de
soumissions (à vérifier sur kobotoolbox.org, ce chiffre évolue), et les
fonctionnalités de contrôle qualité restent plus basiques que chez
SurveyCTO.

**Le bon choix si** : l'équipe est déjà familière avec ce blog — ce qui
est probablement le cas si tu lis cet article — ou plus généralement pour
une petite structure sans budget dédié à la collecte de données, sans
besoin d'infrastructure à gérer.

## ODK

Un écosystème open source complet : **ODK Central** (le serveur,
auto-hébergé) et **ODK Collect** (l'application Android, la même que
celle utilisée sous le capot par KoboToolbox). C'est ODK qui a défini le
standard XLSForm que les deux autres outils réutilisent.

**Points forts** : entièrement gratuit si auto-hébergé, contrôle total sur
les données et l'infrastructure (important pour des données sensibles ou
des contraintes de souveraineté des données), aucune limite artificielle
de soumissions.

**Limites** : auto-héberger implique de gérer un serveur, les mises à
jour, la sécurité — une charge réelle sans équipe technique. ODK Cloud
(version hébergée, payante) réduit cette charge mais rapproche le coût de
celui des solutions commerciales.

**Le bon choix si** : l'organisation dispose d'une capacité informatique
interne et a des raisons fortes de vouloir héberger elle-même ses
données (contraintes réglementaires, souveraineté, volumétrie très
importante).

## SurveyCTO

Plateforme commerciale construite sur les mêmes fondations XLSForm,
pensée pour la recherche à grande échelle : audits audio des entretiens
(détecter des réponses potentiellement fabriquées), monitoring en temps
réel du travail des enquêteurs, chiffrement renforcé, préchargement de
données complexes.

**Points forts** : fonctionnalités de contrôle qualité les plus abouties
des trois, hébergement géré avec support professionnel réactif,
robustesse éprouvée sur de grandes études (plusieurs milliers
d'entretiens, enquêteurs multiples, zones à faible connectivité).

**Limites** : le coût, nettement supérieur aux deux autres options —
généralement injustifié pour une petite structure qui n'utilisera jamais
les fonctionnalités avancées. Certaines équipes rapportent aussi une forte
dépendance à la personne qui a conçu les formulaires XLSForm complexes.

**Le bon choix si** : le projet est une étude de recherche à grande
échelle où la rigueur méthodologique et la détection de fraude/erreur
d'enquêteur sont critiques — un essai clinique, une enquête nationale,
une étude longitudinale avec de forts enjeux de qualité.

## En résumé

Une manière simple de trancher, largement partagée dans le secteur du
suivi-évaluation :

> **KoboToolbox pour la majorité des cas. SurveyCTO quand la rigueur
> méthodologique le justifie. ODK si l'organisation a la capacité
> informatique — et de bonnes raisons — d'auto-héberger.**

Concrètement : si KoboToolbox répond déjà au besoin, il est rarement utile
de migrer vers SurveyCTO uniquement parce que l'outil a une meilleure
réputation — l'écart de coût est réel, et la plupart des équipes
n'exploitent jamais les fonctionnalités avancées qui le justifient.

## Et pour la suite de ce blog ?

Puisque le standard XLSForm est commun aux trois, tout ce qui a été (et
sera) publié ici sur KoboToolbox — conception de formulaire, [nettoyage
de données terrain avec pandas](/posts/nettoyage-donnees-kobotoolbox-pandas/) —
s'applique presque à l'identique à des données exportées depuis ODK ou
SurveyCTO : les noms de colonnes et le format d'export varient légèrement,
mais la logique de nettoyage reste la même.
