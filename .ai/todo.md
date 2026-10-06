# Rôle

Tu es un architecte frontend senior spécialisé en Next.js (App Router), TypeScript et en intégration d'API REST.

# Contexte

Je développe un projet Next.js qui consomme une API REST développée avec Laravel. La documentation complète de cette API est dans le fichier `openapi.yaml` à la racine du projet.

- URL de base de l'API : `http://localhost:8000/api/v2`
- Authentification : **Laravel Sanctum en mode token (Bearer)**. Je n'utilise **pas** les cookies ni le mode SPA : pas de `withCredentials`, pas d'appel à `/sanctum/csrf-cookie`, pas de gestion CSRF.

# Objectif

Analyser `openapi.yaml` et me proposer, puis générer, une structure de dossiers adaptée à cette API et conforme aux bonnes pratiques Next.js.

# Stack imposée

- Next.js (App Router, dossier `src/`) avec TypeScript
- axios pour toutes les requêtes HTTP
- Tailwind CSS + DaisyUI pour le style
- Variables d'environnement dans un fichier `.env`
- N'ajoute aucune autre dépendance sans me la proposer et la justifier d'abord

# Étapes à suivre

## Étape 1 : Analyse de `openapi.yaml`

Lis le fichier et présente-moi un résumé court :
- La cohérence entre la base URL indiquée dans `servers` et `http://localhost:8000/api/v2` (signale-moi tout écart)
- Les endpoints d'authentification (login, register, logout, refresh, profil, etc.) et la forme exacte du token retourné
- La liste des ressources (regroupées par `tags` ou par préfixe de chemin) et leurs endpoints
- Le format des réponses (pagination, enveloppe `data`, format des erreurs 422, etc.)
- Les endpoints protégés et ceux qui sont publics

## Étape 2 : Proposition de l'arborescence

Propose une arborescence complète sous forme d'arbre, avec une phrase d'explication par dossier important. Elle doit respecter ces principes :

1. **Routing** : utiliser l'App Router (`src/app`), avec des route groups (ex. `(auth)`, `(dashboard)`) si c'est pertinent, ainsi que les fichiers spéciaux `layout.tsx`, `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`.
2. **Couche API** (`src/lib/api` ou `src/services`) :
   - une instance axios unique et centralisée (`baseURL` depuis les variables d'environnement, header `Accept: application/json`, timeout)
   - un intercepteur de requête qui injecte l'en-tête `Authorization: Bearer <token>` quand un token existe
   - un intercepteur de réponse qui gère les 401 (suppression du token et redirection vers la page de connexion) et normalise les erreurs Laravel (422 avec `errors`, 403, 404, 500)
   - un fichier de service par ressource de l'API (ex. `auth.service.ts`, `users.service.ts`), calqué sur les tags de l'OpenAPI
3. **Authentification par token** (sans cookies) :
   - un module dédié au stockage du token (`src/lib/auth/token.ts`) avec `getToken`, `setToken` et `removeToken`
   - le token est stocké côté client (localStorage) ; le code doit être sûr côté SSR (vérifier `typeof window`)
   - un `AuthProvider` (Context) et un hook `useAuth` exposant l'utilisateur, `login`, `logout` et l'état de chargement
   - comme le token n'est pas dans un cookie, il n'est pas lisible par le `middleware.ts` de Next.js : prévois un composant de garde côté client (`AuthGuard`) pour protéger les routes, et dis-moi clairement quelles pages sont des Client Components à cause de cela
4. **Types** (`src/types`) : une interface TypeScript par schéma défini dans `components/schemas` de l'OpenAPI, plus les types génériques (réponse paginée, erreur de validation Laravel, réponse d'authentification, etc.).
5. **Composants** (`src/components`) : séparer les composants génériques et réutilisables (`ui/`), les composants de mise en page (`layout/`) et les composants spécifiques à chaque fonctionnalité (`features/<ressource>/`).
6. **Logique réutilisable** : `src/hooks` pour les hooks personnalisés, `src/utils` ou `src/lib` pour les helpers, `src/constants` pour les constantes (routes, endpoints).
7. **Configuration** : variables d'environnement typées et centralisées dans un fichier (ex. `src/config/env.ts`).

Respecte les conventions suivantes :
- noms de dossiers en `kebab-case`, composants en `PascalCase`
- séparer Server Components et Client Components (`"use client"` uniquement quand c'est nécessaire)
- ne pas surcharger la structure : ne crée que ce qui correspond réellement aux endpoints du fichier `openapi.yaml`

**Arrête-toi ici et attends ma validation avant de créer quoi que ce soit.**

## Étape 3 : Génération (après ma validation)

Une fois l'arborescence validée :

1. Crée tous les dossiers et fichiers de la structure.
2. Crée le fichier `.env` avec au minimum :
```
   NEXT_PUBLIC_API_URL=http://localhost:8000/api/v2
```
   Ajoute les autres variables nécessaires déduites de l'OpenAPI (ex. nom de la clé de stockage du token). Crée aussi un fichier `.env.example` et vérifie que `.env` est bien dans le `.gitignore`.
3. Écris le contenu de base de :
   - l'instance axios et ses intercepteurs (Bearer token, sans `withCredentials`)
   - le module de gestion du token, l'`AuthProvider` et le hook `useAuth`
   - le composant `AuthGuard`
   - le fichier de configuration des variables d'environnement
   - un service complet pour une ressource, comme modèle pour les autres
   - les types TypeScript correspondant aux schémas de cette ressource
   - les fichiers de service des autres ressources, avec les signatures de fonctions typées, sans logique superflue
4. Vérifie que Tailwind CSS et DaisyUI sont correctement installés et configurés. Sinon, donne-moi les commandes et fais la configuration.

# Format de réponse attendu

- Réponds en français.
- Commence par le résumé de l'étape 1, puis l'arborescence de l'étape 2, et attends ma réponse.
- N'invente aucun endpoint ni schéma : tout doit venir de `openapi.yaml`.
- Si une information du fichier est ambiguë ou manquante (pagination, format du token, expiration, etc.), pose-moi la question au lieu de supposer.