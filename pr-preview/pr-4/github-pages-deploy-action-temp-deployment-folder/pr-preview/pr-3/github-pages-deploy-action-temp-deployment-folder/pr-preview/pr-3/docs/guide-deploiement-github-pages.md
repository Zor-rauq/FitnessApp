# Déployer une page HTML statique avec GitHub Actions + Previews de PR

## 0. Architecture retenue

GitHub Pages propose deux modes de source :
- **"GitHub Actions"** (mode natif récent, `actions/deploy-pages`) : un seul environnement, une seule URL. Pratique mais **ne permet pas nativement plusieurs déploiements simultanés** (donc pas de preview par PR).
- **"Deploy from a branch"** (branche `gh-pages`, servie telle quelle) : plus "old school", mais permet de publier **plusieurs sous-dossiers en parallèle** (`/` pour main, `/pr-preview/pr-42/` pour une PR).

Comme tu veux à la fois `main` en stable **et** des previews de PR, on part sur le mode **"Deploy from a branch"**, piloté par deux actions communautaires très utilisées :
- [`peaceiris/actions-gh-pages`](https://github.com/peaceiris/actions-gh-pages) pour `main`
- [`rossjrw/pr-preview-action`](https://github.com/rossjrw/pr-preview-action) pour les PR (elle gère aussi le nettoyage automatique quand la PR est fermée/mergée)

---

## 1. Préparer le repo

1. Ton HTML statique doit être servable tel quel : un `index.html` à la racine (ou dans un dossier, ex. `public/`, `dist/`, à toi de choisir — j'utilise `./` dans les exemples ci-dessous, adapte le `source-dir`/`publish_dir` si besoin).
2. Pas besoin de créer la branche `gh-pages` toi-même : elle sera créée automatiquement au premier run du workflow.

## 2. Créer le workflow

Crée le fichier `.github/workflows/deploy.yml` :

```yaml
name: Deploy static site

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    types: [opened, reopened, synchronize, closed]

permissions:
  contents: write        # pousser sur la branche gh-pages
  pull-requests: write   # commenter la PR avec le lien de preview

concurrency:
  group: pages-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # --- Déploiement stable : main ---
  deploy-main:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to gh-pages (root)
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./
          keep_files: true   # NE PAS écraser les dossiers pr-preview/ existants

  # --- Preview : une PR ouverte / mise à jour / fermée ---
  deploy-preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy PR preview
        uses: rossjrw/pr-preview-action@v1
        with:
          source-dir: ./
          preview-branch: gh-pages
          umbrella-dir: pr-preview
```

Points clés :
- `keep_files: true` sur le job `main` est **indispensable** : sans ça, chaque déploiement de `main` efface tout le contenu de `gh-pages`, y compris les dossiers de preview des PR en cours.
- `rossjrw/pr-preview-action` gère seul : création du dossier `pr-preview/pr-<numéro>/`, commentaire automatique sur la PR avec le lien, et suppression du dossier quand la PR est fermée/mergée (grâce à `types: [..., closed]`).
- `concurrency` évite que deux runs se marchent dessus sur la même branche/ref.

## 3. Configurer l'onglet "Pages" du repo

1. **Settings → Pages**
2. **Build and deployment → Source** : choisir **"Deploy from a branch"**
3. **Branch** : sélectionner `gh-pages` puis `/ (root)` — n'apparaît qu'**après le premier run réussi** du workflow (donc fais d'abord un push sur `main` pour déclencher la création de la branche)
4. L'URL sera de la forme `https://<user>.github.io/<repo>/`
5. Les previews de PR seront visibles sur `https://<user>.github.io/<repo>/pr-preview/pr-<numéro>/`

## 4. Vérifier que ça fonctionne

1. Push sur `main` → onglet **Actions**, vérifier que `deploy-main` passe au vert, puis ouvrir l'URL du site.
2. Créer une branche, ouvrir une PR vers `main` → le job `deploy-preview` se déclenche et **poste un commentaire automatique** dans la PR avec le lien de preview.
3. Fermer/merger la PR → le dossier de preview correspondant est supprimé automatiquement de `gh-pages`.

---

## 5. Ajustements minimaux pour industrialiser rapidement

Par ordre de priorité / rapport effort-valeur :

1. **Protéger `main`** (Settings → Branches → Branch protection rule) : exiger une PR + le check `deploy-preview` au vert avant merge. Ça transforme la preview en véritable gate de validation.
2. **Lint HTML basique** : ajouter un step `html-validate` ou `htmlhint` en amont du déploiement (job séparé `lint`, requis avant `deploy-preview`) pour attraper les erreurs de balisage.
3. **Vérification des liens morts** : action `lycheeverse/lychee-action` ou `linkinator`, à lancer sur chaque PR.
4. **Badge de statut** dans le `README.md` (`![Deploy](https://github.com/<user>/<repo>/actions/workflows/deploy.yml/badge.svg)`) pour visibilité immédiate.
5. **Lighthouse CI** (`treosh/lighthouse-ci-action`) sur l'URL de preview générée, pour suivre perf/accessibilité/SEO dès le début plutôt que de le découvrir tard.
6. **CODEOWNERS** si plusieurs personnes contribuent, pour forcer une revue ciblée.
7. Quand le site grossira (ajout d'un bundler, JS/CSS compilés) : passer `source-dir`/`publish_dir` sur le dossier de build (ex. `dist/`) et ajouter un job `build` avant les jobs de déploiement — le reste du workflow ne change pas.

Ces ajouts restent chacun de quelques lignes de YAML et peuvent être introduits progressivement sans remettre en cause l'architecture ci-dessus.
