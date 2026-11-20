# AGENTS.md — Learnity

Mémoire du projet pour les agents de code (Claude Code). À lire avant toute tâche.
Si une règle ici contredit une demande, signale-le avant d'agir. Garde ce fichier sous 200 lignes.

## Le produit
- Plateforme de formation à la tech (modèle Believemy / OpenClassrooms) : web et mobile d'abord, puis IA, web design, e-commerce.
- Mono-éditeur : l'équipe crée tous les cours et encaisse seule. Pas de formateurs externes en V1.
- Public : Sénégal puis Afrique francophone. Interface en français, tutoiement.
- Promesse (proposition) : « Le déclic qui lance ta carrière tech. »
- Deux univers sur un seul moteur de cours (`Course.kind`) :
  - **Formations** (`training`) : catalogue arbre libre domaine › technologie + tags ; niveaux `beginner` / `intermediate` / `advanced`.
  - **Renforcement** (`reinforcement`) : étudiants UNCHK, université › filière › semestre › UE (`AcademicUnit`) ; au lancement filière IDA, S1–S5 (S6 = projet, non couvert), avec prépa examen et séances live.
- Ne jamais coder en dur un domaine, une université, une filière ou une UE : tout vient du référentiel en base.
- Docs de référence : `docs/PRD.md` (exigences avec ID, v1.4), `docs/PLAN_V1.md` (architecture, données, paiements), maquette « Learnity — Maquette V1 » (29 écrans).
- Cite l'ID d'exigence (ex. `PAY-04`) dans les commits, PR et noms de tests.

## Stack
- Next.js 16 (App Router), React 19, TypeScript `strict`.
- Monorepo pnpm + Turborepo. Node 20+.
- Tailwind CSS 4 + shadcn/ui, thémé avec les tokens Learnity (`packages/ui`).
- Zod pour toute validation. Drizzle ORM + PostgreSQL. Auth : Better Auth (ou Supabase Auth, décision sprint 0).
- Vidéo : Bunny Stream ou Mux (URLs signées). Fichiers : stockage S3 compatible (R2).
- E-mails : Resend + React Email. Tests : Vitest, Playwright. Suivi : Sentry, PostHog.

## Commandes
```bash
pnpm install          # dépendances
pnpm dev              # app web en local
pnpm build            # build de tout le monorepo
pnpm lint             # ESLint
pnpm typecheck        # tsc --noEmit sur tous les packages
pnpm test             # Vitest
pnpm test:e2e         # Playwright
pnpm db:generate      # migrations Drizzle depuis le schéma
pnpm db:migrate       # appliquer les migrations
pnpm db:seed          # données de démo
```
Avant de dire qu'une tâche est finie : `pnpm typecheck && pnpm lint && pnpm test` doivent passer.

## Structure
```text
apps/web/src/app/(marketing)   pages publiques SEO (accueil, formations, renforcement, premium, blog)
apps/web/src/app/(auth)        connexion, inscription, mot de passe
apps/web/src/app/(app)         espace apprenant (dont renforcement/mon-semestre, [ue], [ue]/examen)
apps/web/src/app/admin         back-office (rôle admin)
apps/web/src/app/api           webhooks/[provider], cron/[job]
apps/web/src/features          UI par module (course-player, quiz-runner, exam-prep, academic-picker, checkout)
apps/web/src/actions           Server Actions fines
packages/core/src/<module>     métier pur : catalog, academic, examprep, learning, quiz, billing, gamification, community, notifications
packages/db                    schéma Drizzle, migrations, seed
packages/ui                    design system Learnity
packages/emails                modèles React Email
packages/config                tsconfig, eslint, preset Tailwind
```
Gabarit d'un module `core` : `schemas.ts` · `service.ts` · `repository.ts` · `policies.ts` · `events.ts` · `*.test.ts`.

## Règles d'architecture (non négociables)
1. Dépendances à sens unique : `app` → `core` → `db`. Un composant ou une page n'importe jamais `packages/db`.
2. `packages/core` n'importe ni React ni Next.js.
3. Seul `repository.ts` parle à la base dans un module.
4. Toute écriture = Server Action (ou Route Handler) qui : valide avec Zod → vérifie la session et le rôle → appelle un service `core`.
5. Server Components par défaut. `"use client"` seulement pour l'interactivité réelle (lecteur, quiz, formulaires).
6. Les modules communiquent par événements (`lesson.completed`, `quiz.passed`, `payment.succeeded`), pas par imports croisés.
7. Variables d'environnement validées au démarrage avec Zod ; aucun secret côté client (`NEXT_PUBLIC_` = public).
8. Pas de nouvelle dépendance sans le dire et l'expliquer dans la PR.

