# Règles de travail pour l'agent

## Contexte du projet
- Application Next.js (App Router, TypeScript strict) qui consomme une API REST externe.
- Pas de base de données locale, pas d'ORM, pas de système d'authentification propre.
- Gestionnaire de paquets : **npm** (n'utilise jamais pnpm/yarn, ne modifie que `package-lock.json` via npm).
- L'API est la source de vérité : ne jamais dupliquer sa logique métier côté front.

## Commandes
- Dev : `npm run dev`
- Lint : `npm run lint`
- Types : `npx tsc --noEmit`
- Tests : `npm test` (à adapter si aucun framework n'est configuré)
- Build : `npm run build`

## Structure (à adapter)
- `app/` : routes, layouts, `loading.tsx`, `error.tsx`
- `components/` : composants UI réutilisables
- `lib/api/` : client API, types et fonctions d'appel (seul endroit autorisé à faire des `fetch` vers l'API)
- `.ai/changelog.md` : journal des modifications

---

## Workflow de modification

### 0. Avant de planifier
- Explore le code existant : réutilise helpers, composants et patterns en place
  plutôt que d'en créer de nouveaux.
- Consulte la doc de la version de Next.js installée (`package.json`) pour tout ce qui
  touche au cache, au routing ou à `fetch`. Ne te fie pas à ta mémoire : les
  comportements par défaut ont changé entre versions.
- Si un contrat d'API existe (OpenAPI/Swagger, doc, exemples de réponses), lis-le
  avant de typer quoi que ce soit. Ne devine jamais la forme d'une réponse.
- Si la demande est ambiguë, pose tes questions (3 maximum, les plus bloquantes)
  avant de planifier.

### 1. Plan avant modification
Avant toute modification de code (nouveau fichier, édition, suppression, refactor),
présente un plan concis avant d'écrire quoi que ce soit :
- Fichiers créés/modifiés/supprimés
- Approche technique choisie (et alternative écartée si pertinent)
- Impact Next.js si applicable :
  - Server Component vs Client Component (justifier tout `"use client"`)
  - Stratégie de cache / revalidation (options de `fetch`, `revalidatePath`, `revalidateTag`)
  - Nouvelles routes, layouts, middleware ou Route Handlers
- Endpoints de l'API utilisés (méthode, chemin, paramètres, forme de réponse attendue)
- Nouvelles variables d'environnement (préciser lesquelles sont `NEXT_PUBLIC_`)
- Nouvelles dépendances éventuelles (à valider avant installation)
- Risques ou points d'incertitude

Attends ma validation explicite avant d'exécuter le plan.

Paliers :
- Correction triviale à un seul fichier sans impact fonctionnel (typo, formatage,
  commentaire) : pas de plan.
- Changement de moins de 3 fichiers sans impact cache/API/routing : plan en 2-3 lignes.
- Tout le reste : plan détaillé.

### 2. Pendant la modification
- Commente les modifications importantes pour qu'une autre personne puisse se retrouver.
- Commentaires en anglais, qui expliquent le *pourquoi* (choix, contrainte, piège),
  pas ce que le code fait déjà clairement.
- Respecte la structure et les conventions existantes (App Router, TypeScript strict, pas de `any`).
- Ne modifie que les fichiers prévus dans le plan validé. Pas de refactor, renommage ou
  "amélioration" non demandés. Les problèmes hors périmètre sont signalés à la fin, pas corrigés.
- Si le plan doit changer en cours de route, arrête-toi et redemande validation.
- Ne jamais exposer de secret ou de code serveur côté client.

### 3. Règles spécifiques à la consommation de l'API
- Tous les appels passent par `lib/api/` : pas de `fetch` éparpillé dans les composants.
- L'URL de base vient d'une variable d'environnement, jamais codée en dur.
- Si l'API exige une clé ou un token : appel côté serveur uniquement (Server Component
  ou Route Handler), jamais `NEXT_PUBLIC_`.
- Type les réponses. Pour les données non fiables ou susceptibles de dériver du contrat,
  valide aux frontières (par exemple avec Zod) plutôt que de caster avec `as`.
- Gère explicitement : réponse non-OK (4xx/5xx), erreurs réseau, timeout, liste vide,
  champs optionnels ou `null`. Aucun `fetch` sans gestion d'erreur.
- Choisis volontairement le comportement de cache de chaque appel (statique, revalidé,
  dynamique) et documente-le en commentaire.
- Pas de waterfalls inutiles : parallélise les appels indépendants (`Promise.all`).
- Ne jamais logger de données sensibles (tokens, données personnelles).
- Pour les mutations (POST/PUT/DELETE), prévois l'état de chargement, l'erreur
  affichée à l'utilisateur et la revalidation des données concernées.

