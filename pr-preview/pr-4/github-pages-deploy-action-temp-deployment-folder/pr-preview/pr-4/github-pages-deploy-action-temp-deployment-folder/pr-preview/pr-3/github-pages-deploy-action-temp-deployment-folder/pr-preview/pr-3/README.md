# FitnessApp

Petit site statique pour suivre la nutrition, le poids, les macros et les performances.

## Objectif

Ce projet est une application web statique, hébergée via GitHub Pages.

Le site est conçu pour permettre :

- un accès simple à la version stable publique
- des previews de branches / pull requests
- un suivi personnel de la progression via stockage local dans le navigateur

## Déploiement GitHub Pages

Le site est déployé avec une configuration de type branche GitHub Pages, pas via le mode GitHub Actions.

### Configuration requise dans GitHub

Dans le dépôt GitHub :

1. Ouvrir Settings
2. Ouvrir Pages
3. Choisir :
   - Source: Deploy from a branch
   - Branch: gh-pages
   - Folder: / (root)

Cette configuration est obligatoire pour que le site public soit servi depuis la branche `gh-pages`.

## Règles de publication

### Version stable

- push sur `main` → publie à la racine du site (job `deploy-main`)
- URL attendue :
  - `https://<user>.github.io/<repo>/`

### Previews de pull request

- pull request vers `main` (ouverte / mise à jour / fermée) → publie dans :
  - `/pr-preview/pr-<numero>/`
- gérée automatiquement par `rossjrw/pr-preview-action` (job `deploy-preview`) : création du dossier, commentaire sur la PR avec le lien, suppression automatique à la fermeture/merge

Il n'y a pas de preview pour les simples push sur une branche hors PR — seules les pull requests génèrent une preview, ce qui évite les doublons et les conflits de publication.

## Workflow utilisé

Le workflow est dans :

- `.github/workflows/github-pages.yml`

Deux jobs distincts, chacun déclenché par un seul type d'événement (push sur `main` ou événement `pull_request`), publient sur la branche `gh-pages` avec `keep_files: true` pour ne pas s'écraser mutuellement.

## Points importants

- Un seul mode de publication doit être actif : le mode branche `gh-pages`
- Les workflows de type `actions/deploy-pages` doivent être évités dans ce setup si l’on veut publier via `gh-pages`
- Les previews ne doivent pas écraser la racine du site ; elles doivent être stockées dans le sous-dossier `pr-preview/`

## Structure du projet

- `index.html` : page d’accueil / version publique du site
- `seche-log.html` : application de suivi nutrition / performance
- `.github/workflows/` : workflows de déploiement
- `.nojekyll` : permet d’éviter le traitement Jekyll sur GitHub Pages

## Lancer localement

Ouvrir le fichier HTML directement dans le navigateur ou utiliser un petit serveur local si nécessaire.

Exemple rapide :

```bash
python -m http.server 8000
```

Puis ouvrir :

```text
http://localhost:8000
```

## Remarque

Ce projet est un outil personnel de suivi et ne remplace ni un avis médical ni un accompagnement professionnel.
