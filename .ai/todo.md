# Mission : proposer et créer la structure de la couche d'accès à l'API

## Contexte
Ce projet Next.js (App Router) doit consommer une API REST Laravel documentée dans `openapi.yaml` (à la racine du projet ; demande-moi le chemin s'il est introuvable).
URL de base de l'API : `http://localhost:8000/api/v2`
Authentification : **jetons personnels Laravel Sanctum, envoyés dans l'en-tête `Authorization: Bearer <token>`**. Ce n'est PAS le mode Sanctum « SPA » par cookie : aucun cookie de session ni jeton CSRF Laravel n'est utilisé entre Next.js et l'API.

Lis d'abord `AGENTS.md` et respecte-le intégralement (plan avant modification, validation, changelog, etc.). Cette mission s'y ajoute, elle ne le remplace pas.

## Objectif
Concevoir, puis créer après ma validation, **uniquement la structure de dossiers et les fichiers fonctionnels** nécessaires pour consommer l'API : client HTTP, services, types, hooks, schémas de validation, gestion d'erreurs, gestion sécurisée du jeton, configuration.

**Hors périmètre, n'y touche pas** : pages, layouts, composants UI, formulaires, styles, middleware, pages de connexion. Si un de ces éléments te semble nécessaire plus tard, liste-le dans « À prévoir plus tard » du plan, sans le créer.

## Étape 1 : analyse de `openapi.yaml` (aucune création de fichier)
Lis le fichier en entier et relève :
- les ressources (regroupement par `tags` et par chemins) et leurs opérations (méthode, chemin, paramètres, corps, réponses) ;
- les `components/schemas` réutilisés ;
- les `securitySchemes`, les routes protégées et, si la spec les mentionne, les **abilities** Sanctum associées aux routes ;
- les endpoints d'authentification (connexion, déconnexion / révocation du jeton, utilisateur courant) et ce qu'ils renvoient ;
- la forme des réponses : enveloppe (`data`, `meta`, `links`), pagination, format des erreurs (dont les erreurs de validation 422) ;
- tout élément ambigu ou incohérent dans la spec (signale-le, ne le corrige pas en silence).

## Étape 2 : plan à me soumettre
Suis le format de plan de `AGENTS.md` et ajoute :
1. **L'arborescence proposée**, avec une ligne d'explication par dossier et par fichier.
2. **Un tableau de correspondance** : endpoint OpenAPI → fichier / fonction / hook qui le couvrira.
3. **Les dépendances proposées**, hors axios déjà validé, chacune avec sa justification et une alternative sans dépendance.
4. **Le modèle de stockage et de transmission du jeton**, avec la recommandation de la section « Sécurité » ci-dessous et ses alternatives, pour que je choisisse.
5. **Les autres décisions à valider** :
   - librairie de gestion des données côté client pour les hooks (ex. TanStack Query, SWR, ou aucune) ;
   - types TypeScript écrits à la main ou générés depuis la spec (ex. `openapi-typescript`).
6. **Les questions ouvertes** et, si la spec est volumineuse, un découpage par lots (infrastructure et sécurité d'abord, puis ressource par ressource).

**Ne crée aucun fichier avant ma validation explicite.**

## Sécurité (non négociable)

### Jeton Sanctum : stockage et transmission
- **Interdit** : stocker le jeton dans `localStorage`, `sessionStorage`, un cookie lisible par JavaScript, une variable `NEXT_PUBLIC_*`, une URL ou un paramètre de requête. Ne le passe jamais en props à un Client Component. Ne le journalise jamais.
- **Approche recommandée à proposer dans le plan** : le jeton est conservé côté serveur Next.js dans un **cookie `HttpOnly`** posé et lu uniquement par du code serveur (Server Actions, Route Handlers, Server Components). Le serveur Next.js ajoute l'en-tête `Authorization: Bearer` lors de l'appel à Laravel. Le navigateur ne voit jamais le jeton. (Ce cookie appartient à Next.js ; il n'a rien à voir avec les cookies de session Sanctum.)
  - Attributs du cookie : `HttpOnly`, `Secure` en production, `SameSite=Lax` au minimum (`Strict` si le parcours le permet), `Path=/`, durée alignée sur la durée de vie du jeton.
  - Conséquence à traiter : comme un cookie est envoyé automatiquement, les mutations exposées par Next.js (Route Handlers) doivent **vérifier l'en-tête `Origin`** (les Server Actions le font déjà nativement). Pas de mutation en `GET`.
  - Lectures via Server Components ; mutations via Server Actions ; accès depuis le client via des Route Handlers ciblés si les hooks en ont besoin. **Pas de proxy générique non filtré** (`/api/proxy/[...path]`) : s'il y en a un, il n'accepte que les chemins et méthodes présents dans la spec (liste blanche).
- Présente dans le plan au moins une alternative (ex. jeton en mémoire côté client, appels directs du navigateur vers Laravel), avec ses risques : exposition au vol par XSS, perte du jeton au rechargement, configuration CORS nécessaire côté Laravel. **Ne la mets en œuvre que si je la choisis.**
- Les modules qui manipulent le jeton importent `server-only` pour qu'un import accidentel côté client fasse échouer le build. Si ce paquet doit être ajouté, mentionne-le dans les dépendances proposées.
- Fournis des fonctions dédiées et centralisées pour lire, écrire et supprimer le jeton. Aucun autre fichier n'accède directement au cookie.
- Déconnexion : appelle l'endpoint de révocation du jeton s'il existe dans la spec, puis supprime le cookie.

### Client axios
- **Une seule instance** par contexte d'exécution, configurée à un seul endroit : `baseURL` issue de l'env, `timeout` raisonnable, limites `maxContentLength` et `maxBodyLength`, **`maxRedirects: 0`** côté serveur (l'API ne doit pas rediriger, et un jeton ne doit jamais suivre une redirection vers un autre hôte).
- En-têtes par défaut : `Accept: application/json` (sans lui, Laravel peut répondre par une redirection HTML au lieu d'un 401 JSON) et `Content-Type: application/json`.
- `withCredentials: false` : aucun cookie n'est envoyé à Laravel.
- **Les services n'appellent que des chemins relatifs construits par le code.** Jamais d'URL absolue ni d'URL fournie par l'utilisateur ou par une réponse d'API (risque de fuite du jeton et de SSRF). Si la version d'axios installée propose l'option `allowAbsoluteUrls`, passe-la à `false` ; sinon ajoute un garde-fou équivalent.
- Chaque segment de chemin dynamique (identifiants) est encodé avec `encodeURIComponent`.
- **Intercepteur d'erreurs** : normalise tout (réseau, 401, 403, 404, 422 avec détail par champ, 429 avec `Retry-After`, 5xx) en **un seul type d'erreur applicatif**, sans le message brut ni la trace du serveur.
  - **Les erreurs axios embarquent la configuration de la requête, donc l'en-tête `Authorization`.** Ne journalise ni ne renvoie jamais l'objet d'erreur brut. Le type d'erreur normalisé ne contient ni en-têtes, ni jeton, ni corps de requête.
  - 401 : le jeton est invalide ou expiré, le serveur le supprime et l'erreur typée permet à l'UI de réagir plus tard. 403 : permission ou ability manquante, **pas** de suppression du jeton.
- Nouvelles tentatives automatiques : uniquement pour les requêtes idempotentes (`GET`), en nombre limité, avec délai croissant. **Jamais** sur `POST`, `PUT`, `PATCH`, `DELETE`, ni sur un 4xx.

### Données et réponses
- Valide les entrées de chaque service **avant l'envoi** (validateur déjà présent dans le projet, ex. Zod). Cette validation sert l'expérience utilisateur : la validation de Laravel reste l'autorité.
- Traite les réponses de l'API comme des données non fiables. Propose dans le plan la validation à l'exécution des réponses sensibles (ex. `safeParse` du schéma) plutôt qu'un simple cast TypeScript.
- **Aucune donnée authentifiée dans un cache partagé entre utilisateurs** (cache de données Next.js, `unstable_cache`, `use cache`, cache de route statique). Les appels authentifiés rendent la route dynamique et ne sont pas mis en cache, ou sont isolés par utilisateur.
- Ne renvoie jamais au navigateur plus de champs que nécessaire (pas d'objet utilisateur brut de l'API passé tel quel aux composants clients).

### Configuration et journaux
- Variables d'environnement lues et validées **à un seul endroit** (fichier de configuration), jamais de `process.env` dispersé. Le nom de la variable d'URL de l'API n'a **pas** le préfixe `NEXT_PUBLIC_` sauf si l'approche choisie impose des appels depuis le navigateur.
- En production, refuse une URL d'API en `http://` (autorisée uniquement pour `localhost`). Si la variable est absente ou invalide, l'application échoue au démarrage avec un message explicite.
- Journaux (côté serveur uniquement) : méthode, chemin, statut, durée. **Jamais** le jeton, les mots de passe, les corps de requête ni les données personnelles.
- Aucune information interne de l'API (traces, messages SQL, noms de classes Laravel) n'est transmise à l'UI.

## Étape 3 : exécution (après validation)

### Règles d'architecture
- `src/app/` reste réservé au routage, aux Server Actions et aux Route Handlers. Aucune logique d'API dedans : ils appellent les services.
- Organisation **par ressource** (un module par ressource OpenAPI), avec le code partagé séparé et centralisé : client axios, gestion du jeton, erreurs normalisées, configuration, types communs.
- Les **services** sont des fonctions typées, sans dépendance à React. Ceux qui touchent au jeton sont `server-only`.
- Les **hooks** sont côté client uniquement (`"use client"`), ne font aucun appel HTTP direct vers Laravel et ne voient jamais le jeton : ils passent par les Server Actions ou Route Handlers prévus.
- Pas de barrel files (`index.ts`) en cascade ; un par module au maximum, si c'est utile.
- Nommage : fichiers en `kebab-case`, types en `PascalCase`.

### Variables d'environnement
- **Dérogation explicite à la section 6 de `AGENTS.md`, valable uniquement pour cette mission** : tu peux créer `.env` et `.env.example`, qui ne contiennent que l'URL de l'API (valeur ci-dessus, non secrète). Tu ne lis ni ne modifies aucun autre fichier d'environnement.
- Vérifie le `.gitignore` : `.env` doit être ignoré par git, et `.env.example` doit rester suivi (les modèles Next.js ignorent souvent tous les `.env*`). Si un ajustement est nécessaire, **inclus-le dans le plan** au lieu de le faire seul.

### Types et schémas
- Un type TypeScript pour chaque schéma pertinent de la spec, nommé comme dans la spec.
- Types génériques pour les enveloppes de réponse et la pagination, selon ce que Laravel renvoie réellement d'après la spec.
- Si le projet n'a pas encore de validateur, propose-le dans le plan, ne l'ajoute pas seul.

### Fidélité à la spec
- Une fonction de service typée par opération de la spec, ni plus ni moins. **N'invente aucun endpoint, paramètre ou champ.**
- Les paramètres de requête, de chemin et les corps sont typés d'après la spec.
- Chaque fonction de service porte un court commentaire en anglais (endpoint couvert, authentification requise, particularité éventuelle), conformément aux règles de commentaires de `AGENTS.md`.

## Livrable final
Quand tu as terminé, donne-moi :
1. l'arborescence **réellement créée** ;
2. la liste des endpoints **couverts** et, le cas échéant, **non couverts** (avec la raison) ;
3. les résultats des vérifications de `AGENTS.md` (lint, types, build si pertinent) ;
4. **une checklist sécurité** confirmant point par point : jeton jamais exposé au client, cookie `HttpOnly`, aucune URL absolue possible, erreurs normalisées sans en-têtes, aucun cache partagé de données authentifiées, journaux sans données sensibles, variables validées au démarrage ;
5. les éléments à prévoir plus tard : UI, middleware de protection des routes, en-têtes de sécurité (CSP, etc.) dans `next.config`, limitation de débit sur la connexion, configuration CORS côté Laravel si les appels directs depuis le navigateur sont retenus.