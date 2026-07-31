# FIECE — Zone Afrique
## Architecture & Spécifications de la Plateforme Web Institutionnelle Panafricaine

**Fédération Internationale des Églises Chrétiennes Évangéliques — Zone Afrique**
Document de référence technique, fonctionnelle et UX/UI — v1.0

---

## Sommaire

1. Vision produit et principes directeurs UX/UI
2. Architecture globale du système
3. Modèle de données multi-pays (Prisma / PostgreSQL)
4. Modèle de rôles et permissions (RBAC)
5. Arborescence du site et structure Next.js (App Router)
6. Module « Pays membres » et carte interactive d'Afrique
7. Plateforme communautaire / espace membres
8. Gouvernance et annuaire des responsables
9. Tableau de bord statistique international
10. Événements (internationaux / nationaux)
11. Centre de ressources bibliques
12. Actualités
13. Bibliothèque documentaire
14. Espace privé de collaboration (Intranet des responsables)
15. Dons et Mobile Money
16. Stack technique détaillée & intégrations
17. Sécurité
18. Performance, SEO, Accessibilité (WCAG 2.2)
19. DevOps, déploiement, environnements
20. Roadmap de mise en œuvre (phases)

---

## 1. Vision produit et principes directeurs UX/UI

### 1.1 Territoire de marque
Le site doit incarner simultanément : **Excellence · Crédibilité · Unité · Spiritualité · Modernité · Proximité**. Ces six mots deviennent des critères de recette pour chaque écran livré.

| Valeur | Traduction UX/UI |
|---|---|
| Excellence | Grille typographique soignée, whitespace généreux, micro-interactions discrètes, pas de gratuité visuelle |
| Crédibilité | Chiffres vérifiables (dashboard live), fiches dirigeants signées, mentions légales et gouvernance visibles dès le menu |
| Unité | Un design system unique décliné dans les 54 pays potentiels, un même gabarit de page pays, une identité FIECE dominante avant l'identité nationale |
| Spiritualité | Iconographie sobre (croix discrète, lumière, cartes africaines stylisées), citations bibliques en accroche, pas de kitsch |
| Modernité | Next.js App Router, animations fluides (Framer Motion), dark/light mode, mobile-first |
| Proximité | Carte interactive, drapeaux, langues locales, contenu localisé par pays et par église locale |

