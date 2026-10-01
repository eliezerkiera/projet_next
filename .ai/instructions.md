# RÈGLE PRIORITAIRE : PLAN AVANT MODIFICATION

Cette règle prime sur toute autre consigne et sur ta tendance à agir directement.

Tant que je n'ai pas écrit « GO » (ou « valide ») dans un message, tu as INTERDICTION de :
- créer, modifier ou supprimer un fichier ;
- exécuter une commande qui modifie le projet (installation de dépendance, génération de code, etc.).

Tu peux lire des fichiers et lancer des commandes en lecture seule pour préparer ton plan.

Procédure :
1. Explore le code (lecture seule).
2. Présente le plan (voir « Plan avant modification »).
3. Termine ta réponse par : « En attente de ton GO. » et ARRÊTE-TOI. N'écris aucun code de modification dans cette réponse.
4. N'implémente qu'après mon « GO ». Si j'ajoute des remarques avec mon GO, applique-les telles quelles.

Si tu as déjà modifié des fichiers sans GO, arrête-toi, dis-le-moi et propose d'annuler.

Seule exception : typo, formatage ou commentaire dans un seul fichier, sans aucun impact
fonctionnel. En cas de doute, ce n'est PAS une exception.

---

# Fichiers de référence

- `.ai/api-rules.md` : règles d'implémentation de l'API (auth, tokens, refresh, locale, erreurs).
  **Lis-le EN ENTIER avant toute tâche qui touche à l'API, à l'authentification, aux cookies,
  au proxy ou à la locale.** Dis-moi que tu l'as lu en commençant ton plan.
