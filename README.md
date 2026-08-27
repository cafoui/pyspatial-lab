# PySpatial Lab — Blog Géomatique

Blog géomatique (Python, PostGIS, Web-SIG, KoboToolbox), construit avec
**Hugo** + thème **PaperMod**, déployé automatiquement sur **GitHub Pages**.

---

## PHASE 1 : Configuration et Déploiement (Semaine 1)

### ☐ 1. Créer un compte GitHub dédié au projet
Si ce n'est pas déjà fait : https://github.com/join

### ☐ 2. Créer le dépôt et ajouter ces fichiers
1. Sur GitHub, créez un nouveau dépôt public nommé par exemple `pyspatial-lab`.
2. Sur votre machine, avec Hugo installé (https://gohugo.io/installation/) :
   ```bash
   git clone https://github.com/VOTRE-PSEUDO-GITHUB/pyspatial-lab.git
   cd pyspatial-lab
   ```
3. Copiez tous les fichiers/dossiers de cet export dans le dépôt cloné.
4. Ajoutez le thème PaperMod en sous-module :
   ```bash
   git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
   git submodule update --init --recursive
   ```
5. Dans `hugo.yaml`, remplacez :
   - `VOTRE-PSEUDO-GITHUB` par votre nom d'utilisateur GitHub (baseURL + liens sociaux)
   - `Votre Nom` / `votre.email@example.com` par vos informations
   - `VOTRE-PROFIL` (LinkedIn) si utilisé

### ☐ 3. Activer GitHub Pages via GitHub Actions
1. Poussez le tout sur `main` :
   ```bash
   git add .
   git commit -m "Initialisation du blog PySpatial Lab (Hugo + PaperMod)"
   git push origin main
   ```
2. Sur GitHub : **Settings → Pages → Build and deployment → Source** →
   choisissez **GitHub Actions**.
3. Le workflow `.github/workflows/hugo.yml` se déclenche automatiquement à
   chaque push sur `main` et publie le site.
4. Votre blog sera visible à :
   `https://VOTRE-PSEUDO-GITHUB.github.io/pyspatial-lab/`

### ☐ 3bis. Alternative : déployer via Render (connecté à GitHub)

Au lieu de GitHub Pages, tu peux héberger sur **Render** avec déploiement
automatique à chaque `push` :

1. Pousse d'abord le dépôt sur GitHub (étapes 1-2 ci-dessus).
2. Va sur https://render.com et crée un compte (tu peux te connecter avec GitHub directement).
3. **New +** → **Static Site** → choisis ton dépôt `pyspatial-lab`.
4. Render détecte automatiquement `render.yaml` à la racine du projet
   (Build Command, Publish Directory et version de Hugo déjà configurés).
   Si on te demande de confirmer manuellement :
   - **Build Command** : `hugo --gc --minify --baseURL "$RENDER_EXTERNAL_URL"`
   - **Publish Directory** : `public`
   - Variable d'environnement `HUGO_VERSION` = `0.134.3`
5. Clique **Create Static Site**. Render build et publie le site — l'URL
   est du type `https://pyspatial-lab.onrender.com` (un domaine personnalisé
   peut être ajouté gratuitement ensuite dans **Settings → Custom Domains**).
6. À chaque `git push` sur `main`, Render redéploie automatiquement — aucune
   action manuelle supplémentaire.

> ⚠️ Si tu utilises Render, le workflow `.github/workflows/hugo.yml` (GitHub
> Actions → GitHub Pages) devient inutile : tu peux soit le supprimer, soit
> le laisser désactivé — les deux plateformes ne se gênent pas si tu choisis
> de n'en garder qu'une active dans **Settings → Pages** de ton dépôt.

### ☐ 4. Page "À Propos"
Déjà rédigée dans `content/about.md` — à personnaliser avec vos vraies
compétences, expériences et liens.

### ☐ 5. Catégories principales
Déjà configurées dans le menu (`hugo.yaml`) et illustrées par un article de
démarrage (brouillon) dans `content/posts/` pour chacune :
- **Python** (`premiers-pas-geopandas.md`)
- **PostGIS** (`introduction-postgis.md`)
- **Web-SIG** (`carte-interactive-leaflet.md`)
- **KoboToolbox** (`collecte-terrain-kobotoolbox.md`)

Passez `draft: true` à `draft: false` dans le front-matter quand un article
est prêt à publier.

---

## Tester en local

```bash
hugo server -D
```
Puis ouvrez http://localhost:1313/

## Identité visuelle

Le thème PaperMod est conservé (recherche, mode sombre, SEO, RSS...) mais
habillé d'une identité sur-mesure inspirée des instruments d'arpentage et
des cartes topographiques :

- **Palette** : bleu carte marine (`#0F2138`), laiton (`#C08A3E`), vert
  courbe de niveau (`#5C7A66`) — définie dans `assets/css/extended/custom.css`
- **Typographies** : Fraunces (titres) + Public Sans (texte) + JetBrains Mono
  (coordonnées, tags, code)
- **Hero** : courbes de niveau animées en arrière-plan + **panneau de
  couches**, une reprise directe du panneau de calques d'un logiciel SIG
  (QGIS/ArcGIS), qui sert de navigation vers les 4 catégories
  (`layouts/partials/home_info.html`)

Pour ajuster les couleurs des couches ou en ajouter une nouvelle catégorie,
éditez `layouts/partials/home_info.html` (bloc `.psl-layer`).

## Configurer Google AdSense

1. Ton site doit d'abord être **en ligne avec du vrai contenu** (plusieurs
   articles publiés, pages À Propos/Contact/Politique de confidentialité
   remplies) — AdSense refuse les sites vides ou en construction.
2. Inscris-toi sur https://adsense.google.com avec l'URL de ton blog.
3. Une fois approuvé, Google te donne un **Publisher ID** au format
   `ca-pub-XXXXXXXXXXXXXXXX`.
4. Renseigne-le dans `hugo.yaml` :
   ```yaml
   params:
     adsenseClientID: "ca-pub-XXXXXXXXXXXXXXXX"
   ```
   Le script AdSense (`layouts/partials/extend_head.html`) ne se charge
   que si cette valeur est remplie — donc rien ne s'affiche tant que tu
   n'es pas approuvé.
5. Mets à jour `static/ads.txt` avec ton vrai Publisher ID (remplace
   `pub-0000000000000000`) :
   ```
   google.com, pub-XXXXXXXXXXXXXXXX, DIRECT, f08c47fec0942fa0
   ```
   Ce fichier est **obligatoire** pour qu'AdSense diffuse des annonces.
6. Pour placer un bloc publicitaire précis dans un article, utilise le
   shortcode dédié :
   ```
   {{< pub slot="1234567890" >}}
   ```
   (le numéro de "slot" est fourni par AdSense quand tu crées une unité
   publicitaire manuelle — sinon les "Auto ads" de Google placent les
   annonces automatiquement une fois le script chargé, sans shortcode).

## Structure du projet

```
pyspatial-lab/
├── hugo.yaml                 # configuration du site (menu, thème, params)
├── content/
│   ├── about.md              # page "À Propos"
│   └── posts/                # articles du blog
├── archetypes/default.md     # modèle pour "hugo new posts/mon-article.md"
├── .github/workflows/hugo.yml # déploiement automatique GitHub Pages
└── static/images/            # favicon, images, etc.
```