### 1.2 Direction artistique
- **Palette** : bleu profond / indigo (institution, confiance) + or/ambre (excellence, onction) + blanc cassé (clarté) ; une couleur d'accent par pays optionnelle sur sa page dédiée uniquement.
- **Typographie** : une serif éditoriale pour les titres (autorité, intemporalité) + une sans-serif géométrique pour le corps de texte (lisibilité, modernité) — ex. Fraunces/Playfair + Inter/General Sans.
- **Grille** : 12 colonnes, rayons de bordure modérés (8–12px), ombres douces, pas d'effets « glassmorphism » excessifs.
- **Photographie** : bannière plein cadre par pays, portraits dirigeants cadrés identiquement (cohérence de l'annuaire), pas de stock générique — contenu réel priorisé.
- **Ton rédactionnel** : institutionnel mais chaleureux, jamais commercial.

### 1.3 Principes d'interaction
- Mobile-first strict (majorité de l'audience africaine se connecte en mobile, réseaux parfois lents → budget de performance strict, voir §18).
- Navigation à deux niveaux : **Global (FIECE Zone Afrique)** puis **Contextuel (Pays sélectionné)** — un sélecteur de pays persistant en header une fois qu'un pays est choisi.
- Dégradation gracieuse : le site doit rester utilisable en 2G/3G (images responsive, lazy-loading, mode texte prioritaire).
- Multilingue dès la conception : FR / EN / PT (couvre l'essentiel de l'Afrique francophone, anglophone, lusophone), extensible à d'autres langues (Kiswahili, Arabe...).

---

## 2. Architecture globale du système

### 2.1 Vue d'ensemble (architecture logique)

```
                         ┌────────────────────────────┐
                         │        Utilisateurs         │
                         │  Public · Membres · Admins  │
                         └──────────────┬───────────────┘
                                        │ HTTPS
                         ┌──────────────▼───────────────┐
                         │   Next.js App Router (SSR/ISR)│
                         │   React 18 · TypeScript       │
                         │   Tailwind CSS · shadcn/ui     │
                         └───────┬───────────────┬───────┘
                                 │               │
                    ┌────────────▼───┐   ┌───────▼─────────┐
                    │  API Routes /   │   │  Route Handlers  │
                    │  Server Actions │   │  (REST interne)  │
                    └────────┬────────┘   └────────┬─────────┘
                             │                      │
                    ┌────────▼──────────────────────▼────────┐
                    │            Couche Services              │
                    │  Auth.js · RBAC · Validation (Zod)      │
                    │  Business logic (pays, membres, dons…)  │
                    └────────┬─────────────────────┬──────────┘
                             │                      │
                 ┌───────────▼─────────┐  ┌─────────▼───────────┐
                 │   Prisma ORM         │  │  Intégrations tierces│
                 │   PostgreSQL         │  │  Cloudinary, Maps,   │
                 │   (multi-schéma /    │  │  Mobile Money, FCM,  │
                 │   tenant par pays)   │  │  GA4/GTM, Zoom/Meet  │
                 └───────────────────────┘  └──────────────────────┘
```

### 2.2 Modèle multi-pays : « single-app, multi-tenant logique »
Plutôt que de dupliquer l'application par pays (coûteux à maintenir), on retient un **monolithe modulaire multi-tenant logique** :

- Une seule base de code Next.js, un seul déploiement.
- Chaque enregistrement métier (membre, église, événement, actualité, document…) porte une clé `countryId` (tenant key).
- Le **middleware Next.js** résout le pays courant via l'URL (`/pays/[countrySlug]/...`) ou via le profil de l'utilisateur connecté, et injecte le contexte pays dans chaque requête.
- Les policies RBAC + RLS PostgreSQL (Row Level Security) garantissent qu'un Administrateur National ne peut **techniquement pas** (pas seulement via l'UI) lire les données d'un autre pays.
- Ajouter un nouveau pays = **une opération de configuration**, pas de développement : création d'une entrée `Country`, upload du drapeau/couverture, activation des modules, création du premier Administrateur National. Objectif : nouveau pays en ligne en < 1 heure.

### 2.3 Pourquoi pas du multi-tenant physique (DB par pays) ?
Rejeté pour cette échelle (dizaines de pays, pas de besoin d'isolation physique réglementaire) : complexifierait les statistiques globales, les migrations, les coûts d'infra. Le tenant logique + RLS offre l'isolation nécessaire avec une seule base à faire évoluer.

---

## 3. Modèle de données multi-pays (Prisma / PostgreSQL)

Extrait du schéma Prisma (simplifié, cœur du système) :

```prisma
// ---------- ORGANISATION ----------

model Country {
  id              String   @id @default(cuid())
  name            String
  officialName    String
  slug            String   @unique          // ex: "cote-d-ivoire"
  isoCode         String   @unique           // ex: "CI"
  flagUrl         String
  coverImageUrl   String?
  status          CountryStatus @default(PLANNED) // PLANNED | ACTIVE | INACTIVE
  region          String?                    // Afrique de l'Ouest, Centrale, etc.
  languages       String[]                   // ["fr","en"]
  history         String?  @db.Text
  vision          String?  @db.Text
  address         String?
  phone           String?
  email           String?
  socialLinks     Json?                      // {facebook, youtube, instagram...}
  latitude        Float?
  longitude       Float?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  nationalLeader  User?    @relation("NationalLeader", fields: [nationalLeaderId], references: [id])
  nationalLeaderId String?

  users           User[]
  churches        LocalChurch[]
  events          Event[]
  news            NewsArticle[]
  documents       Document[]
  provinces       Province[]
  stats           CountryStats?
  galleries       MediaItem[]
}

enum CountryStatus {
  PLANNED   // affiché en "zone de développement futur" sur la carte
  ACTIVE    // représentation nationale opérationnelle
  INACTIVE
}

model Province {
  id         String  @id @default(cuid())
  name       String
  country    Country @relation(fields: [countryId], references: [id])
  countryId  String
  churches   LocalChurch[]
  leader     User?   @relation("ProvincialLeader", fields: [leaderId], references: [id])
  leaderId   String?
}

model LocalChurch {
  id          String   @id @default(cuid())
  name        String
  country     Country  @relation(fields: [countryId], references: [id])
  countryId   String
  province    Province? @relation(fields: [provinceId], references: [id])
  provinceId  String?
  address     String?
  pastor      User?    @relation("LocalPastor", fields: [pastorId], references: [id])
  pastorId    String?
  membersCount Int     @default(0)
  createdAt   DateTime @default(now())
}

// ---------- UTILISATEURS & RBAC ----------

model User {
  id              String   @id @default(cuid())
  email           String   @unique
  passwordHash    String?
  firstName       String
  lastName        String
  photoUrl        String?
  internalId      String   @unique          // matricule interne
  phone           String?
  ministry        String?                    // ministère d'appartenance
  function        String?                    // fonction
  bio             String?  @db.Text
  country         Country? @relation(fields: [countryId], references: [id])
  countryId       String?
  role            Role     @relation(fields: [roleId], references: [id])
  roleId          String
  status          UserStatus @default(ACTIVE)
  isPublicProfile Boolean  @default(false)   // fiche dirigeant visible publiquement
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  prayerRequests     PrayerRequest[]
  eventRegistrations EventRegistration[]
  donations          Donation[]
  pastoralFollowUps  PastoralFollowUp[]
  sentMessages       Message[]              @relation("Sender")
}

model Role {
  id          String   @id @default(cuid())
  key         String   @unique   // SUPER_ADMIN | INTL_ADMIN | NATIONAL_ADMIN | LOCAL_LEADER | MEMBER
  label       String
  permissions Permission[]
  users       User[]
}

model Permission {
  id     String @id @default(cuid())
  key    String @unique  // ex: "country:manage", "member:view:own_country"
  roles  Role[]
}

enum UserStatus {
  ACTIVE
  PENDING
  SUSPENDED
}

// ---------- COMMUNAUTÉ ----------

model PrayerRequest {
  id         String   @id @default(cuid())
  user       User?    @relation(fields: [userId], references: [id])
  userId     String?
  isAnonymous Boolean @default(false)
  content    String   @db.Text
  country    Country  @relation(fields: [countryId], references: [id])
  countryId  String
  visibility PrayerVisibility @default(PASTORAL_TEAM) // PUBLIC | PASTORAL_TEAM | PRIVATE
  status     PrayerStatus @default(NEW)
  createdAt  DateTime @default(now())
}

model PastoralFollowUp {
  id        String   @id @default(cuid())
  member    User     @relation(fields: [memberId], references: [id])
  memberId  String
  notes     String   @db.Text
  createdBy String
  createdAt DateTime @default(now())
}

// ---------- ÉVÉNEMENTS ----------

model Event {
  id           String   @id @default(cuid())
  title        String
  slug         String   @unique
  scope        EventScope   // INTERNATIONAL | NATIONAL
  country      Country? @relation(fields: [countryId], references: [id]) // null si international
  countryId    String?
  startAt      DateTime
  endAt        DateTime
  location     String?
  latitude     Float?
  longitude    Float?
  speakers     Json?         // [{name, bio, photoUrl}]
  program      Json?
  livestreamUrl String?
  coverImageUrl String?
  materials    Document[]
  registrations EventRegistration[]
}

enum EventScope { INTERNATIONAL NATIONAL }

model EventRegistration {
  id       String @id @default(cuid())
  event    Event  @relation(fields: [eventId], references: [id])
  eventId  String
  user     User   @relation(fields: [userId], references: [id])
  userId   String
  status   String @default("CONFIRMED")
  createdAt DateTime @default(now())
}

// ---------- RESSOURCES BIBLIQUES ----------

model Resource {
  id          String   @id @default(cuid())
  title       String
  type        ResourceType  // VIDEO | PODCAST | SERMON | BOOK | MAGAZINE | PDF | BIBLE_STUDY
  country     Country? @relation(fields: [countryId], references: [id])
  countryId   String?
  language    String
  theme       String?
  speaker     String?
  publishedAt DateTime
  fileUrl     String?
  thumbnailUrl String?
  accessLevel AccessLevel @default(PUBLIC)
}

enum ResourceType { VIDEO PODCAST SERMON BOOK MAGAZINE PDF BIBLE_STUDY }
enum AccessLevel { PUBLIC MEMBER LEADER }

// ---------- ACTUALITÉS ----------

model NewsArticle {
  id        String   @id @default(cuid())
  title     String
  slug      String   @unique
  scope     NewsScope // INTERNATIONAL | NATIONAL
  country   Country?  @relation(fields: [countryId], references: [id])
  countryId String?
  category  String?
  excerpt   String
  content   String   @db.Text
  coverImageUrl String?
  publishedAt DateTime
  authorId  String
}

enum NewsScope { INTERNATIONAL NATIONAL }

// ---------- DOCUMENTS ----------

model Document {
  id          String   @id @default(cuid())
  title       String
  category    String   // statuts, PV, rapports, formulaires...
  country     Country? @relation(fields: [countryId], references: [id])
  countryId   String?
  accessLevel AccessLevel @default(PUBLIC)
  fileUrl     String
  event       Event?   @relation(fields: [eventId], references: [id])
  eventId     String?
  createdAt   DateTime @default(now())
}

// ---------- DONS ----------

model Donation {
  id         String   @id @default(cuid())
  user       User?    @relation(fields: [userId], references: [id])
  userId     String?
  country    Country? @relation(fields: [countryId], references: [id])
  countryId  String?
  amount     Decimal
  currency   String
  provider   String    // MTN MoMo, Orange Money, Wave, Stripe, carte...
  reference  String    @unique
  status     String    // PENDING | SUCCESS | FAILED
  purpose    String?   // dîme, offrande, missions, projet social...
  createdAt  DateTime @default(now())
}

// ---------- STATISTIQUES (matérialisées) ----------

model CountryStats {
  country          Country @relation(fields: [countryId], references: [id])
  countryId        String  @id
  membersCount     Int @default(0)
  churchesCount    Int @default(0)
  pastorsCount     Int @default(0)
  provincesCount   Int @default(0)
  conversionsCount Int @default(0)
  baptismsCount    Int @default(0)
  eventsCount      Int @default(0)
  donationsTotal   Decimal @default(0)
  updatedAt        DateTime @updatedAt
}
```

**Points clés du modèle :**
- `Country.status = PLANNED` alimente directement le style « zone de développement futur » sur la carte (§6).
- `CountryStats` est une table matérialisée recalculée par job planifié (cron / queue) pour que le dashboard s'affiche instantanément, sans agrégations lourdes en temps réel à chaque visite.
- Tous les modèles à portée pays exposent `countryId` nullable pour distinguer le contenu **international** (null) du contenu **national**.

### 3.1 Row Level Security (PostgreSQL)
En complément du RBAC applicatif, activer RLS sur les tables sensibles (`User`, `PrayerRequest`, `Donation`, `Document`) avec une policy du type :

```sql
CREATE POLICY country_isolation ON "User"
USING (
  current_setting('app.role') IN ('SUPER_ADMIN','INTL_ADMIN')
  OR "countryId" = current_setting('app.country_id')::text
);
```

Cela garantit une isolation **au niveau base de données**, indépendante d'un éventuel bug applicatif.

---

## 4. Modèle de rôles et permissions (RBAC)

| Rôle | Périmètre | Exemples de droits |
|---|---|---|
| **Super Administrateur** | Système entier | Gestion des rôles, configuration technique, création de pays, accès à tous les tenants, logs de sécurité |
| **Administrateur International** | Tous les pays (lecture/écriture métier) | Publication d'actualités/événements internationaux, validation des nouveaux pays, vue globale du dashboard, gouvernance |
| **Administrateur National** | Son pays uniquement | Gestion du contenu de la page pays, des membres de son pays, des événements nationaux, des actualités nationales, validation des Responsables Locaux |
| **Responsable Local (province/église)** | Son église/sa province | Gestion des membres de son église, suivi pastoral, demandes de prière locales, calendrier local |
| **Membre** | Lui-même | Profil, historique personnel, demandes de prière, inscriptions événements, téléchargements autorisés |

Implémentation : table `Permission` en base + fichier de policies déclaratif (`lib/rbac/policies.ts`) évalué côté serveur (jamais uniquement côté client) :

```ts
// lib/rbac/policies.ts
export const policies = {
  "member:view": (actor: Session, target: { countryId: string }) =>
    actor.role === "SUPER_ADMIN" ||
    actor.role === "INTL_ADMIN" ||
    (actor.role === "NATIONAL_ADMIN" && actor.countryId === target.countryId) ||
    (actor.role === "LOCAL_LEADER" && actor.churchId === target.churchId),

  "news:publish:international": (actor: Session) =>
    ["SUPER_ADMIN", "INTL_ADMIN"].includes(actor.role),
};
```

Chaque Server Action / Route Handler commence par `assertPermission(session, "resource:action", context)` — centralisé, testable, auditable.

---

## 5. Arborescence du site et structure Next.js (App Router)

```
app/
├─ (public)/
│  ├─ page.tsx                          # Accueil international
│  ├─ a-propos/
│  │  ├─ page.tsx                       # Présentation, vision, mission, valeurs
│  │  ├─ histoire/page.tsx
│  ├─ gouvernance/
│  │  ├─ page.tsx                       # Organigramme
│  │  └─ [slug]/page.tsx                # Fiche dirigeant
│  ├─ pays/
│  │  ├─ page.tsx                       # Carte interactive Afrique
│  │  └─ [countrySlug]/
│  │     ├─ page.tsx                    # Page pays complète
│  │     ├─ actualites/page.tsx
│  │     ├─ evenements/page.tsx
│  │     ├─ eglises/page.tsx
│  │     ├─ galerie/page.tsx
│  │     └─ contact/page.tsx
│  ├─ evenements/
│  │  ├─ page.tsx                       # Internationaux
│  │  └─ [slug]/page.tsx
│  ├─ ressources/
│  │  ├─ page.tsx                       # Filtre pays/langue/thème/orateur/date
│  │  └─ [slug]/page.tsx
│  ├─ actualites/
│  │  ├─ page.tsx
│  │  └─ [slug]/page.tsx
│  ├─ documents/page.tsx
│  ├─ dons/page.tsx
│  ├─ priere/page.tsx                   # Demande de prière (public + membre)
│  └─ contact/page.tsx
│
├─ (auth)/
│  ├─ connexion/page.tsx
│  ├─ inscription/page.tsx
│  └─ mot-de-passe-oublie/page.tsx
│
├─ (member)/                            # Zone authentifiée — espace membre
│  └─ mon-espace/
│     ├─ page.tsx                       # Tableau de bord personnel
│     ├─ profil/page.tsx
│     ├─ activites/page.tsx
│     ├─ evenements/page.tsx
│     ├─ dons/page.tsx
│     └─ documents/page.tsx
│
├─ (admin)/                             # Back-office — protégé par middleware RBAC
│  └─ admin/
│     ├─ page.tsx                       # Dashboard (scope selon rôle)
│     ├─ pays/                          # SUPER_ADMIN / INTL_ADMIN
│     │  ├─ page.tsx
│     │  └─ [countryId]/page.tsx
│     ├─ membres/page.tsx               # scope filtré automatiquement
│     ├─ eglises/page.tsx
│     ├─ evenements/page.tsx
│     ├─ actualites/page.tsx
│     ├─ ressources/page.tsx
│     ├─ documents/page.tsx
│     ├─ dons/page.tsx
│     ├─ utilisateurs/page.tsx          # gestion rôles (SUPER_ADMIN)
│     └─ parametres/page.tsx
│
├─ (intranet)/                          # Espace privé responsables
│  └─ intranet/
│     ├─ messagerie/page.tsx
│     ├─ reunions/page.tsx
│     ├─ calendrier/page.tsx
│     ├─ documents/page.tsx
│     ├─ annuaire/page.tsx
│     └─ visioconference/page.tsx
│
├─ api/
│  ├─ auth/[...nextauth]/route.ts
│  ├─ webhooks/
│  │  ├─ mobile-money/route.ts
│  │  └─ cloudinary/route.ts
│  └─ v1/...                            # endpoints REST internes si besoin (app mobile future)
│
├─ middleware.ts                        # résolution pays + garde RBAC + i18n
└─ layout.tsx
```

**Middleware (`middleware.ts`)** : résout la langue (i18n), résout le pays courant depuis l'URL ou la session, vérifie la session Auth.js sur les groupes `(member)`, `(admin)`, `(intranet)`, redirige si rôle insuffisant.

Rendu :
- **SSG/ISR** pour les pages publiques peu volatiles (à propos, gouvernance) → performance et SEO maximum.
- **SSR** pour les pages pays/actualités/événements (contenu géré dynamiquement).
- **CSR** ciblé pour la carte interactive, le dashboard temps réel, la messagerie.

---

## 6. Module « Pays membres » et carte interactive d'Afrique

### 6.1 Emplacement
Entrée de menu principal **« Pays membres »** → `/pays`.

### 6.2 Composant carte
- Rendu **SVG vectoriel de l'Afrique** (topoJSON allégé, ~50kb) avec `react-simple-maps` ou D3 + Tailwind, jamais une image bitmap (accessibilité + perf + zoom net).
- Chaque pays est un `<path>` cliquable, `role="button"`, `aria-label="Côte d'Ivoire — représentation active"`.
- **États visuels distincts** :
  - Pays actif (`status = ACTIVE`) : couleur pleine (bleu institutionnel) + drapeau miniature au survol.
  - Pays planifié (`status = PLANNED`) : contour pointillé, remplissage clair/hachuré, badge « Zone de développement future », non cliquable ou lien vers une page « Devenir une représentation nationale ».
  - Pays sans donnée : gris neutre, non interactif.
- **Tooltip au survol / focus clavier** (accessible) affichant : drapeau, nombre de membres, nombre d'églises, nombre de pasteurs, nombre de provinces — alimenté par `CountryStats`.
- Clic → navigation vers `/pays/[countrySlug]`.
- Version mobile : la carte devient une **liste filtrable/recherchable** de pays (cartes avec drapeau) sous la carte simplifiée, pour rester praticable au doigt.

### 6.3 Page pays dédiée (`/pays/[countrySlug]`)
Gabarit unique réutilisable pour tous les pays (garantit l'§ « Unité ») :

1. Bannière : photo de couverture + drapeau + nom officiel.
2. Bloc présentation de la représentation nationale.
3. Historique de l'implantation (timeline).
4. Vision nationale (citation mise en avant).
5. Bloc coordonnées : adresse siège, téléphone, e-mail, réseaux sociaux (icônes cliquables).
6. Responsable national (carte profil + lien fiche gouvernance).
7. Équipe dirigeante nationale (grille de fiches).
8. Calendrier des activités (widget, filtré `countryId`).
9. Actualités nationales (3 dernières + lien « voir tout »).
10. Galerie photos (masonry, lightbox accessible).
11. Vidéos (intégration YouTube/Cloudinary, lazy-loaded).
12. Liste des églises locales affiliées (carte + liste, filtrable par province).
13. Formulaire de contact (dirigé vers l'e-mail national, anti-spam via honeypot + rate limiting).

### 6.4 Administration décentralisée
Chaque Administrateur National dispose d'un back-office `/admin` **scoping automatique sur son `countryId`** — il ne voit jamais d'écran listant d'autres pays. Le contenu (textes, médias, événements, actualités, églises) est géré exclusivement par lui, sans ticket à l'équipe internationale, ce qui répond à l'exigence « sans impacter les autres ».

---

## 7. Plateforme communautaire / espace membres

### 7.1 Inscription et connexion
- Auth.js (NextAuth) avec :
  - Email/mot de passe (hash Argon2id).
  - OAuth optionnel (Google) pour simplifier l'accès en zones à faible littératie numérique.
  - Vérification e-mail obligatoire + option SMS (Africa's Talking / Twilio) pour les pays où l'e-mail est peu utilisé.
- À l'inscription, sélection obligatoire du **pays** puis, si souhaité, de l'**église locale** → attribution automatique de `countryId`.
- Génération automatique du **numéro d'identification interne** (`internalId`), format `CI-2026-000123` (code pays + année + séquence).

### 7.2 Profil personnel
- Photo, informations personnelles, ministère d'appartenance, fonction, historique des activités (événements suivis, formations, dons — visible seulement par le membre lui-même et son responsable).
- Suivi pastoral : journal privé alimenté par le Responsable Local (`PastoralFollowUp`), jamais visible du membre directement (posture pastorale), sauf si l'organisation décide d'un mode « partagé ».

### 7.3 Fonctionnalités
- Demandes de prière (publiques anonymisables, ou réservées à l'équipe pastorale).
- Participation aux événements (inscription en 1 clic, rappel automatique via FCM/e-mail).
- Cotisations et dons (activables par pays — un `FeatureFlag` par pays permet de couper cette fonctionnalité là où elle n'est pas pertinente légalement/culturellement).
- Téléchargement de documents selon `accessLevel`.

### 7.4 Cloisonnement des données
- Un Responsable Local voit uniquement les membres de son église/sa province.
- Un Administrateur National voit uniquement les membres de son pays.
- Un Administrateur International / Super Administrateur a une vue consolidée, avec **journal d'audit** de chaque consultation cross-pays (traçabilité, conformité RGPD-like/African data protection acts).

---

## 8. Gouvernance et annuaire des responsables

Page `/gouvernance`, structurée en niveaux hiérarchiques :

```
Président international
 ├─ Vice-présidents
 ├─ Secrétaire général
 ├─ Trésorier
 ├─ Conseil exécutif
 ├─ Coordinateurs régionaux (par grande zone Afrique)
 │   └─ Responsables nationaux (par pays)
 │        └─ Responsables provinciaux
 │             └─ Responsables locaux
```

- Rendu en **organigramme interactif** (arbre pliable/dépliable) + **annuaire filtrable** (par pays, fonction, région).
- Fiche individuelle (`/gouvernance/[slug]`) : photo, biographie, responsabilités, pays, coordonnées publiques **si `isPublicProfile = true`** — champ que chaque personne (ou son admin) active volontairement, en cohérence avec la protection des données personnelles.
- Cette même donnée `User` alimente aussi l'**annuaire mondial des responsables** de l'intranet (§14), avec cette fois les coordonnées internes toujours visibles (pas de filtre `isPublicProfile`) puisque réservé aux responsables authentifiés.

---

## 9. Tableau de bord statistique international

### 9.1 Indicateurs (source : table matérialisée `CountryStats` + agrégats)
Nombre total de pays · représentations nationales · membres · pasteurs · églises affiliées · responsables · conversions · baptêmes · événements organisés · projets missionnaires · actions sociales · dons collectés.

### 9.2 Filtres
Par pays, par région (Afrique de l'Ouest, Centrale, Est, Australe, Nord), par période (mois/trimestre/année/personnalisée).

### 9.3 Implémentation
- Widgets `recharts` (barres, courbes d'évolution, carte choroplèthe réutilisant le composant de §6).
- Rafraîchissement : job planifié (Vercel Cron ou worker dédié) recalcule `CountryStats` toutes les 15–30 min ; option de rafraîchissement à la demande pour les Super Administrateurs.
- Export CSV/PDF du dashboard filtré (utile pour rapports au conseil exécutif).
- Accès : vue complète pour Super Admin / Admin International ; vue restreinte à son pays pour Admin National (mêmes composants, `countryId` forcé côté serveur).

---

## 10. Événements (internationaux / nationaux)

| | Internationaux | Nationaux |
|---|---|---|
| Visibilité | Tous les pays | Pays concerné uniquement |
| Création | Admin International / Super Admin | Admin National (son pays) |
| Page listing | `/evenements` | `/pays/[slug]/evenements` |

Chaque événement (`Event`) comprend : programme (structuré en blocs horaires), intervenants (fiches liées à `User` quand possible, sinon saisie libre), formulaire d'inscription (Server Action → `EventRegistration`, confirmation e-mail/SMS), carte Google Maps (composant chargé en lazy avec consentement cookies), diffusion en direct (embed YouTube Live / Facebook Live / lien Zoom selon disponibilité), supports téléchargeables (liés à `Document`, avec contrôle `accessLevel`).

---

## 11. Centre de ressources bibliques

`/ressources` — filtres combinables : pays, langue, thème, orateur, date, type (vidéo, podcast, prédication, livre, magazine, PDF, étude biblique).

- Recherche plein texte (PostgreSQL `tsvector` ou Meilisearch/Algolia si volume important).
- Lecteur vidéo/audio intégré (streaming via Cloudinary ou Mux), avec sous-titres si disponibles (accessibilité).
- Fiches ressources avec métadonnées structurées (Schema.org `VideoObject` / `Article` pour le SEO).
- Contrôle d'accès `PUBLIC / MEMBER / LEADER` par ressource.

---

## 12. Actualités

Deux niveaux (`NewsScope.INTERNATIONAL` / `NATIONAL`), filtrables par pays, catégorie, date. Page listing avec pagination/infinite scroll léger, fiche article en SSG/ISR pour le SEO (balisage `Article` Schema.org, Open Graph dynamique par article).

---

## 13. Bibliothèque documentaire

`/documents` (public) + section équivalente en back-office. Catégories : statuts, règlement intérieur, procès-verbaux, rapports, comptes rendus, formulaires, supports de formation, documents administratifs. Chaque document a un `accessLevel` (`PUBLIC / MEMBER / LEADER`) contrôlé côté serveur (les fichiers ne sont **jamais** servis par une simple URL Cloudinary publique pour les documents restreints — génération d'URL signée à durée limitée).

---

## 14. Espace privé de collaboration (Intranet des responsables)

Accessible uniquement aux rôles `LOCAL_LEADER` et au-dessus, sous `/intranet` :

1. **Messagerie interne** — conversations 1:1 et groupes (par pays, par fonction), stockée en base (`Message`, `Conversation`), notifications push via FCM.
2. **Réunions et convocations** — création de convocations avec liste de destinataires ciblés (par rôle/pays/région), accusé de réception, rappel automatique.
3. **Calendrier partagé** — vue consolidée des événements internationaux + réunions internes + échéances (synchronisation optionnelle Google Calendar via API).
4. **Dépôt documentaire sécurisé** — distinct de la bibliothèque publique, chiffrement au repos, journal d'accès, versionning des fichiers.
5. **Annuaire mondial des responsables** — recherche par pays/région/fonction/ministère, coordonnées internes complètes (téléphone direct, e-mail), export contact (vCard).
6. **Visioconférence** — intégration Zoom/Google Meet via leurs API (création de réunion depuis l'interface, lien injecté automatiquement dans la convocation et le calendrier) plutôt que redéveloppement d'un module WebRTC propriétaire (moins de risque, meilleure fiabilité réseau en Afrique).
7. **Notifications ciblées** — segmentation par pays, région, fonction ou ministère, envoyées via Firebase Cloud Messaging (push web/mobile) + relai e-mail pour les responsables peu connectés.

Cet espace est techniquement une **zone à part du site public** (même code base, groupe de routes `(intranet)`, mais design plus dense/outillé, moins « vitrine »).

---

## 15. Dons et Mobile Money

- Page `/dons` publique + module dans l'espace membre.
- Fournisseurs adaptés au contexte africain : **MTN Mobile Money, Orange Money, Moov Money, Wave**, complétés par carte bancaire (Stripe/Flutterwave/CinetPay comme agrégateur multi-pays — un seul contrat plutôt que N intégrations locales).
- Flux : initiation du don → redirection/USSD ou popup opérateur → webhook de confirmation (`api/webhooks/mobile-money`) → mise à jour `Donation.status` → reçu généré (PDF) → notification.
- Traçabilité complète par pays et par projet (dîme, offrande, missions, action sociale), alimentant le tableau de bord (§9).
- Conformité : jamais de stockage de données de paiement sensibles côté FIECE (délégué à l'agrégateur, conformité PCI-DSS assurée par le prestataire).

---

## 16. Stack technique détaillée & intégrations

| Domaine | Choix | Justification |
|---|---|---|
| Framework | Next.js 14+ (App Router) | SSR/ISR/SSG hybride, excellent pour SEO institutionnel |
| UI | React 18, TypeScript strict | Robustesse, maintenabilité à long terme |
| Style | Tailwind CSS + shadcn/ui (Radix) | Design system cohérent, accessible par défaut (Radix) |
| Animations | Framer Motion (usage mesuré) | Modernité sans nuire à la performance |
| ORM / DB | Prisma + PostgreSQL | Typage fort, migrations versionnées, RLS natif Postgres |
| Auth | Auth.js (NextAuth v5) | Multi-provider, JWT/session DB, compatible RBAC custom |
| Médias | Cloudinary | Transformation/optimisation d'images à la volée, essentiel pour la perf multi-pays |
| Paiement | CinetPay / Flutterwave (agrégateurs Mobile Money Afrique) | Un point d'intégration pour de multiples opérateurs locaux |
| Cartes | Google Maps Platform (pages événements/églises) + carte SVG maison (page Pays) | Maps pour géolocalisation précise, SVG pour la carte panafricaine (perf + design) |
| Analytics | Google Analytics 4 + Google Tag Manager | Pilotage marketing/institutionnel, mode consentement (RGPD-like) |
| Notifications | Firebase Cloud Messaging | Push web + mobile futur, gratuit à grande échelle |
| Recherche | PostgreSQL full-text (v1) → Meilisearch (si besoin, v2) | Simplicité puis montée en charge |
| Visio | Zoom SDK / Google Meet API | Fiabilité réseau, pas de réinvention |
| CI/CD | GitHub Actions | Tests, lint, build, migrations automatisées |
| Hébergement | Vercel (front + edge) + PostgreSQL managé (Neon/Supabase/RDS) | Scalabilité globale, edge caching proche des utilisateurs africains via CDN |
| Conteneurisation | Docker (pour les workers/cron et environnements de secours hors Vercel) | Portabilité si migration vers infra dédiée (souveraineté des données) |
| Emails/SMS | Resend/Postmark + Africa's Talking/Twilio | Fiabilité de délivrabilité en Afrique |
| i18n | next-intl | FR / EN / PT, extensible |

---

## 17. Sécurité

- **Authentification** : hash Argon2id, verrouillage après tentatives échouées, MFA optionnel (TOTP) obligatoire pour `SUPER_ADMIN` et `INTL_ADMIN`.
- **Autorisation** : RBAC applicatif (§4) **+** RLS PostgreSQL (§3.1) — défense en profondeur, aucune donnée d'un pays accessible même en cas de faille applicative.
- **Transport** : HTTPS strict (HSTS), cookies `httpOnly`, `secure`, `sameSite=strict`.
- **Entrées utilisateurs** : validation systématique via Zod côté serveur (jamais de confiance au client), échappement/sanitisation du contenu riche (DOMPurify pour tout HTML généré par un éditeur WYSIWYG).
- **Anti-abus** : rate limiting (Upstash/Redis) sur formulaires publics (contact, prière, inscription), reCAPTCHA v3/hCaptcha discret sur les formulaires sensibles.
- **Uploads** : validation du type MIME réel (pas seulement l'extension), scan antivirus (ClamAV côté worker) avant mise à disposition, URLs signées à durée limitée pour les documents restreints.
- **Journalisation & audit** : table `AuditLog` pour toute action sensible (connexion admin, export de données, changement de rôle, consultation cross-pays), conservée et consultable par le Super Administrateur.
- **Protection des données personnelles** : consentement explicite à l'inscription, page de politique de confidentialité claire, droit à l'export/suppression du profil, minimisation des données visibles publiquement (`isPublicProfile` opt-in).
- **Gestion des secrets** : variables d'environnement chiffrées (Vercel/GitHub Secrets), rotation régulière des clés API tierces.
- **Sauvegardes** : sauvegardes automatiques quotidiennes de la base, tests de restauration périodiques, plan de reprise d'activité documenté.
- **Veille** : dépendances scannées (Dependabot/Snyk), en-têtes de sécurité (CSP, X-Frame-Options, Referrer-Policy) configurés via `next.config.js`/middleware.

---

## 18. Performance, SEO, Accessibilité (WCAG 2.2)

### 18.1 Performance (objectif Lighthouse ≥ 95, chargement < 2 s)
- Rendu SSG/ISR pour les pages à faible volatilité, SSR ciblé ailleurs — pas de sur-utilisation du client-side rendering.
- Images : format AVIF/WebP automatique via Cloudinary, `next/image`, dimensions explicites (pas de CLS), lazy-loading systématique hors-fold.
- Polices : auto-hébergées via `next/font`, `font-display: swap`, sous-ensemble de caractères optimisé.
- JS : code-splitting par route (natif App Router), imports dynamiques pour les modules lourds (carte, éditeur riche, visio), suppression du JS inutilisé.
- Cache : CDN edge (Vercel), en-têtes `Cache-Control` adaptés par type de page, ISR avec revalidation ciblée (webhook déclenché à la publication de contenu).
- Budget de performance strict pour l'Afrique : test systématique en profil réseau « Fast 3G » / CPU throttlé, car une partie significative des utilisateurs n'a pas de connexion fibre.

### 18.2 SEO (objectif = 100)
- Métadonnées dynamiques (`generateMetadata`) par page/pays/article, Open Graph et Twitter Cards.
- Données structurées Schema.org : `Organization`, `Article`, `Event`, `VideoObject`, `BreadcrumbList`.
- `sitemap.xml` généré dynamiquement (inclut chaque page pays, article, événement, ressource), `robots.txt` maîtrisé.
- URLs propres et stables (`/pays/senegal`, `/actualites/[slug]`), redirections 301 gérées en cas de changement de slug.
- Maillage interne fort entre pays, gouvernance, ressources, actualités (cohérent avec le PageRank interne et l'expérience utilisateur).
- Contenu multilingue avec balises `hreflang`.

### 18.3 Accessibilité (WCAG 2.2 AA minimum)
- Contraste de couleurs validé (texte/fond ≥ 4.5:1), jamais d'information portée uniquement par la couleur (ex. carte des pays : icône + motif, pas seulement teinte).
- Navigation clavier complète, focus visible et cohérent, ordre de tabulation logique — carte interactive entièrement opérable au clavier (`tabindex`, `aria-*`).
- Composants `shadcn/ui`/Radix : accessibilité gérée nativement (dialogues, menus, listes).
- Textes alternatifs obligatoires sur toute image/média (workflow d'upload qui **impose** un champ `alt` non vide côté back-office).
- Sous-titres/transcriptions pour les vidéos de prédication (au moins pour les contenus phares).
- Formulaires : labels explicites, messages d'erreur clairs associés au champ (`aria-describedby`), pas de dépendance à la seule couleur pour signaler une erreur.
- Cible tactile ≥ 44×44px sur mobile, zoom texte jusqu'à 200 % sans perte de fonctionnalité.
- Tests : audit automatisé (axe-core en CI) + tests manuels lecteur d'écran (NVDA/VoiceOver) sur les parcours critiques (inscription, don, demande de prière).

---

## 19. DevOps, déploiement, environnements

- **Environnements** : `development` → `staging` (recette par pays pilote avant activation) → `production`.
- **CI/CD (GitHub Actions)** : lint (ESLint/Prettier), typecheck (`tsc --noEmit`), tests unitaires (Vitest) et E2E (Playwright) sur les parcours critiques (inscription, don, publication contenu par un Admin National, isolation des données), audit Lighthouse CI et axe-core en pipeline, migrations Prisma automatisées en staging puis validées manuellement en production.
- **Hébergement** : Vercel pour le front (edge network global, utile pour la latence en Afrique via ses PoP), base PostgreSQL managée avec réplication en lecture pour absorber la charge du dashboard.
- **Docker** : utilisé pour les workers asynchrones (recalcul des stats, envoi de notifications, traitement des webhooks Mobile Money) et pour permettre, si nécessaire à terme, un hébergement souverain (infrastructure cloud africaine/dédiée) sans réécriture.
- **Observabilité** : Sentry (erreurs front/back), logs structurés, alerting sur échec de paiement/webhook, tableau de bord infra (Vercel Analytics + Grafana si infra dédiée).
- **Feature flags** : activation/désactivation de modules par pays (ex. dons, cotisations) sans déploiement, via une table `FeatureFlag` simple.

---

## 20. Roadmap de mise en œuvre (phases)

**Phase 1 — Fondations (6–8 semaines)**
Design system, architecture Next.js/Prisma/RBAC, authentification, modèle `Country`/`User`/`Role`, page d'accueil, page « À propos », gouvernance (statique), 2–3 pays pilotes avec page complète, carte interactive (v1).

**Phase 2 — Communauté & contenu (6–8 semaines)**
Espace membre complet, inscriptions/connexions, demandes de prière, suivi pastoral, actualités (2 niveaux), ressources bibliques, bibliothèque documentaire, dashboard statistique v1.

**Phase 3 — Événements, dons, extension pays (6 semaines)**
Module événements complet (inscriptions, live, supports), intégration Mobile Money multi-opérateurs, généralisation du processus d'ajout de pays (self-service pour Admin International), extension à l'ensemble des pays actifs.

**Phase 4 — Intranet responsables & notifications (4–6 semaines)**
Messagerie interne, réunions/convocations, calendrier partagé, dépôt sécurisé, annuaire mondial, intégration Zoom/Meet, notifications ciblées FCM.

**Phase 5 — Optimisation, audit, mise à l'échelle**
Audit Lighthouse/SEO/Accessibilité complet, tests de charge (pic lors d'un grand événement international), durcissement sécurité (pentest externe recommandé), documentation d'exploitation, formation des Administrateurs Nationaux.

---

*Ce document constitue le cahier de spécifications de référence. Il peut être décliné en tickets techniques (épics/US) pour un backlog Jira/Linear, ou servir de base à un prototype Figma avant développement.*
