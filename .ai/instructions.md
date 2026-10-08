# AGENTS.md

> Ce fichier s'adresse à **tous les agents de codage IA** (Claude Code, Codex, Cursor, Copilot, Gemini, etc.) qui interviennent sur ce dépôt.
> Plusieurs agents se relaient sur ce projet : ce fichier et le dossier `.ai/` sont la **source de vérité commune**.
> Lis ce fichier en entier avant toute modification. En cas de conflit entre ce fichier et tes habitudes par défaut, **ce fichier gagne**.

---

## 0. Éléments à compléter par l'humain

Les valeurs entre `<...>` sont à remplir une fois. Si l'une d'elles est encore vide, **demande-la, ne la devine pas**.

- **Nom du projet** : `projet_next`
- **Version de Next.js** : `16.3.5` (App Router)
- **Gestionnaire de paquets** : `npm`
- **UI** : `Tailwind CSS + DaisyUI`

---

## 1. Règles d'or (non négociables)

1. **Comprendre avant d'agir.** Lis les fichiers concernés, `.ai/handoff.md` et l'historique git récent avant d'écrire du code.
2. **Plan d'abord, code ensuite.** Tout changement passe par le workflow de la section 2 : plan, validation de l'humain, exécution.
3. **Fais la plus petite modification qui résout le problème.** Pas de refactoring, de renommage, de reformatage ou de « petite amélioration » non demandés. Si tu repères un autre problème, note-le dans `.ai/handoff.md` au lieu de le corriger.
4. **Ne réinvente pas l'existant.** Cherche d'abord un composant, un hook, un utilitaire ou un pattern déjà présent dans le dépôt et réutilise-le.
5. **Aucune dépendance ajoutée, aucun changement de stack ou d'architecture sans accord explicite.** Cela inclut : nouvelle librairie, passage au Pages Router, changement d'ORM ou de librairie UI, changement de gestionnaire de paquets, modification de la configuration (`next.config.*`, `tsconfig.json`, ESLint, Tailwind, CI).
6. **N'invente rien.** Pas d'API, de props, d'options de configuration ou de fonctions « qui devraient exister ». Si tu n'es pas sûr, lis le code, la documentation officielle ou `node_modules`, ou pose la question.
7. **Ne casse pas ce qui marche.** Si ton changement peut avoir un effet de bord ailleurs, vérifie-le (recherche des usages) ou signale-le dans le plan.
8. **Ne dis jamais « c'est fait » sans preuve.** Lance les vérifications de la section 2.3 et rapporte leur résultat réel.

---

## 2. Workflow de modification

### 2.1 Plan avant modification

Avant toute modification de code (nouveau fichier, édition, suppression, migration, refactor), **présente un plan concis, dans la langue de l'humain, avant d'écrire quoi que ce soit** :

```markdown
### Plan
- **Fichiers** : créés / modifiés / supprimés
- **Approche** : choix technique retenu (+ alternative écartée si pertinent)
- **Données** : migrations, seed, variables d'environnement (sinon « aucun impact »)
- **Dépendances / config** : ajout ou changement (sinon « aucun »)
- **Vérifications prévues** : tests ajoutés ou mis à jour, commandes à lancer
- **Risques / incertitudes** : effets de bord possibles, questions ouvertes
```

**Attends la validation explicite de l'humain avant d'exécuter le plan.** Le silence, ou l'absence de réponse, ne vaut pas validation.

Si, pendant l'exécution, tu t'aperçois que le plan doit changer de façon significative (fichiers non prévus, migration imprévue, nouvelle dépendance, approche différente), **arrête-toi et présente un plan mis à jour** avant de continuer.

**Exception : corrections triviales.** Typo, formatage, commentaire, sur un seul fichier et sans impact fonctionnel : pas de plan préalable ni d'entrée de changelog. En cas de doute, ce n'est pas trivial.

### 2.2 Pendant la modification