## Règles métier à ne jamais casser
- **Accès** : uniquement via `getAccess()` dans `core/learning`, côté serveur. Jamais de `if (user.isPremium)` dans un composant.
- Aperçu (formations seulement) : `previewRank 1` sans compte, `2–3` avec compte gratuit, le reste exige un `Entitlement` actif. Renforcement : aucun aperçu, scope `reinforcement` requis.
- Quiz, test final, exercices corrigés, PDF, code source, XP et classement exigent un droit payant (Premium ou achat du cours).
- Scopes d'`Entitlement` : `all_courses` (formations), `course` (achat à vie), `reinforcement`. Premium seul n'ouvre pas le renforcement ; l'offre `bundle_monthly` accorde `all_courses` + `reinforcement`.
- Refus d'accès typé : `premium_required` (formations) ou `reinforcement_required` → l'UI renvoie vers `/renforcement#tarifs`.
- Vidéos et PDF : URL signée courte, générée seulement après `getAccess()`.
- **Argent** : montants en entiers FCFA (`XOF`). Le prix est lu en base, jamais reçu du client.
- **Seul le webhook fait foi.** La page de retour affiche « confirmation en cours » et lit le statut serveur.
- Webhook : vérifier la signature, traiter de façon idempotente sur `providerRef`, stocker `rawPayload`.
- Paiement confirmé → une seule transaction : `Payment`, `Subscription` ou achat, `Entitlement`, événement.
- Paiements derrière l'interface `PaymentProvider`. En dev et preview : `FakePaymentProvider`. Jamais de clé de production hors production.
- **Abonnement** prépayé : 30 ou 365 jours, rappel J‑3, 5 jours de grâce (accès maintenu), puis expiré. Renouveler prolonge depuis l'ancienne fin.
- Offres (`Plan`, lues en base, jamais en dur) : Premium 3 500 / mois et 29 000 / an ; Pass Renforcement 3 000 / mois et 28 800 / an ; Premium + Renforcement 5 900 / mois ; achat à vie = `Course.lifetimePrice`. `Plan` porte `scopes` et `displayOn` (`home` | `reinforcement`) : les offres renforcement ne s'affichent que sur `/renforcement` et dans le tunnel qui en part ; accueil et `/premium` = Gratuit + Premium.
- Changer d'offre en cours d'abonnement (ex. Pass → Premium + Renforcement) : la nouvelle démarre à l'échéance de l'actuelle, pas de prorata en V1.
- **XP** : un `XpEvent` unique par (userId, reason, refId). Rejouer un quiz réussi ne rapporte rien.
- Jobs (cron) idempotents ; `EmailLog` empêche tout double envoi.
- Toute action admin sensible (prolonger, rembourser, accorder) écrit dans `AuditLog`.
- `Course.ownerId` toujours renseigné (l'équipe en V1) ; politiques « propriétaire ou admin ». Ne pas construire de marketplace, de rôle formateur ni de partage de revenus sans décision écrite.

## Renforcement (UNCHK)
- Référentiel saisi dans l'admin : University › Program › Semester (`hasExam`) › AcademicUnit. V1 = UNCHK › IDA, S1–S5 ; S6 (`hasExam = false`) n'est jamais proposé. Ajouter une filière = données, pas de code.
- Niveaux : L1 = S1–S2, L2 = S3–S4, L3 = S5. Le sélecteur de semestre est filtré par le niveau.
- Le semestre visible dépend de `AcademicProfile` (demandé à l'inscription via « Tu es étudiant ? », modifiable dans le profil) ; `getAccess` s'applique aux leçons de renforcement comme aux autres.
- Public sans droit : présentation, liste des UE et programme d'une UE. Tout le contenu (leçons, ressources, quiz, prépa, replays) exige le scope `reinforcement`.
- Page UE : onglets Cours · Ressources · Quiz · Prépa examen · Séances live. Prépa examen : examen blanc (`Quiz.kind = exam`), questions à préparer (`PracticeQuestion` `open`, marquées « Je sais / À revoir »), flashcards, check-list « Prêt pour l'examen » calculée côté serveur.
- Examen blanc chronométré : temps restant contrôlé côté serveur ; XP selon la règle `XpEvent`.
- Correction des examens blancs et réponses modèles envoyées au client seulement après soumission ou demande explicite.
- Lien Meet d'une `LiveSession` visible seulement avec un droit actif ; jamais dans le HTML public.
- Ne jamais stocker ni servir les PDF ou textes officiels de l'université : seulement nos propres cours, résumés et guides.
- Pages renforcement : mention « Learnity est indépendant de l'UNCHK ».

## Base de données
- Tables et colonnes en anglais, `camelCase` côté TS. Dates en UTC (affichage heure de Dakar, UTC+0).
- Toute modification de schéma = migration Drizzle versionnée. Ne jamais éditer une migration déjà appliquée.
- Suppression logique (`deletedAt`) pour cours et utilisateurs.
- Index sur toutes les clés étrangères et sur les colonnes filtrées (status, endsAt).

## Conventions de code
- Code, noms de variables et de fichiers en anglais ; textes visibles et URLs en français.
- Fichiers en `kebab-case`, composants React en `PascalCase`, fonctions en `camelCase`.
- Pas de `any`. Pas de `as` pour masquer une erreur de type. Types dérivés des schémas Zod (`z.infer`).
- Erreurs métier typées (`AccessDeniedError`, `PaymentError`…) ; messages utilisateur en français, clairs et sans jargon.
- Textes de l'interface centralisés dans des fichiers de messages, pas en dur dans les composants.
- Fonctions courtes, pas d'abstraction prématurée. Commentaires pour le « pourquoi », pas le « quoi ».

## Design (charte Learnity)
- Couleurs via tokens uniquement, jamais d'hex en dur dans les composants :
  `bg` #FAF8F3 papier · `ink` #15130F encre · `declic` #FF5B1F orange · `surligneur` #FFE36E
  `ink-muted` #6B655B · `line` #E4DFD4 · `success` #15803D · `error` #B91C1C.
- L'orange `declic` est un **aplat** : jamais pour du texte sur fond clair ; texte sur orange = encre, jamais blanc. Pour un texte orange, utiliser `declic-text` #C2410C.
- Polices : Bricolage Grotesque (titres), Inter (interface et leçons), JetBrains Mono (chiffres, compteurs, code), via `next/font`.
- Rayons : 8 badges, 12 boutons, 16 cartes, 24 blocs, pill pour filtres et barres. Grille de 4 px.
- Boutons : primaire encre ; `declic` (orange + ombre tampon) réservé aux actions d'apprentissage (lancer/valider un quiz, leçon suivante).
- Icônes Lucide, trait 2 px. Pas d'emoji dans l'interface. Pas de dégradés décoratifs.
- Accessibilité AA : contraste ≥ 4,5:1, cibles ≥ 44 px, vrais `<button>`/`<a>`, focus visible orange, états correct/incorrect jamais par la couleur seule (icône + texte).
- Mobile d'abord : chaque page doit fonctionner à 390 px. Barre mobile : Accueil · Formations · Renforcement · Mes cours · Profil.
- Navigation publique : Formations · Renforcement (badge « UNCHK ») · Blog · Premium.
- Composants récurrents de la maquette : `CodeBlock` (fond encre, langage, bouton « Copier »), `ExerciseBox` (énoncé, indice et correction dépliables), carte cours (aplat + glyphe, jamais de photo), `PricingCard`, bandeau « Mon semestre ». Le renforcement réutilise ces composants : seuls le badge UNCHK et le libellé « UE » changent.
- Les bascules « États de l'écran (démo) » de la maquette ne sont pas des fonctionnalités : en production, l'état vient du serveur.
- La maquette (canevas « Learnity — Maquette V1 ») fait référence pour les écrans ; noter tout écart dans la PR.

## Tests
- Obligatoires dans `core` : `getAccess` (dont scopes `reinforcement` et `bundle`), visibilité des offres par `displayOn`, cycle d'abonnement (actif → grâce → expiré), traitement des webhooks (signature, idempotence), attribution d'XP.
- Un test par critère d'acceptation P0 touché ; nommer le test avec l'ID (`PAY-04 rejects invalid signature`).
- E2E Playwright : inscription (avec profil académique), suivre une leçon, payer (avec `FakePaymentProvider`), Pass Renforcement → page UE → examen blanc corrigé après envoi.
- Pas de test qui appelle un vrai prestataire de paiement ou d'e-mail.

## Git et PR
- Une branche par tâche : `feat/PAY-04-webhook`, `fix/LRN-01-signed-url`.
- Commits conventionnels en anglais : `feat(billing): handle duplicate webhook (PAY-04)`.
- PR petite et relue par un humain ; CI verte obligatoire avant merge. Ne jamais pousser sur `main` directement.
- Ne jamais commiter `.env`, clés, dumps de base ou données réelles d'apprenants.

## Façon de travailler (agent)
- Lire le PRD et le module concerné avant d'écrire. Chercher un code existant avant d'en créer un nouveau.
- Pour une tâche de plus d'un fichier : proposer un plan court, puis implémenter.
- Changer seulement ce qui est demandé ; signaler les problèmes vus ailleurs sans les corriger d'office.
- En cas de doute sur une règle métier, demander plutôt que deviner.
- Mettre à jour `docs/` quand une décision change (et ce fichier si une règle change).
- Ne jamais désactiver un test, une règle lint ou la vérification de type pour « faire passer ».

## Décisions déjà prises
- Monolithe modulaire, pas de microservices.
- Abonnement prépayé sans prélèvement automatique (mobile money).
- Agrégateur de paiement choisi après le développement ; tout passe par `PaymentProvider`.
- Connexion par téléphone : pas en V1.
- Repositionnement du 20/11/2026 : Bac, BEFEM et Licence abandonnés au profit des cours tech.
- Renforcement (20/11) : offres dédiées (Pass 3 000 / mois, 28 800 / an, Premium + Renforcement 5 900 / mois), pas d'aperçu gratuit, filière IDA seule, contenus 100 % originaux, pas de démarche auprès de l'UNCHK.
- Leçons texte : blocs de code avec coloration syntaxique et bouton « Copier » (Shiki côté serveur).