- `lib/api/ENDPOINTS.md` : contrat vivant des endpoints V2 (aucun contrat OpenAPI n'existe).
- `.ai/changelog.md` : journal des modifications.
- `.ai/backlog.md` : propositions reportées (optionnel).

---

# Contexte du projet

- Application **Next.js 16** (App Router, TypeScript strict, Turbopack par défaut).
  Vérifie le numéro mineur dans `package.json`.
- Elle consomme une API REST externe (Laravel, V2). Pas de base de données locale, pas d'ORM.
- Gestionnaire de paquets : **npm** (jamais pnpm/yarn ; `package-lock.json` modifié uniquement via npm).
- L'API est la source de vérité : ne duplique jamais sa logique métier côté front.
- Le front ne gère pas les utilisateurs : l'authentification est déléguée à l'API.

## Commandes
- Dev : `npm run dev`
- Lint : `npm run lint` (doit appeler ESLint directement : `next lint` n'existe plus en v16)
- Types : `npx tsc --noEmit`
- Tests : `npm test` (À COMPLÉTER si aucun framework n'est configuré)
- Build : `npm run build` (ne lance PAS le lint en v16 : exécute-le séparément)

## Structure (à adapter)
- `app/` : routes, layouts, `loading.tsx`, `error.tsx`
- `components/` : composants UI réutilisables
- `lib/api/` : seul endroit autorisé à appeler l'API
- `lib/api/types/` : types et schémas Zod issus de réponses réelles
- `proxy.ts` : remplace `middleware.ts` en Next 16

## Particularités Next.js 16 (à respecter)
- `proxy.ts` remplace `middleware.ts` (ne crée jamais de `middleware.ts`).
- `cookies()`, `headers()`, `params` et `searchParams` sont asynchrones : toujours `await`.
- Un Server Component ne peut pas écrire de cookie : écriture uniquement dans
  `proxy.ts`, un Route Handler ou une Server Action.
- Le cache a changé (Cache Components en opt-in, signature de `revalidateTag`,
  `updateTag`/`refresh` pour les Server Actions). Ne te fie pas à ta mémoire :
  consulte la doc officielle de la v16 avant toute décision de cache, et dis-moi
  quelle page de doc tu as utilisée.
- N'utilise jamais de cache partagé (`use cache`, `revalidate`, rendu statique) pour des
  données dépendant d'un utilisateur.
- Si tu as un doute sur une API de Next 16, dis-le au lieu de deviner depuis des
  exemples des versions 13 à 15.

---

# Workflow de modification

## 0. Avant de planifier
- Explore le code existant : réutilise helpers, composants et patterns en place.
- Si la tâche touche l'API : lis `.ai/api-rules.md` et `lib/api/ENDPOINTS.md`.
- Si la demande est ambiguë, pose tes questions (3 maximum, les plus bloquantes) avant de planifier.
- Si tu vois une meilleure approche que celle demandée, dis-le ICI, avant de planifier.

## 1. Plan avant modification
Présente un plan concis :
- Fichiers créés/modifiés/supprimés
- Approche technique choisie (et alternative écartée si pertinent)
- Impact Next.js si applicable : Server vs Client Component (justifier tout `"use client"`),
  stratégie de cache, nouvelles routes/layouts/`proxy.ts`/Route Handlers/Server Actions
- Si l'API est touchée : endpoints utilisés avec leur statut
  (« observé » ou « supposé »), conformément à `.ai/api-rules.md`
- Nouvelles variables d'environnement (préciser lesquelles sont `NEXT_PUBLIC_`)
- Nouvelles dépendances éventuelles (à valider avant installation)
- Risques ou points d'incertitude

Paliers :
- Correction triviale à un seul fichier sans impact fonctionnel : pas de plan (voir règle prioritaire).
- Moins de 3 fichiers sans impact cache/API/auth/routing : plan en 2-3 lignes.
- Tout le reste : plan détaillé.

## 2. Pendant la modification
- Commentaires en anglais, qui expliquent le *pourquoi* (choix, contrainte, piège),
  pas ce que le code fait déjà clairement.
- Respecte la structure et les conventions existantes (App Router, TypeScript strict, pas de `any`).
- Ne modifie que les fichiers prévus dans le plan validé. Pas de refactor, renommage ou
  « amélioration » non demandés. Les problèmes hors périmètre sont signalés à la fin, pas corrigés.
- Si le plan doit changer en cours de route, arrête-toi et redemande validation.
- Ne jamais exposer de secret ou de code serveur côté client.

---

# Checklist UI (si composant ou page modifié)
- États loading, error et vide gérés.
- `next/image` pour les images, `"use client"` uniquement si nécessaire.
- Labels, `alt` et navigation clavier de base.
- Formulaires : état de chargement, erreurs de validation par champ, erreur générique,
  pas de double soumission.

# Sécurité et actions interdites
- Ne lis, ne modifie et n'affiche jamais les fichiers `.env*` (utilise `.env.example`).
- Ne jamais logger ni afficher un token (access, refresh, challenge, verification, reset),
  un code OTP, un mot de passe, un cookie ou un header `Authorization`, même en debug.
- Pas d'identifiants ni de tokens de test en dur dans le code.
- Pas de commande destructive (`rm -rf`, `git push --force`, `git reset --hard`) sans accord explicite.
- Pas de commit ni de push sauf demande explicite. Ne travaille jamais sur `main`.
- Pas d'appel à une API de production ni de mutation sur des données réelles.
- Avant d'ajouter une dépendance : vérifie qu'elle existe, qu'elle est maintenue, et
  qu'aucune dépendance déjà présente ne couvre le besoin.

# Vérification avant de conclure
Exécute et corrige : `npx tsc --noEmit && npm run lint && npm test`
Pour toute modification touchant le routing, `proxy.ts`, le cache, les Server/Client
Components, les cookies, l'auth, les appels API ou la config Next.js, exécute aussi `npm run build`.
Si une commande échoue et que tu ne peux pas la corriger, dis-le au lieu de conclure.
Ne prétends jamais qu'un test ou un build passe sans l'avoir exécuté.

# Propositions proactives
Si tu repères quelque chose qui mérite mon attention, propose-le. Ne l'implémente jamais
sans mon accord.

Sujets pertinents : bug ou fragilité hors périmètre, duplication, risque de sécurité,
problème de performance, écart entre l'API réelle et `ENDPOINTS.md`, dépendance inutile,
règle manquante dans ces fichiers, meilleure approche que celle demandée.

Format (section « Propositions » en fin de réponse), pour chacune :
- **Constat** (avec le fichier concerné)
- **Proposition**
- **Effort / impact** (faible, moyen, élevé)
- **Urgence** (maintenant, prochainement, optionnel)

Règles :
- Maximum 3 par réponse, les plus utiles d'abord. S'il n'y a rien de pertinent, n'écris rien.
- Ne répète pas une proposition refusée ou reportée. Pas de changements purement stylistiques.
- Distingue fait vérifié et intuition (« je n'ai pas vérifié, mais... »).
- Si une proposition change le périmètre en cours, arrête-toi et demande ma décision.
- Sur demande, ajoute les propositions reportées en tête de `.ai/backlog.md`.

# Rapport final
Termine chaque tâche par :
1. Ce qui a été fait (et pourquoi).
2. Les commandes exécutées et leur résultat réel.
3. Ce qui n'a PAS été vérifié ou reste incertain, dont les endpoints et formes de
   réponse non confirmés par une vraie réponse de l'API.
4. Comment tester manuellement (URL, étapes).
5. Propositions, si pertinent.

# Mise à jour du changelog
Après toute modification validée et terminée, ajoute une entrée en tête de
`.ai/changelog.md` (pas à la fin) avec : date, résumé (2-5 lignes, « ce qui a changé et
pourquoi »), fichiers principaux touchés, endpoints API ajoutés ou utilisés, nouvelles
variables d'environnement ou dépendances.

Format :
```markdown
## [2026-09-30] — {résumé court}
- {détail 1}
- {détail 2}
Fichiers : `app/login/page.tsx`, `lib/api/auth.ts`
API : `POST /auth/login` (observé), `POST /auth/login/verify-code` (documenté)
Env / deps : `API_BASE_URL` (serveur uniquement), `zod`
```

Ne jamais réécrire ou supprimer les entrées précédentes, uniquement ajouter.

# Amélioration continue
Si je te corrige sur un point qui pourrait se répéter, propose une ligne à ajouter à ces fichiers.