- **Commente les modifications importantes dans le code, en anglais**, pour qu'une autre personne (ou un autre agent) puisse s'y retrouver. Un commentaire explique le **pourquoi** (choix, contrainte, piège), pas le quoi.
- Ne commente pas l'évident. Les commentaires décrivent le code tel qu'il est, pas son historique : pas de « modifié par l'agent X », « ancien code », « fix du bug Y ». L'historique va dans le changelog et dans git.
- Pas de code mort, pas de `console.log` oubliés, pas de `TODO` sans entrée correspondante dans `.ai/handoff.md`.
- Ne supprime pas de code, de tests ou de commentaires que tu ne comprends pas : demande.
- Ne contourne pas une erreur au lieu de la comprendre (test désactivé, cast en `any`, exception attrapée et ignorée).
- **Après deux tentatives infructueuses sur le même problème, arrête-toi**, résume ce que tu as essayé et demande de l'aide.

### 2.3 Vérifications avant de conclure

Une tâche n'est terminée que si **toutes** ces vérifications passent, et que tu en rapportes le résultat :

1. `lint` : aucune erreur.
2. `tsc --noEmit` : aucune erreur de type.
3. `test` : tous les tests passent (ajoute ou mets à jour les tests pertinents pour le code modifié).
4. `build` : réussit si le changement touche le routage, la configuration, les dépendances ou le rendu serveur.
5. Le comportement demandé a été **vérifié** (test, exécution, ou raisonnement explicite si l'exécution est impossible).

Si une vérification échoue pour une raison **antérieure à ton changement**, ne la « corrige » pas en douce : signale-la dans `.ai/handoff.md`.
Si tu ne peux pas exécuter une vérification, **dis-le clairement** ; ne prétends pas qu'elle a réussi.

### 2.4 Mise à jour du changelog

Après toute modification **validée et terminée**, ajoute une entrée **en tête** de `.ai/changelog.md` (pas à la fin). Si le fichier n'existe pas, crée-le avec le titre `# Changelog`.

Format d'entrée (en français) :

```markdown
## [AAAA-MM-JJ] — {résumé court}
- {ce qui a changé et pourquoi, détail 1}
- {détail 2}
Agent : {nom de l'agent / modèle}
Fichiers : `src/app/invoices/page.tsx`, `src/lib/invoices.ts`
Migrations : `{chemin de la migration}` (ou « aucune »)
Dépendances / env : {paquets ou variables d'environnement ajoutés, ou « aucun »}
```

- 2 à 5 lignes de détail, orientées « ce qui a changé et pourquoi ».
- Ne jamais réécrire ou supprimer les entrées précédentes, **uniquement en ajouter**. Si une ancienne entrée est devenue fausse, ajoute une nouvelle entrée qui la corrige.
- Inclus la mise à jour du changelog dans le même commit que le code correspondant.

---

## 3. Commandes

Utilise **uniquement** ces commandes:

| Action | Commande |
|---|---|
| Installer | `npm install` |
| Développement | `npm dev` |
| Build de production | `npm build` |
| Lint | `npm lint` |
| Vérification des types | `npm exec tsc --noEmit` |
| Tests | `npm test` |

- N'utilise **jamais** un autre gestionnaire de paquets que celui du projet. Ne crée ni ne modifie de lockfile d'un autre gestionnaire.
- Ne lance pas de serveur de dev en tâche bloquante sans nécessité. Si un serveur tourne déjà, ne le relance pas.

---

## 4. Structure du projet

```
src/
├── app/                 # Routes (App Router), layouts, pages, route handlers
│   ├── (groupes)/       # Groupes de routes
│   ├── api/             # Route handlers (route.ts)
│   └── layout.tsx
├── components/
│   ├── ui/              # Composants UI génériques (boutons, inputs...)
│   └── <feature>/       # Composants spécifiques à une fonctionnalité
├── lib/                 # Utilitaires, clients (db, auth), helpers
├── hooks/               # Hooks React réutilisables
├── actions/             # Server Actions (si utilisées)
└── types/               # Types partagés
public/                  # Fichiers statiques
.ai/                     # Mémoire partagée des agents (changelog, handoff)
```

> **La structure réelle du dépôt prime sur ce schéma.** Si elle diffère, suis la structure réelle et signale l'écart. Ne déplace pas de fichiers pour « coller » à ce schéma.

- Un composant, un fichier. Fichiers en `kebab-case`, composants en `PascalCase`.
- Place le code au plus près de son usage ; ne le remonte dans `components/ui` ou `lib` que s'il est réellement réutilisé.
- Utilise les alias d'import de `tsconfig.json` (ex. `@/`), pas de chemins relatifs profonds.

---

## 5. Conventions de code

### TypeScript
- **TypeScript strict.** Interdits : `any`, `@ts-ignore`, `@ts-expect-error` sans justification en commentaire, assertions `as` pour faire taire une erreur.
- Type les entrées et sorties des fonctions exportées.
- Valide les données externes (formulaires, query params, corps de requête, réponses d'API) avec le validateur déjà utilisé dans le projet (ex. Zod). N'en introduis pas un nouveau.

### Next.js (App Router)
- **Server Components par défaut.** N'ajoute `"use client"` que si le composant a besoin d'état, d'effets, d'événements navigateur ou d'une librairie cliente. Place-le le plus bas possible dans l'arbre.
- Récupère les données **côté serveur** (Server Components, route handlers, Server Actions), pas avec `useEffect` + `fetch`, sauf besoin client justifié.
- **Vérifie la version de Next.js avant d'écrire du code** : certaines API changent selon la version majeure (ex. `params`, `searchParams`, `cookies()` et `headers()` asynchrones dans les versions récentes ; cache par défaut). En cas de doute, consulte la documentation officielle de la version installée.
- Ne mélange pas `app/` et `pages/`. Pas d'API du Pages Router (`getServerSideProps`, `getStaticProps`, `next/router`) dans `app/`.
- Utilise les composants natifs : `next/image`, `next/link`, `next/font`, `metadata` / `generateMetadata`.
- Gère les états **chargement** (`loading.tsx` ou Suspense), **erreur** (`error.tsx`) et **non trouvé** (`not-found.tsx`) pour les routes qui chargent des données.
- Les Server Actions et route handlers **valident les entrées et vérifient l'autorisation à chaque appel**, jamais seulement côté client.

### React
- Composants fonctionnels et hooks uniquement.
- Pas d'état dérivé stocké dans `useState` s'il peut être calculé. Pas de `useEffect` pour ce qui peut se faire au rendu ou dans un gestionnaire d'événement.
- Clés (`key`) stables dans les listes, jamais l'index si l'ordre peut changer.

### Style et UI
- Utilise le système de style existant. N'ajoute ni nouvelle librairie de style, ni CSS inline étendu.
- Accessibilité : HTML sémantique, `alt` sur les images, labels sur les champs, navigation clavier.
- Responsive par défaut (mobile d'abord).
- Les textes visibles par l'utilisateur sont dans la langue d'interface de la section 0. Le code, les noms et les commentaires sont en **anglais**.
- Respecte le style existant (formatage, nommage). Le formateur et le linter du projet font foi.

---

## 6. Sécurité et données

- **Ne jamais committer de secrets** (clés d'API, tokens, mots de passe, URL de base de données avec identifiants).
- **Ne jamais lire, afficher, modifier ou committer** `.env`, `.env.local` ni aucun fichier de secrets. Si une variable d'environnement est nécessaire, ajoute-la à `.env.example` (sans valeur réelle) et mentionne-la dans le plan et le changelog.
- Les variables `NEXT_PUBLIC_*` sont exposées au navigateur : **jamais de secret dedans**.
- **Pas de commandes destructrices** sans accord explicite : suppression massive de fichiers, `git reset --hard`, `git push --force`, `git clean`, réinitialisation ou migration destructive de la base de données, suppression de données.
- Ne touche pas à la base de production. Les migrations sont **créées** (et listées dans le plan), jamais appliquées en production par un agent.
- Ne désactive jamais une règle de sécurité ou de lint pour « faire passer » le code (CSP, auth, validation, ESLint, TypeScript).
- N'envoie pas de code ou de données du projet vers des services externes non prévus.

---

## 7. Git

- Langue des messages de commit : `<anglais | français>`.
- Format **Conventional Commits** (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`).
- Commits **petits et atomiques** : un commit = une intention.
- Ne travaille pas directement sur `main`. Branche `<type>/<description-courte>` (ex. `feat/page-profil`).
- Avant de commencer : `git status` et `git log --oneline -10`. Si le répertoire contient des modifications que tu n'as pas faites, **ne les écrase pas** et ne les inclus pas dans ton commit : signale-les.
- Ne committe, ne pousse et ne fusionne que si l'humain l'a demandé ou si le projet l'exige explicitement.
- N'amende pas et ne réécris pas un historique déjà partagé.

---

## 8. Quand t'arrêter et poser des questions

**Si tu ne comprends pas quelque chose, pose des questions** plutôt que de supposer. Arrête-toi en particulier si :

- la demande est ambiguë ou plusieurs interprétations plausibles sont coûteuses à défaire ;
- la tâche exige une nouvelle dépendance, un changement d'architecture ou une migration de données ;
- tu dois modifier du code sensible (authentification, paiement, permissions, données personnelles) ;
- le code existant contredit ces règles ou la demande ;
- les tests ou le build échouent pour une raison que tu ne comprends pas ;
- le travail d'un agent précédent te semble incomplet, contradictoire ou erroné.

Pose **une question claire, avec les options que tu envisages**, plutôt qu'un long message vague.

---

## 9. Mémoire partagée entre agents (`.ai/`)

Deux fichiers, deux rôles distincts :

| Fichier | Rôle | Écriture |
|---|---|---|
| `.ai/changelog.md` | **Historique permanent** de ce qui a été fait (section 2.4) | Ajout uniquement, en tête |
| `.ai/handoff.md` | **État courant** : où on en est, pièges, prochaines étapes | Réécrit à chaque fin de session |

### Au début de ta session
1. Lis `AGENTS.md`, puis `.ai/handoff.md`, puis les dernières entrées de `.ai/changelog.md`.
2. Fais `git status` et `git log --oneline -10` pour vérifier que ces fichiers correspondent à l'état réel du code.
3. S'ils divergent, **fais confiance au code** et signale l'écart dans `.ai/handoff.md`.
4. Reformule en deux lignes la tâche que tu vas faire. Si elle ne correspond pas à « Prochaines étapes », demande confirmation.

### À la fin de ta session (obligatoire)
Mets à jour `.ai/handoff.md` avec **uniquement des faits vérifiés**, selon ce modèle (si le fichier n'existe pas, crée-le) :

```markdown
# Handoff

## État actuel
- Branche : `<nom>`
- Dernier commit : `<hash> <message>`
- Build : passe / échoue / non vérifié
- Tests : passent / échouent / non vérifiés
- Dernière mise à jour : <AAAA-MM-JJ> par <agent>

## En cours / inachevé
- <élément> : <où j'en suis exactement, fichiers concernés>

## Problèmes connus et pièges
- <bug, dette technique, comportement surprenant, fichier à ne pas toucher>

## Points non vérifiés
- <ce que je n'ai pas pu tester ou exécuter>

## Prochaines étapes (par priorité)
1. <action concrète et précise>
2. <action suivante>

## Questions ouvertes pour l'humain
- <question>
```

Règles :
- **Factuel et vérifiable** : pas de « devrait marcher ». Écris ce que tu as réellement exécuté.
- **Actionnable** : « Prochaines étapes » doit permettre à un autre agent de reprendre sans te poser de question.
- **Concis** : ne recopie pas le code ; référence les fichiers et les commits. Ce qui a été fait va dans le changelog, pas ici.
- Ne laisse jamais le dépôt cassé en fin de session. Si le travail est incomplet, isole-le (branche, commit `wip:` explicite) et décris-le dans « En cours ».
- Ne supprime pas les informations d'un autre agent sans raison : complète ou corrige-les en expliquant pourquoi.

---

## 10. Réponse finale à l'humain

Termine chaque tâche par un court compte rendu :

1. **Ce que j'ai fait** (2-5 puces, avec les fichiers modifiés).
2. **Comment je l'ai vérifié** (commandes exécutées et résultats).
3. **Ce que je n'ai pas pu vérifier ou ce qui reste à faire.**
4. **Décisions ou questions** qui nécessitent ton attention.

Un échec ou une limite clairement exposés valent mieux qu'un faux succès.