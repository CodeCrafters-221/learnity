# PRD — Learnity V1

> Product Requirements Document · version 1.4 · 20 novembre 2026 — Formations tech + Renforcement UNCHK
> Compagnons : `docs/PLAN_V1.md` (architecture, modèle de données, paiements) · charte graphique · maquette Learnity V1.
> Chaque exigence a un identifiant (`AUTH-01`…). Les commits, issues et tests y font référence.

---

## 1. Contexte et objectif

Learnity est une plateforme de **formation à la tech** en français, sur le modèle de **Believemy** et d'**OpenClassrooms** : développement web et mobile au lancement, puis IA, web design, e-commerce. L'équipe (développeurs web et mobile) crée tous les cours et encaisse seule les revenus. Public : Sénégal d'abord, puis l'Afrique francophone. La V1 remplace le site WordPress actuel par une application Next.js maintenable. L'ouverture à des formateurs externes est une option pour une V2 ou V3, non décidée.

**Deux univers** :
- **Formations** : cours tech ouverts à tous (catalogue domaine › technologie).
- **Renforcement** : accompagnement des étudiants de l'**UNCHK** (Université numérique Cheikh Hamidou Kane), organisé par filière, niveau (L1–L3), semestre et UE. Filière au lancement : Informatique, Développement d'Applications (IDA), semestres S1 à S5. L'équipe suit le programme et le rythme de chaque UE avec ses propres cours (aucune copie du cours officiel), explique avec du code, puis prépare l'examen (quiz, examen blanc, questions probables, séances live). C'est ce que l'équipe fait déjà en Google Meet avec sa promotion, et c'est le différenciateur de Learnity.

Les deux univers partagent le même moteur de cours (`Course.kind = training | reinforcement`).

**Objectif de la V1** : qu'un apprenant puisse découvrir un cours gratuitement, payer en Wave, Orange Money ou carte, suivre ses leçons, réussir ses quiz et voir sa progression, pendant que l'équipe publie et gère les cours sans développeur.

**Promesse** (proposition) : « Le déclic qui lance ta carrière tech. »

**Domaines** : Développement web et Développement mobile au lancement ; IA, Web design / UI-UX et E-commerce ensuite. Le catalogue est un arbre libre (domaine › technologie) géré dans l'admin.

## 2. Utilisateurs

| Rôle | Description | Besoin principal |
| --- | --- | --- |
| Visiteur | Débutant, étudiant, personne en reconversion ou freelance, non connecté | Comprendre l'offre, voir un aperçu de cours |
| Apprenant gratuit | Compte créé, sans droit payant | Tester 3 leçons par cours, poser des questions |
| Apprenant Premium | Abonnement mensuel ou annuel actif (ou en grâce) | Tout le catalogue de formations, quiz, PDF, XP, classement |
| Étudiant UNCHK | Apprenant avec un profil académique et un Pass Renforcement (ou Premium + Renforcement) | Comprendre ses UE du semestre et réussir ses examens |
| Acheteur à vie | A acheté un cours précis | Ce cours en entier, PDF compris, sans limite de durée |
| Admin | Membre de l'équipe Learnity | Créer les cours, gérer apprenants, abonnements, paiements, modération |
| Formateur externe (V2/V3) | Hors périmètre ; seul `Course.ownerId` est conservé pour plus tard | — |

## 3. Indicateurs de succès

À mesurer dès le lancement (PostHog + base) ; cibles à fixer par l'équipe après la bêta.

| Indicateur | Définition |
| --- | --- |
| Activation | % d'inscrits qui terminent au moins 1 leçon dans les 7 jours |
| Conversion Premium | % d'apprenants gratuits qui paient dans les 30 jours |
| Réussite paiement | % de paiements initiés qui sont confirmés |
| Renouvellement | % d'abonnements renouvelés avant la fin de la période de grâce |
| Engagement | Leçons terminées et quiz réussis par apprenant actif et par semaine |

## 4. Règles métier (source de vérité)

### 4.1 Accès au contenu

