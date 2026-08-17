# Brancher Supabase sur Sèche Log — persistance des données

## Sommaire

- [1. Contexte et objectif](#1-contexte-et-objectif)
- [2. Prérequis](#2-prérequis)
- [3. Créer le projet Supabase](#3-créer-le-projet-supabase)
- [4. Modéliser les données](#4-modéliser-les-données)
- [5. Sécuriser l'accès (Row Level Security)](#5-sécuriser-laccès-row-level-security)
- [6. Activer l'authentification par lien magique](#6-activer-lauthentification-par-lien-magique)
- [7. Récupérer les clés d'API](#7-récupérer-les-clés-dapi)
- [8. Intégrer le client Supabase dans l'app](#8-intégrer-le-client-supabase-dans-lapp)
- [9. Écrire la couche de synchronisation](#9-écrire-la-couche-de-synchronisation)
- [10. Migrer les données existantes](#10-migrer-les-données-existantes)
- [11. Vérifier que ça fonctionne](#11-vérifier-que-ça-fonctionne)
- [12. Limites connues et pièges à éviter](#12-limites-connues-et-pièges-à-éviter)
- [13. Pour la suite](#13-pour-la-suite)
- [Sources](#sources)

---

## 1. Contexte et objectif

`seche-log.html` stocke aujourd'hui tout son état (`entries`, `meals`, `settings`) dans le `localStorage` du navigateur — voir `STORAGE_KEY = 'sechelog_v3'` dans le fichier. C'est propre à **un seul navigateur sur un seul appareil** : un cache vidé, un changement de téléphone ou même juste l'ouverture du site dans un autre navigateur, et l'historique de suivi disparaît sans recours.

Le déploiement sur GitHub Pages (voir [`guide-deploiement-github-pages.md`](./guide-deploiement-github-pages.md)) règle la disponibilité de l'app, pas ce problème.

**Objectif de ce guide** : ajouter Supabase (Postgres hébergé, palier gratuit) comme source de vérité durable, tout en gardant le `localStorage` actuel comme cache local pour un usage hors-ligne. Un test de bout en bout à la fin du guide ([§11](#11-vérifier-que-ça-fonctionne)) confirme que les données survivent même si le stockage local est vidé.

Ce guide correspond à la **Phase 0** de la [feuille de route du projet](#13-pour-la-suite) : fiabiliser l'usage quotidien avant toute réécriture d'architecture.

---

## 2. Prérequis

| Élément | Détail |
|---|---|
| Compte Supabase | Gratuit, création manuelle sur [supabase.com](https://supabase.com) — connexion GitHub recommandée pour rester cohérent avec le dépôt |
| Accès au dépôt | Pour éditer `seche-log.html` et y ajouter l'URL + la clé publique du projet |
| Outillage | Aucun — pas de build, pas de Node.js requis. Le client Supabase est chargé via CDN, toute la configuration serveur se fait dans le dashboard Supabase (SQL Editor inclus) |
| Un email personnel | Utilisé comme unique compte d'authentification (app mono-utilisateur) |

---

## 3. Créer le projet Supabase

1. Aller sur [supabase.com](https://supabase.com) → **Start your project** → se connecter avec GitHub.
2. **New project** :
   - **Name** : `seche-log`
   - **Database Password** : laisser Supabase la générer, la stocker immédiatement dans un gestionnaire de mots de passe (elle ne sera plus jamais affichée en clair)
   - **Region** : `Europe (Frankfurt) — eu-central-1` (latence la plus faible depuis la France)
   - **Pricing Plan** : Free
3. Valider et attendre le provisioning (~2 minutes). Le dashboard du projet s'ouvre automatiquement une fois prêt.

---

## 4. Modéliser les données

### Choix de modélisation

`seche-log.html` manipule déjà des objets JS complets par date (`state.entries['2026-08-17'] = {weight, kcal, prot, ...}`) et une liste `state.meals`. Plutôt que de normaliser chaque champ en colonne typée maintenant — ce qui demanderait de réécrire toute la couche de lecture/écriture de l'app actuelle — chaque enregistrement est stocké comme un **blob JSONB**, avec juste les clés nécessaires à l'identification et aux policies de sécurité en colonnes réelles.

C'est un choix pragmatique pour cette phase : la normalisation complète est prévue pendant la migration Angular ([§13](#13-pour-la-suite)), où la couche de données sera de toute façon réécrite derrière une interface de service.

### Mapping état local → tables

| Table | Correspond à | Clé primaire |
|---|---|---|
| `settings` | `state.settings` (un seul objet) | `user_id` |
| `entries` | `state.entries[iso]`, une ligne par date | `(user_id, entry_date)` |
| `meals` | un élément de `state.meals[]` | `(user_id, meal_id)` |

### SQL à exécuter

Dans le dashboard Supabase : **SQL Editor** → **New query** → coller et exécuter (**Run**) :

```sql
create table public.settings (
  user_id uuid primary key references auth.users (id) on delete cascade,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

create table public.entries (
  user_id uuid not null references auth.users (id) on delete cascade,
  entry_date date not null,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now(),
  primary key (user_id, entry_date)
);

create table public.meals (
  user_id uuid not null references auth.users (id) on delete cascade,
  meal_id text not null,
  name text not null,
  prot numeric not null default 0,
  gluc numeric not null default 0,
  lip numeric not null default 0,
  updated_at timestamptz not null default now(),
  primary key (user_id, meal_id)
);
```

**Vérification** : dans **Table Editor**, les trois tables `settings`, `entries`, `meals` apparaissent, vides.

---

## 5. Sécuriser l'accès (Row Level Security)

### Pourquoi c'est obligatoire ici

L'app est un site statique public sur GitHub Pages — son code source, y compris la clé publique Supabase (`anon key`), est visible par n'importe qui. Cette clé est *censée* être publique : ce qui empêche un inconnu de lire ou d'écrire les données, ce sont les policies **Row Level Security (RLS)**, pas le secret de la clé. Sans RLS activée, n'importe qui disposant de la clé publique peut lire ou modifier toutes les lignes de toutes les tables.

### SQL à exécuter

Toujours dans **SQL Editor** :

```sql
alter table public.settings enable row level security;
alter table public.entries enable row level security;
alter table public.meals enable row level security;

create policy "settings_owner" on public.settings
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "entries_owner" on public.entries
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "meals_owner" on public.meals
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

Chaque policy restreint toutes les opérations (`for all` = select/insert/update/delete) aux lignes où `user_id` correspond à l'utilisateur authentifié courant (`auth.uid()`).

**Vérification** : dans **Table Editor**, chaque table affiche un badge *RLS enabled*.

---

## 6. Activer l'authentification par lien magique

Pas de mot de passe à gérer pour un usage mono-utilisateur : un lien magique envoyé par email suffit.

1. **Authentication → Providers** : vérifier que le provider **Email** est activé (il l'est par défaut).
2. **Authentication → URL Configuration** :
   - **Site URL** : `https://<user>.github.io/<repo>/` (l'URL stable de `main`)
   - **Redirect URLs** : ajouter la même URL, puis `https://<user>.github.io/<repo>/pr-preview/**` pour que les liens magiques fonctionnent aussi depuis les previews de PR.
3. Rien d'autre à configurer — pas besoin de créer de compte manuellement, le premier lien magique envoyé crée l'utilisateur automatiquement dans `auth.users`.

---

## 7. Récupérer les clés d'API

**Project Settings → API** :

| Valeur | Usage |
|---|---|
| **Project URL** | à coller dans `seche-log.html` |
| **anon / public key** | à coller dans `seche-log.html` — publique par design, protégée par RLS |
| **service_role key** | **à ne jamais utiliser côté client** — donne un accès total en contournant RLS. Ne la copier nulle part dans ce projet |

---

## 8. Intégrer le client Supabase dans l'app

### Charger le SDK

Ajouter avant la balise `</body>` de `seche-log.html`, avant le `<script>` existant :

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
```

### Initialiser le client

En tête du `<script>` principal, avec les autres constantes de configuration :

```js
const SUPABASE_URL = 'https://xxxxxxxxxxxx.supabase.co';   // Project URL, §7
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIs...';        // anon public key, §7
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

let currentUser = null; // rempli après connexion
```

### Écran de connexion minimal

Un utilisateur non connecté doit pouvoir demander un lien magique. Ajouter un bloc simple, affiché tant qu'il n'y a pas de session :

```js
async function requestMagicLink(email) {
  const { error } = await supabase.auth.signInWithOtp({
    email,
    options: { emailRedirectTo: window.location.href }
  });
  if (error) alert('Échec de l’envoi du lien : ' + error.message);
  else alert('Lien envoyé — ouvre l’email reçu depuis ce même appareil.');
}

supabase.auth.onAuthStateChange((_event, session) => {
  currentUser = session ? session.user : null;
  if (currentUser) pullRemoteState(); // §9
});

// au chargement, restaure la session si elle existe déjà (retour sur l'app)
supabase.auth.getSession().then(({ data }) => {
  currentUser = data.session ? data.session.user : null;
  if (currentUser) pullRemoteState();
});
```

L'app reste utilisable hors connexion (le `localStorage` fonctionne toujours) : la connexion Supabase est un **rehaussement**, pas un blocage — cohérent avec l'objectif « pas de perte de données », pas « obligation d'être en ligne ».

---

## 9. Écrire la couche de synchronisation

### Principe

- **Écriture** : chaque sauvegarde locale (`scheduleSave()`, déjà présent dans le code) pousse aussi la même donnée vers Supabase, en tâche de fond.
- **Lecture** : à la connexion, l'app va chercher l'état distant et le fusionne avec le local par horodatage (`updated_at` le plus récent gagne).
- **Panne réseau** : un échec de synchronisation reste silencieux pour l'utilisateur (log console) — le `localStorage` fait toujours foi localement, la prochaine sauvegarde retentera.

### Pousser une entrée

```js
async function syncEntryToRemote(iso, entry) {
  if (!currentUser) return;
  const { error } = await supabase.from('entries').upsert({
    user_id: currentUser.id,
    entry_date: iso,
    data: entry,
    updated_at: new Date().toISOString()
  });
  if (error) console.error('Sync entrée échouée', iso, error);
}
```

Brancher cet appel dans `scheduleSave()` (fonction existante, ligne ~686 de `seche-log.html`) :

```js
function scheduleSave(){
  clearTimeout(saveTimeout);
  saveTimeout = setTimeout(()=>{
    saveState();
    syncEntryToRemote(currentDate, state.entries[currentDate]); // <-- ajout
    const s = document.getElementById('saveStatus');
    if(s) s.textContent = '✓ Sauvegardé · ' + new Date().toLocaleTimeString('fr-FR', {hour:'2-digit', minute:'2-digit'});
  }, 400);
}
```

Les paramètres (`state.settings`) et le catalogue de repas (`state.meals`) suivent le même principe : un `upsert` vers `settings`/`meals` à chaque modification (dans `saveSettingsBtn` et `addMealBtn`).

### Tirer l'état distant à la connexion

```js
async function pullRemoteState() {
  const [{ data: settingsRow }, { data: entryRows }, { data: mealRows }] = await Promise.all([
    supabase.from('settings').select('data, updated_at').eq('user_id', currentUser.id).maybeSingle(),
    supabase.from('entries').select('entry_date, data'),
    supabase.from('meals').select('meal_id, name, prot, gluc, lip')
  ]);

  if (settingsRow) {
    state.settings = Object.assign(structuredClone(DEFAULT_STATE.settings), settingsRow.data);
  }
  if (entryRows) {
    entryRows.forEach(row => { state.entries[row.entry_date] = row.data; });
  }
  if (mealRows) {
    state.meals = mealRows.map(r => ({ id: r.meal_id, name: r.name, prot: r.prot, gluc: r.gluc, lip: r.lip }));
  }

  saveState();      // réécrit le localStorage avec l'état fusionné
  renderJournal();
  renderWeek();
  renderMeals();
  renderSettings();
}
```

Cette version « le distant écrase le local à la connexion » est volontairement simple : pour un usage mono-utilisateur qui édite rarement deux appareils en même temps, un vrai merge champ-par-champ n'apporte rien et complique le code inutilement (voir aussi [§12](#12-limites-connues-et-pièges-à-éviter)).

---

## 10. Migrer les données existantes

Les données actuelles n'existent que dans le `localStorage` de l'appareil déjà utilisé. Une fois la connexion en place :

1. Se connecter dans l'app (lien magique, §8) depuis l'appareil qui a l'historique actuel.
2. Ajouter temporairement un bouton dans l'onglet **Paramètres** (à retirer après usage) :

```js
document.getElementById('pushAllBtn')?.addEventListener('click', async () => {
  if (!currentUser) return alert('Connecte-toi d’abord.');
  await Promise.all(Object.entries(state.entries).map(([iso, entry]) => syncEntryToRemote(iso, entry)));
  await supabase.from('settings').upsert({ user_id: currentUser.id, data: state.settings, updated_at: new Date().toISOString() });
  await Promise.all(state.meals.map(m => supabase.from('meals').upsert({
    user_id: currentUser.id, meal_id: m.id, name: m.name, prot: m.prot, gluc: m.gluc, lip: m.lip, updated_at: new Date().toISOString()
  })));
  alert('Migration terminée.');
});
```

3. Cliquer une fois, vérifier dans **Table Editor** que les lignes apparaissent, puis retirer le bouton et son handler.

---

## 11. Vérifier que ça fonctionne

Trois vérifications, de la plus simple à celle qui valide vraiment l'objectif du guide :

1. **Table Editor** (dashboard Supabase) : les tables `entries`, `settings`, `meals` contiennent les données attendues.
2. **Deuxième appareil** : ouvrir l'app depuis un autre navigateur/téléphone, se connecter avec le même email → les données doivent apparaître après `pullRemoteState()`.
3. **Le test qui compte** : sur l'appareil principal, vider le `localStorage` volontairement (`DevTools → Application → Local Storage → Clear`, ou `localStorage.clear()` dans la console), recharger la page, se reconnecter. Les données doivent revenir intégralement depuis Supabase. C'est ce comportement qui répond au risque de perte de données posé au [§1](#1-contexte-et-objectif).

---

## 12. Limites connues et pièges à éviter

- **Pause du projet gratuit** : un projet Supabase Free se met en pause après 7 jours sans requête. Un usage quotidien (l'objectif même de ce guide) évite ce cas ; en cas de pause, une simple visite du dashboard suffit à le réactiver.
- **Écriture concurrente** : la stratégie « le plus récent gagne » (§9) n'effectue pas de fusion fine entre deux modifications simultanées sur deux appareils différents. Non bloquant pour un usage mono-utilisateur asynchrone, mais à garder en tête si l'app devient multi-appareils actifs en parallèle.
- **`service_role key`** : ne doit jamais apparaître dans le code client ni être poussée sur le dépôt, même privé. Seule la clé `anon` va dans `seche-log.html`.
- **Redirect URLs Supabase** : toute nouvelle URL de preview de PR suit un motif prévisible (`/pr-preview/pr-<n>/`) déjà couvert par le wildcard du §6 — pas besoin de reconfigurer à chaque PR.

---

## 13. Pour la suite

Ce guide couvre la **Phase 0** de la feuille de route du projet (fiabiliser l'usage quotidien). Les étapes suivantes prévues :

- **Phase 1 — Migration Angular** : le schéma JSONB posé ici est délibérément provisoire ; la normalisation (colonnes typées, relation propre `entries ↔ meals`) est prévue à ce moment-là, derrière un service de données abstrait.
- **Phase 2 — Usine v1** : durcissement du pipeline CI (lint, tests, checks obligatoires).

---

## Sources

- [Supabase — Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase — Auth: Email OTP / Magic Link](https://supabase.com/docs/guides/auth/auth-email-passwordless)
- [Supabase — JavaScript client (`supabase-js`)](https://supabase.com/docs/reference/javascript/introduction)
- [Supabase — Pricing & limites du palier gratuit](https://supabase.com/pricing)