### 4. Checklist UI (si composant ou page modifié)
- États loading, error et vide gérés.
- `next/image` pour les images, `"use client"` uniquement si nécessaire.
- Labels, `alt` et navigation clavier de base.

### 5. Sécurité et actions interdites
- Ne lis, ne modifie et n'affiche jamais les fichiers `.env*` (utilise `.env.example`).
- Pas de commande destructive (`rm -rf`, `git push --force`, `git reset --hard`)
  sans accord explicite.
- Pas de commit ni de push sauf demande explicite. Ne travaille jamais sur `main`.
- Pas d'appel à une API de production pour des tests ni de mutation sur des données réelles.
- Avant d'ajouter une dépendance : vérifie qu'elle existe, qu'elle est maintenue,
  et qu'aucune dépendance déjà présente ne couvre le besoin.

### 6. Vérification avant de conclure
Exécute et corrige :
`npx tsc --noEmit && npm run lint && npm test`
Pour toute modification touchant le routing, le cache, les Server/Client Components,
les appels API ou la config Next.js, exécute aussi `npm run build`.
Si une commande échoue et que tu ne peux pas la corriger, dis-le explicitement
au lieu de conclure. Ne prétends jamais qu'un test ou un build passe sans l'avoir exécuté.

### 7. Rapport final
Termine chaque tâche par :
1. Ce qui a été fait (et pourquoi).
2. Les commandes exécutées et leur résultat réel.
3. Ce qui n'a PAS été vérifié ou reste incertain (par exemple un endpoint non testé
   contre la vraie API).
4. Comment tester manuellement (URL, étapes).

### 8. Mise à jour du changelog
Après toute modification validée et terminée, ajoute une entrée en tête de
`.ai/changelog.md` (pas à la fin) avec :
- Date
- Résumé du travail effectué (2-5 lignes, orienté "ce qui a changé et pourquoi")
- Fichiers principaux touchés
- Endpoints API ajoutés ou utilisés, le cas échéant
- Nouvelles variables d'environnement ou dépendances, le cas échéant

Format d'entrée :
```markdown
## [2026-09-30] — {résumé court}
- {détail 1}
- {détail 2}
Fichiers : `app/products/page.tsx`, `lib/api/products.ts`
API : `GET /products?page=` (revalidé toutes les 60 s)
Env / deps : `API_BASE_URL` (serveur uniquement), `zod`
```

Ne jamais réécrire ou supprimer les entrées précédentes, uniquement ajouter.

### 9. Amélioration continue
Si je te corrige sur un point qui pourrait se répéter, propose une ligne à ajouter
à ce fichier de règles.


### 10. Propositions proactives
Si tu repères quelque chose qui mérite mon attention, propose-le. Ne l'implémente jamais
sans mon accord : une proposition n'est pas une autorisation de modifier.

Sujets qui méritent une proposition :
- Bug, incohérence ou fragilité repérés hors périmètre
- Duplication ou code qui gagnerait à être factorisé
- Risque de sécurité (secret exposé, donnée sensible loggée, `"use client"` qui fuit du code serveur)
- Problème de performance (waterfalls de requêtes, cache mal choisi, bundle alourdi, images non optimisées)
- Écart entre l'API réelle et les types ou la doc du projet
- Dépendance inutile, dépréciée ou redondante
- Règle qui manque dans ce fichier, si une erreur se répète
- Meilleure approche que celle que j'ai demandée (dis-le AVANT de planifier, pas après)

Format : regroupe les propositions dans une section « Propositions » à la fin de ta réponse.
Pour chacune :
- **Constat** : ce que tu as vu, avec le fichier concerné
- **Proposition** : ce que tu ferais
- **Effort / impact** : faible, moyen ou élevé, et ce que ça apporte
- **Urgence** : à faire maintenant, prochainement ou optionnel

Règles :
- Maximum 3 propositions par réponse, les plus utiles d'abord. Pas de remplissage :
  s'il n'y a rien de pertinent, n'écris rien.
- Ne répète pas une proposition que j'ai déjà refusée ou reportée.
- Ne propose pas de changements purement stylistiques ou de préférence personnelle.
- Distingue ce qui est un fait vérifié de ce qui est une intuition (« je n'ai pas vérifié, mais... »).
- Si une proposition change le périmètre de la tâche en cours, arrête-toi et demande
  ma décision avant de continuer.
- Pour une proposition non traitée, je peux te demander de l'ajouter à `.ai/backlog.md`
  (en tête de fichier, même logique que le changelog).