| Contenu | Visiteur | Gratuit | Achat à vie (ce cours) | Premium |
| --- | --- | --- | --- | --- |
| Fiche cours, programme, avis | Oui | Oui | Oui | Oui |
| Leçon `previewRank = 1` | Oui | Oui | Oui | Oui |
| Leçons `previewRank` 2 et 3 | Non | Oui | Oui | Oui |
| Autres leçons (texte et vidéo) | Non | Non | Oui | Oui |
| Quiz, test final et exercices corrigés | Non | Non | Oui | Oui |
| PDF et code source | Non | Non | Oui (ce cours) | Oui |
| XP, progression comptée, classement | Non | Non | Oui (ce cours) | Oui |
| Poser une question | Non | Oui | Oui | Oui |
| Renforcement : présentation, liste des UE, programme d'une UE | Oui | Oui | Oui | Oui |
| Renforcement : tous les contenus (leçons, guides, quiz, prépa examen, replays, lien Meet) | Non | Non | Non | Non : Pass Renforcement ou Premium + Renforcement requis |

- **BR-01** L'accès est calculé **uniquement côté serveur** par `getAccess()` (voir `PLAN_V1.md`). Aucun composant ne décide seul.
- **BR-02** Un droit actif = un `Entitlement` non expiré : abonnement (`all_courses` pour les formations, `reinforcement` pour le renforcement ; l'offre Premium + Renforcement accorde les deux), achat (`course`) ou geste admin. Le renforcement n'a pas d'aperçu gratuit.
- **BR-03** Les vidéos et PDF ne sont servis que par URL signée de courte durée, générée après `getAccess`.

### 4.2 Offres et prix

| Offre | Prix | Durée | Ouvre | Affichée sur |
| --- | --- | --- | --- | --- |
| Gratuit | 0 FCFA | — | Aperçus des formations | Accueil, Premium |
| Premium mensuel | 3 500 FCFA | 30 jours | Formations | Accueil, Premium |
| Premium annuel | 29 000 FCFA (−31 %) | 365 jours | Formations | Accueil, Premium |
| Pass Renforcement mensuel | 3 000 FCFA | 30 jours | Renforcement | Renforcement uniquement |
| Pass Renforcement annuel | 28 800 FCFA (36 000 − 20 %) | 365 jours | Renforcement | Renforcement uniquement |
| Premium + Renforcement | 5 900 FCFA | 30 jours | Formations + renforcement | Renforcement uniquement |
| Achat à vie | `Course.lifetimePrice`, fixé par l'admin selon le contenu et les heures | Illimitée | Ce cours | Fiche cours |

- **BR-11** Les offres renforcement (`Plan.displayOn = reinforcement`) ne s'affichent que sur `/renforcement` et dans le tunnel de paiement qui en part ; l'accueil et `/premium` n'affichent que Gratuit et Premium.
- **BR-12** Changer d'offre en cours d'abonnement : en V1, la nouvelle offre démarre à l'échéance de l'actuelle (pas de prorata) ; l'admin peut ajuster à la main.

- **BR-04** Les prix sont lus en base (`Plan`, `Course`) côté serveur ; jamais envoyés par le navigateur.
- **BR-05** Montants en entiers FCFA (`XOF`), pas de décimales.

### 4.3 Cycle de l'abonnement (prépayé, sans prélèvement automatique)

1. **Actif** à la confirmation du paiement : `endsAt = startsAt + durationDays`.
2. **J‑3** : e-mail + notification in-app avec lien de renouvellement.
3. **Échéance** : statut `grace` pendant **5 jours**, accès maintenu, bandeau, e-mail J0.
4. **J+5** : statut `expired`, retour au plan gratuit. Progression, XP et achats à vie conservés.
5. Renouveler pendant `active` ou `grace` prolonge à partir de l'ancienne date de fin.

- **BR-06** Les transitions sont faites par un job quotidien idempotent ; `EmailLog` empêche les doublons.

### 4.4 XP et classement

| Action | XP (valeurs par défaut, modifiables en base) |
| --- | --- |
| Leçon terminée | 10 |
| Quiz de chapitre réussi (1re réussite) | 50 |
| Score parfait | +20 |
| Test final réussi | 100 |
| Exercice pratique terminé | 20 |
| Cours terminé | 200 |

- **BR-07** Un gain est unique par (apprenant, raison, objet) : rejouer ne rapporte rien.
- **BR-08** XP et classement réservés aux droits payants sur le cours concerné.
- **BR-09** Classement par cours, semaine et depuis le début, recalculé toutes les 10 minutes.
- **BR-10** Un quiz est réussi à partir de `Quiz.passScore` (défaut 60 %).

## 5. Exigences fonctionnelles

Priorités : **P0** indispensable au lancement · **P1** dans la V1 si le planning tient · **P2** plus tard.
Écran de référence = artboard de la maquette.

### 5.1 Authentification et compte

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| AUTH-01 | Inscription e-mail + mot de passe (8 caractères min.), nom, objectif (découvrir, reconversion, études, freelance), acceptation CGU | P0 | Inscription |
| AUTH-02 | Vérification de l'e-mail par lien ; accès aux aperçus possible avant vérification, paiement seulement après | P0 | MotDePasse (vérifier) |
| AUTH-03 | Connexion e-mail / mot de passe et Google | P0 | Inscription |
| AUTH-04 | Mot de passe oublié : lien valable 1 h, message identique que le compte existe ou non | P0 | MotDePasse |
| AUTH-05 | Profil : nom, photo, objectif, niveau, technologies qui l'intéressent | P1 | Profil |
| AUTH-06 | Préférences de notifications | P1 | Profil |
| AUTH-07 | Changer le mot de passe, exporter ses données, supprimer son compte | P1 | Profil |
| AUTH-08 | Connexion par téléphone (OTP SMS) | P2 | — |

**Critères d'acceptation AUTH-01** : un e-mail déjà utilisé affiche une erreur claire ; le mot de passe est haché ; un e-mail de vérification part dans la minute ; l'apprenant arrive sur son tableau de bord.

### 5.2 Catalogue

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| CAT-01 | Arbre de catégories : domaine › technologie (ex. Développement web › React), libre et géré dans l'admin | P0 | Catalogue |
| CAT-02 | Liste des cours publiés avec filtres domaine, technologie, niveau (débutant, intermédiaire, avancé), format, aperçu gratuit | P0 | Catalogue |
| CAT-03 | Recherche plein texte (titre, matière) | P1 | Catalogue |
| CAT-04 | Fiche cours : objectifs, prérequis, technologies, niveau, programme (chapitres › leçons avec badge Aperçu/Premium), durée, format, avis, 3 options d'accès | P0 | Cours |
| CAT-05 | URLs SEO : `/cours/[categorie]/[sous-categorie]`, `/cours/[slug]` | P0 | — |

**Critères CAT-04** : seules les leçons accessibles sont cliquables ; les boutons mènent à la leçon 1, au paiement Premium ou à l'achat à vie ; données structurées `Course` présentes.

### 5.3 Apprentissage

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| LRN-01 | Lecteur de leçon vidéo (HLS adaptatif, qualité basse par défaut sur réseau lent) | P0 | Lecon |
| LRN-02 | Lecteur de leçon texte façon OpenClassrooms : titres, blocs de code colorés avec bouton « Copier », encadrés Astuce / Attention, captures, surligneur | P0 | Lecon |
| LRN-03 | Programme du cours en barre latérale avec états terminé / en cours / verrouillé | P0 | Lecon |
| LRN-04 | Reprise de la vidéo à la dernière position ; reprise du cours à la dernière leçon | P0 | TableauDeBord |
| LRN-05 | « Marquer comme terminée » + passage à la leçon suivante | P0 | Lecon |
| LRN-06 | Onglet Ressources : PDF et code source téléchargeables si droit payant, sinon incitation Premium | P0 | Lecon |
| LRN-10 | Exercices pratiques : énoncé, indice, correction à dévoiler, « marquer comme fait » | P1 | Lecon |
| LRN-07 | Questions sur une leçon, réponse de l'équipe notifiée | P1 | Lecon |
| LRN-08 | Avis 1–5 étoiles + commentaire après 30 % de progression, un par cours, modéré | P1 | Cours |
| LRN-09 | Favoris | P1 | MesCours |

**Critères LRN-01** : sans droit, l'API ne renvoie jamais l'URL vidéo ; l'URL signée expire en quelques minutes ; le lecteur fonctionne sur mobile Android d'entrée de gamme.

### 5.4 Quiz

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| QZ-01 | Quiz de fin de chapitre : choix unique, multiple, vrai/faux ; énoncés pouvant contenir du code | P0 | Quiz |
| QZ-02 | Validation question par question avec correction expliquée | P0 | Quiz |
| QZ-03 | Score final, statut réussi/échoué, XP gagnés, nouvelle tentative | P0 | Quiz |
| QZ-04 | Test final du cours, chronométré, correction à la fin | P1 | Quiz |
| QZ-05 | Historique des tentatives et scores | P0 | TableauDeBord |
| QZ-06 | Navigation clavier (A–D, Entrée) et états jamais signalés par la couleur seule | P0 | Quiz |

**Critères QZ-02** : la bonne réponse et l'explication ne sont envoyées au client qu'après validation (pas dans le HTML initial).

### 5.5 Espace apprenant et gamification

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| STU-01 | Tableau de bord : reprendre, statistiques, bandeau abonnement (actif / J‑3 / grâce / gratuit) | P0 | TableauDeBord |
| STU-02 | Mes cours : en cours, terminés, achetés, favoris | P0 | MesCours |
| STU-03 | XP par événement (BR-07) et total affiché | P1 | TableauDeBord |
| STU-04 | Classement par cours, semaine / global (BR-09) | P1 | Classement |
| STU-05 | Page abonnement : statut, dates, renouvellement, achats à vie, historique et reçus PDF | P0 | Abonnement |
| STU-06 | Boutique d'articles contre XP | P2 | — |
| STU-07 | Certificat de réussite d'un cours (PDF vérifiable) | P2 | — |
| STU-08 | Parcours métier : suite ordonnée de cours (ex. « Développeur front-end ») | P2 | — |

### 5.6 Paiement et abonnements

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| PAY-01 | Page Premium : offres Gratuit et Premium (mensuel, annuel), comparatif, FAQ | P0 | Premium |
| PAY-11 | Section tarifs de la page Renforcement : 3 offres (Pass mensuel, Pass annuel, Premium + Renforcement), sans offre gratuite (BR-11) | P0 | Renforcement |
| PAY-02 | Tunnel : choix de l'offre, choix Wave / Orange Money / carte, redirection vers le prestataire | P0 | Paiement |
| PAY-03 | Écran « Confirmation en cours » qui attend le statut serveur ; succès ou échec avec relance | P0 | Paiement |
| PAY-04 | Webhook signé, traitement idempotent, octroi des droits dans une transaction | P0 | — |
| PAY-05 | Job de rapprochement des paiements en attente (toutes les 15 min) | P0 | — |
| PAY-06 | Achat à vie d'un cours | P0 | Cours, Paiement |
| PAY-07 | Cycle d'abonnement J‑3 / grâce / expiration (BR-06) | P0 | — |
| PAY-08 | Reçu PDF par paiement réussi | P1 | Abonnement |
| PAY-09 | `FakePaymentProvider` en développement et preview ; agrégateur réel branché en fin de sprint 4 | P0 | — |
| PAY-10 | Codes promo | P2 | — |

**Critères PAY-04** : un même webhook reçu deux fois ne crée qu'un droit ; une signature invalide est rejetée (401) et journalisée ; le payload brut est stocké.

### 5.7 Notifications

| ID | Déclencheur | Canal | Priorité |
| --- | --- | --- | --- |
| NTF-01 | Bienvenue + vérification e-mail | E-mail | P0 |
| NTF-02 | Paiement confirmé (reçu) / échoué | E-mail | P0 |
| NTF-03 | Abonnement J‑3, échéance, fin de grâce | E-mail + in-app | P0 |
| NTF-04 | Réponse à une question | E-mail + in-app | P1 |
| NTF-05 | Conseil de la semaine (opt-in) | E-mail | P2 |

### 5.8 Back-office admin

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| ADM-01 | Accès réservé au rôle `admin` (layout + chaque action serveur) | P0 | — |
| ADM-02 | Indicateurs : inscrits, Premium actifs (dont grâce), revenus, expirations à J‑3, par période | P0 | Admin |
| ADM-03 | CRUD cours : infos, catégorie, format, prix à vie, statut brouillon/publié, liste de vérification avant publication | P0 | AdminCours |
| ADM-04 | Éditeur de structure : chapitres et leçons réordonnables, type, `previewRank`, XP | P0 | AdminCours |
| ADM-05 | Upload vidéo (encodage chez l'hébergeur) et PDF | P0 | AdminCours |
| ADM-06 | Éditeur de leçon texte (Tiptap) avec blocs de code par langage, encadrés, images | P0 | AdminCours |
| ADM-07 | Banque de questions et quiz (seuil, durée, XP) | P0 | AdminGestion |
| ADM-08 | Arbre des domaines et technologies, tags | P0 | AdminGestion |
| ADM-09 | Apprenants : recherche, statut d'accès, prolonger un abonnement, accorder un cours | P0 | AdminGestion |
| ADM-10 | Paiements : filtres par statut, revérifier, rembourser | P0 | AdminGestion |
| ADM-11 | Modération des avis et questions | P1 | AdminGestion |
| ADM-12 | Journal d'audit de toute action admin sensible | P0 | — |

### 5.9 Site public

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| WEB-01 | Accueil : promesse, domaines, méthode, cours à la une, démo quiz, tarifs, CTA | P0 | Main |
| WEB-02 | Blog de tutoriels tech : liste filtrable, article avec code et cours associé | P1 | Blog, Article |
| WEB-03 | À propos, contact (formulaire), CGU, confidentialité | P0 | PagesInfo |
| WEB-04 | Version mobile de toutes les pages (390 px) | P0 | Mobile* |

### 5.10 Renforcement universitaire (UNCHK)

| ID | Exigence | Priorité | Écran |
| --- | --- | --- | --- |
| REN-01 | Profil académique : université (UNCHK par défaut, liste extensible), filière, niveau (L1, L2, L3), semestre (L1 = S1–S2, L2 = S3–S4, L3 = S5 ; le S6, projet noté sans examen, n'est pas proposé en V1) ; demandé à l'inscription ou au premier accès, modifiable | P0 | Renforcement, Profil |
| REN-02 | Rappel de mise à jour du semestre à chaque changement de semestre (dates configurées par l'admin) | P1 | TableauDeBord |
| REN-03 | Page « Mon semestre » : UE du semestre de l'étudiant, avec progression et contenus disponibles | P0 | MonSemestre |
| REN-04 | Changer de semestre ou consulter un autre semestre sans modifier son profil | P1 | MonSemestre |
| REN-05 | Page UE avec onglets Cours, Ressources, Quiz, Prépa examen, Séances live | P0 | UE |
| REN-06 | Contenus par UE : résumé (texte + PDF), guide complet (PDF), explications avec code, quiz de chapitre | P0 | UE |
| REN-07 | Examen blanc chronométré par UE, corrigé à la fin, avec score et points faibles par chapitre | P0 | PrepaExamen |
| REN-08 | Questions à préparer : question probable + réponse modèle à dévoiler, marquées « je sais / à revoir » | P0 | PrepaExamen |
| REN-09 | Flashcards de notions clés, révision par cartes | P1 | PrepaExamen |
| REN-10 | Check-list « prêt pour l'examen » par UE (chapitres terminés, quiz réussis, examen blanc ≥ seuil) | P1 | PrepaExamen |
| REN-11 | Séances live : planning des séances Google Meet par UE, lien visible avec un droit actif, replay ensuite, rappel e-mail la veille | P1 | UE |
| REN-12 | Tableau de bord : bloc « Mon semestre » (UE en cours, prochain examen blanc, prochaine séance live) | P0 | TableauDeBord |
| REN-13 | Admin : référentiel universités, filières, semestres, UE (avec année académique), saisi par l'équipe ; au lancement Informatique, Développement d'Applications (IDA) S1–S5 | P0 | AdminRenforcement |
| REN-14 | Admin : créer le contenu de renforcement d'une UE avec l'éditeur de cours existant, ajouter questions d'examen et flashcards | P0 | AdminRenforcement |
| REN-15 | Admin : planifier les séances live (titre, date, lien Meet), ajouter le replay | P1 | AdminRenforcement |
| REN-17 | Sans droit `reinforcement`, toute leçon ou ressource de renforcement mène à la section tarifs de `/renforcement` | P0 | UE |
| REN-16 | Mention « Learnity est indépendant de l'UNCHK » sur les pages renforcement ; contenus 100 % originaux, aucun PDF ni texte officiel republié | P0 | Renforcement |

**Critères REN-03** : un étudiant L3 · S5 ne voit que les UE de S5 de sa filière ; une UE sans contenu publié n'apparaît pas (ou apparaît « Bientôt » si l'admin l'a annoncée).

**Critères REN-07** : la correction et le score ne sont envoyés qu'à la soumission ; le temps restant est contrôlé côté serveur ; les XP d'un examen blanc suivent BR-07.

## 6. Exigences non fonctionnelles

| ID | Exigence |
| --- | --- |
| NFR-01 | Performance : LCP < 2,5 s en 4G sur mobile milieu de gamme ; < 200 Ko de JS sur l'accueil |
| NFR-02 | Réseau : vidéo adaptative, images optimisées, pages catalogue statiques revalidées |
| NFR-03 | Accessibilité WCAG 2.1 AA (contrastes de la charte, clavier, libellés, sous-titres) |
| NFR-04 | Sécurité : sessions httpOnly, rate limit connexion et webhooks, CSP, secrets serveur uniquement, URLs signées |
| NFR-05 | Données personnelles : consentement, export, suppression ; conformité loi sénégalaise 2008-12 (CDP) |
| NFR-06 | Fiabilité : sauvegarde quotidienne, alertes Sentry, alerte paiement en attente > 1 h |
| NFR-07 | SEO : métadonnées, sitemap, données structurées, URLs françaises |
| NFR-08 | Langue : interface en français, textes centralisés pour ajouter d'autres langues |
| NFR-09 | Qualité : TypeScript strict, CI verte obligatoire, tests sur `core` (accès, abonnements, XP, webhooks) |

## 7. Hors périmètre V1

Formateurs externes et partage de revenus (V2/V3, non décidé), application mobile native, boutique XP, connexion par téléphone, codes promo, forum public, certificats et parcours métier (P2).

## 8. Jalons

| Sprint | Semaines | Livrable | Exigences principales |
| --- | --- | --- | --- |
| 0 | S1–2 | Fondations : monorepo, CI, design system, schéma, auth | AUTH-01..04, NFR-09 |
| 1 | S3–4 | Catalogue + admin cours | CAT-*, ADM-01..08 |
| 2 | S5–6 | Apprentissage | LRN-*, STU-01..02 |
| 3 | S7–8 | Quiz + XP | QZ-*, STU-03..04 |
| 4 | S9–10 | Renforcement UNCHK | REN-* |
| 5 | S11–12 | Paiement (simulé puis réel) + abonnements | PAY-*, NTF-01..03, STU-05, ADM-09..10 |
| ◆ | — | Validation : un vrai paiement Wave, OM et carte de bout en bout | PAY-02..05 |
| 6 | S13–14 | Finitions, blog, SEO, bêta fermée (avec la promotion UNCHK) | WEB-*, LRN-07..09, ADM-11 |
| ◆ | S15 | Lancement public | — |

**Définition de « terminé »** pour toute exigence : code relu, tests passent, critères d'acceptation vérifiés, écran conforme à la maquette en desktop et mobile, pas de régression d'accessibilité.

## 9. Questions ouvertes

- Liste des UE IDA de S1 à S5 (saisie par l'équipe dans l'admin).
- Promesse définitive (proposition : « Le déclic qui lance ta carrière tech. »).
- Liste des premiers cours web et mobile au lancement.
- Choix de l'agrégateur de paiement (après développement).
- Base et auth : Supabase ou Postgres + Better Auth (sprint 0).
- Hébergeur vidéo : Bunny Stream ou Mux (test 3G avant sprint 1).
- Grille de prix des achats à vie.
- Textes juridiques (CGU, confidentialité) à faire relire.
