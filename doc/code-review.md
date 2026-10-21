# Code Review — Rapido SaaS Boilerplate

> Revue du code du projet, réalisée le 21 octobre 2026.

## Verdict général

Un **boilerplate SaaS correct et sympathique pour démarrer vite** : la stack est cohérente (Next.js 14 App Router + TypeScript + Tailwind + shadcn/ui + NextAuth + MongoDB + Stripe + PostHog), l'arborescence est claire (`app/`, `components/`, `lib/`, `utils/`), et il y a déjà les bases utiles (thème clair/sombre, pages légales, magic-link auth, page pricing). En l'état, c'est plutôt un **good starting point** qu'un produit fini — ce qui est normal pour un boilerplate, mais certains points méritent d'être corrigés avant mise en production.

## Points forts

- Connexion Mongo partagée bien gérée (`src/lib/mongodb.ts`) avec le global en dev pour le HMR — le pattern officiel Vercel.
- Zod + react-hook-form installés pour la validation des formulaires.
- `metadata`/`viewport` séparés, providers correctement imbriqués (Auth → PostHog → Theme).
- Modèle User avec `models.User || mongoose.model(...)` pour éviter le recompilation en dev.

## Points faibles (par ordre de gravité)

### Sécurité / fiabilité

1. **Dashboard protégé uniquement côté client** — aucun `middleware.ts`. Pire : dans `src/app/dashboard/page.tsx`, `useSession()` renvoie `data: null` pendant le chargement, donc `if (!session) redirect('/')` se déclenche **y compris pendant le loading** — l'utilisateur risque d'être renvoyé à l'accueil alors qu'il est connecté. Il faudrait tester `status === "unauthenticated"` ou, mieux, un `middleware.ts` + `getServerSession` côté serveur.
2. **API Stripe sans validation ni webhook** (`src/app/api/checkout_sessions/route.ts`) — `amount` est pris brut dans le body (négatif, 0, montant absurde accepté), alors que le pricing n'envoie que `5` de toute façon. Pas de webhook Stripe ⇒ paiements non vérifiés, non enregistrés, et **aucun mécanisme d'abonnement** alors qu'il s'agit d'un boilerplate SaaS. Ajouter un schéma Zod (borné), des prix fixes côté serveur, et un endpoint `webhooks/stripe`.
3. **Double couche d'accès DB** : mongoose (`src/utils/config/db.ts`) *et* driver natif (`src/lib/mongodb.ts`) — il faut choisir. Côté mongoose : `process.exit()` sur erreur de connexion **tuerait le serveur Next**, et `connect()` n'est pas mis en cache (reconnexion à chaque appel).
4. **Config d'email fragile** : `EMAIL_SERVER=smtp.example.com` sans protocole — NextAuth attend un format `smtp://user:pass@host:port` ou un objet `{host, port, auth}`. À corriger dans `.env.template`.

### Qualité de code

5. `src/app/Providers.js` en JavaScript dans un projet TypeScript, avec un `posthog.init()` en side-effect au niveau module.
6. `singInSchema` (typo → `signInSchema`) dans `src/lib/validators.ts`, `bcryptjs` installé mais inutile (auth uniquement par magic link), `next.config.mjs` vide, `metadata.title: "Create Next App"` dans `src/app/layout.tsx`.
7. Pas de `loading.tsx` / `error.tsx`, pas de tests, pas de CI (`.github/` absent).

### Projet / documentation

8. Le README a des restes d'un autre projet : *"To develop Influence locally"*, référence à `.env.example` alors que le fichier s'appelle `.env.template`, features vagues (*"Networking… And more…"*) qui ne décrivent pas ce que fait réellement le code (landing + auth + dons).
9. Stack datée pour octobre 2026 (Next 14, next-auth v4, Tailwind 3) — utilisable, mais à prévoir en migration.

## Priorités

Si on ne devait corriger que 3 choses :

1. Ajouter un `middleware.ts` pour protéger `/dashboard` (et corriger le bug de redirection au chargement).
2. Sécuriser l'API Stripe : validation Zod + prix fixes côté serveur + webhook.
3. Unifier l'accès à MongoDB (supprimer l'une des deux couches) et retirer le `process.exit()`.
