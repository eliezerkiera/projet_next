# Règles d'implémentation de l'API (V2)

Fichier à lire EN ENTIER avant toute tâche touchant l'API, l'authentification, les cookies,
`proxy.ts` ou la locale. Il complète `CLAUDE.md` (la règle « plan avant modification »
s'applique toujours).

## 1. Contexte de l'API
- Serveur Laravel, Sanctum en mode **tokens Bearer**, version **V2**.
- Base : `/api/v2` (auth : `/api/v2/auth`, pays : `/api/v2/countries/active`).
  Le préfixe est défini UNE SEULE FOIS dans `lib/api/` (variable d'environnement).
  N'utilise jamais d'endpoint ou d'exemple d'une autre version (V1).
- Authentification : access token + refresh token à rotation, login avec 2FA par OTP email.
- Durées de vie : **access token = 1 jour, refresh token = 1 mois** (supposé 30 jours,
  À CONFIRMER). Ne déduis aucune autre durée.
- Le pays n'est PAS détecté par IP : n'implémente jamais de géolocalisation IP.
- Aucun contrat OpenAPI : `lib/api/ENDPOINTS.md` tient lieu de contrat.
  Statuts : « observé » (vraie réponse vue) ou « documenté » (guide de test, non vérifié).

## 2. Ne jamais deviner
- N'invente JAMAIS un endpoint, paramètre, nom de champ ou forme de réponse.
  Sans source observée, demande-la-moi.
- Sources acceptables, par ordre de fiabilité :
  1. Une vraie réponse que je te colle (JSON réel + code HTTP, token retiré)
  2. Du code existant qui consomme déjà l'endpoint
  3. Les types présents dans `lib/api/types/`
  Le guide de test est une source « documentée », pas « observée ».
- Dans ton plan, liste chaque endpoint avec « observé (source : ...) » ou
  « supposé (à valider) ». N'implémente pas tant qu'un endpoint est « supposé ».
- Avant un nouvel appel, demande : méthode, chemin, paramètres, exemple de réponse
  réussie ET d'erreur si absents.
- Mets à jour `lib/api/ENDPOINTS.md` à chaque nouvel endpoint ou écart constaté
  (méthode, chemin, auth, paramètres, exemple réel, erreurs, date, statut).
- Si une réponse réelle diffère de la doc, arrête-toi et signale l'écart au lieu de
  t'adapter en silence.
- Ne te fie pas au texte des `message` de l'API (il peut être incohérent) : base ta
  logique sur les codes HTTP et les champs structurés.
- Ne suppose pas les conventions Laravel standard (`{ data, links, meta }`, format
  d'erreur 422) : confirme-les avec une vraie réponse.

## 3. Appels et données
- Tous les appels passent par `lib/api/` : pas de `fetch` éparpillé dans les composants.
- L'URL de base vient d'une variable d'environnement, jamais en dur.
- Type les réponses et valide-les aux frontières avec Zod. Champs non confirmés comme
  toujours présents : `.optional()`/`.nullable()`. Pas de cast avec `as`.
- Gère explicitement : réponse non-OK, erreur réseau, timeout, liste vide, champs `null`.
  Aucun `fetch` sans gestion d'erreur.
- Parallélise les appels indépendants (`Promise.all`).
- Mutations (POST/PUT/PATCH/DELETE) : état de chargement, erreur affichée, revalidation
  des données concernées, pas de double soumission.
- Actions destructives (`DELETE /account`, `DELETE /devices`) : confirmation explicite
  dans l'UI, jamais automatiques.

## 4. Stockage des tokens
- `access_token` et `refresh_token` : UNIQUEMENT dans des cookies `httpOnly`, `Secure`
  (en production), `SameSite=Lax`, posés côté serveur.
- Durée des cookies (plafonds, jamais au-delà de la durée de vie réelle du token) :
  - access : `maxAge` 86 400 s (1 jour)
  - refresh : `maxAge` 2 592 000 s (30 jours, À CONFIRMER)
- Le `path` du cookie refresh est restreint au Route Handler de refresh si possible.
- Un cookie présent ne prouve pas un token valide (révocation côté serveur possible,
  expiration, rotation) : l'API fait foi.
- Jamais de token dans `localStorage`, `sessionStorage`, un état React, `NEXT_PUBLIC_`
  ou l'URL. Le navigateur n'appelle jamais l'API Laravel avec un token : les Client
  Components passent par un Route Handler proxy.
- Tout le code qui lit les cookies, ajoute `Authorization: Bearer` et gère le refresh
  vit dans `lib/api/`.
- N'oublie pas `await cookies()` (asynchrone en Next 16).

## 5. Refresh (rotation obligatoire)
- Chaque `POST /auth/refresh` renvoie un NOUVEAU refresh token et révoque l'ancien.
  Remplace les DEUX cookies immédiatement. Ne réutilise jamais l'ancien.
- Écriture des cookies : uniquement dans `proxy.ts`, un Route Handler ou une Server Action
  (jamais dans un Server Component).
- Déclenchement :
  - cookie access absent/expiré ET cookie refresh présent → refresh avant la requête ;
  - réponse `401` sur un appel authentifié → un refresh, puis rejeu de la requête UNE seule fois.
- Dans `proxy.ts`, limite le `matcher` (exclure assets statiques, et si possible les
  prefetch) pour éviter des refresh inutiles. Dans ton plan, précise comment le nouveau
  token est mis à disposition des Server Components de la MÊME requête (sinon ils
  verraient encore l'ancien cookie).
- Un seul refresh à la fois : des refresh parallèles avec le même token déconnectent
  l'utilisateur (le second appel échoue car l'ancien token est déjà révoqué). Un verrou en
  mémoire ne suffit pas si l'app tourne sur plusieurs instances : explique ta stratégie
  dans le plan. Si un refresh échoue, relis les cookies avant de déconnecter (une autre
  requête a peut-être déjà fait la rotation).
- Échec de refresh (`422` invalide / expiré / révoqué) : supprime les cookies, redirige
  vers la connexion, sans boucle de redirection.
- Ne dépends pas du texte du message d'erreur pour distinguer « expiré » et « invalide ».
- À CONFIRMER : la rotation prolonge-t-elle la durée du refresh token (glissante) ou
  l'échéance du mois est-elle absolue ? Ne suppose pas.

## 6. Flux d'authentification (détails dans `ENDPOINTS.md`)
- Login : `POST /login` (email + mot de passe) → `challenge_token` →
  `POST /login/verify-code` (`challenge_token`, `code`, `device_name`) → access + refresh token.
- Inscription : `send-code` → `verify-code` → `verification_token` → `complete`.
- Mot de passe oublié : `send-code` → `verify-code` → `reset_token` → `reset`.
- `challenge_token`, `verification_token`, `reset_token` : temporaires et sensibles.
  Conserve-les côté serveur (cookie `httpOnly` de courte durée ou état serveur),
  ne les logge jamais, ne les confonds jamais.
- Chaque flux OTP a un « renvoyer le code » : prévois un délai avant renvoi dans l'UI.
- `device_name` est requis à la connexion et à l'inscription : utilise une constante
  cohérente (ex. « Web - Next.js »).
- Logout : `POST /auth/logout`, puis suppression des cookies même si l'appel échoue.
- Utilisateur courant : `GET /auth/me`, jamais reconstruit depuis le token.

## 7. Protection des routes
- `proxy.ts` pour la redirection rapide (présence d'un cookie), ET vérification réelle
  dans le layout/la page/l'action.
- Les Server Actions et Route Handlers privés vérifient eux-mêmes l'authentification.
- Masquer un élément côté client n'est pas une protection.
- Un `401`/`403` de l'API fait foi : une vérification front ne remplace jamais
  l'autorisation côté API.

## 8. Codes d'erreur
- `401` : non authentifié → un refresh (une fois), sinon connexion.
- `403` : accès refusé → message dédié, pas de redirection vers le login.
- `422` : validation OU échec de refresh/OTP. Le format `{ message, errors }` n'est pas
  entièrement confirmé : gère les deux cas après vérification sur une vraie réponse.
- `429` : message à l'utilisateur, pas de retry en boucle (limites non documentées).
- `5xx` / réseau : message générique, aucun détail technique affiché.

## 9. Cache
- Aucune donnée authentifiée en cache partagé : `cache: 'no-store'` (ou API dynamiques)
  sur tout appel avec token. Pas de `use cache` sur des données dépendant d'un utilisateur.
  Justifie toute exception dans le plan. Risque n°1 : un utilisateur qui voit les
  données d'un autre.
- Pour les données publiques (ex. `/countries/active`) : vérifie dans la doc de Next 16
  le mécanisme de cache adapté et inclus la locale dans la clé (la réponse dépend de la langue).

## 10. Langue et pays
- Envoie `Accept-Language` avec la locale courante. Priorité côté API :
  `X-Locale-Override` > `Accept-Language` > préférence utilisateur > langue du pays actif.
- `X-Locale-Override` est une priorité ponctuelle de requête, pas une préférence.
  Pour persister : `PATCH /auth/locale` (`country_id` et/ou `language_id`, ou `reset_to_auto: true`).
- `GET /countries/active` renvoie `country_selection_required` : affiche un sélecteur de
  pays UNIQUEMENT si c'est `true`.
- Les `country_id` / `language_id` viennent de l'API : ne jamais les coder en dur.

## 11. Environnements et OTP
- Les codes OTP visibles dans les logs Laravel ou la table `verification_codes` existent
  en test uniquement. Ne les utilise jamais pour contourner la 2FA, et jamais sur un
  environnement réel.

## 12. Points ouverts (demande-moi, ne devine pas)
- Durée exacte du refresh token (30 jours ?) et rotation glissante ou absolue ?
- `POST /auth/logout` exige-t-il le refresh token dans le corps ?
- Rôle exact de `POST /auth/verify-pending-email` (route publique) ?
- `DELETE /auth/account` demande-t-il le mot de passe ?
- Limites de débit et délai de renvoi des OTP (`429`) ?
- Format exact des erreurs `422` de validation ?
- Format de pagination des futurs endpoints métier ?