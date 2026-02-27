# Dossier de Conception Technique
## Application Tatouage Thérapeutique

**Version :** 1.0  
**Date :** 27 février 2026  
**Auteurs :** MaxOuvrard, Cyrien, Dylan Bathig  
**Statut :** Draft  
**Référence DAT :** v4.0  

---

## Table des matières

1. [Architecture globale](#1-architecture-globale)
2. [Modèle de données](#2-modèle-de-données)
3. [Arborescence des applications](#3-arborescence-des-applications)
4. [Parcours utilisateurs & diagrammes de séquence](#4-parcours-utilisateurs--diagrammes-de-séquence)
5. [Diagrammes d'états](#5-diagrammes-détats)
6. [Architecture des modules Laravel](#6-architecture-des-modules-laravel)
7. [Flux de paiement](#7-flux-de-paiement)
8. [Flux médias & protection des photos](#8-flux-médias--protection-des-photos)
9. [Flux temps réel — Messagerie](#9-flux-temps-réel--messagerie)
10. [Sécurité & RGPD](#10-sécurité--rgpd)
11. [Infrastructure & déploiement](#11-infrastructure--déploiement)

---

## 1. Architecture globale

### 1.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                    │
│                                                                     │
│   ┌───────────────────────┐       ┌───────────────────────────┐     │
│   │     App Mobile        │       │        Web App            │     │
│   │  React Native + Expo  │       │   Next.js 15 + Vercel     │     │
│   │   iOS  │  Android     │       │  Landing Page  │  Shop    │     │
│   └──────────┬────────────┘       └───────────┬───────────────┘     │
└──────────────┼─────────────────────────────────┼───────────────────┘
               │  HTTPS / REST                   │  HTTPS / REST
               │  WSS (WebSocket)                │
               └──────────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │        Laravel 11          │
                    │     Octane (Swoole)        │
                    │   Monolithe Modulaire      │
                    │                            │
                    │  ┌─────────┐ ┌──────────┐  │
                    │  │  API    │ │  Reverb  │  │
                    │  │  REST   │ │ WebSocket│  │
                    │  └────┬────┘ └────┬─────┘  │
                    │       │           │        │
                    │  ┌────▼───────────▼──────┐ │
                    │  │   Horizon (Queues)    │ │
                    │  │  Jobs / Notifications │ │
                    │  └───────────────────────┘ │
                    └──────────────┬─────────────┘
                                   │
             ┌─────────────────────┼────────────────────┐
             │                     │                    │
    ┌────────▼────────┐   ┌────────▼────────┐  ┌────────▼────────┐
    │    Supabase     │   │  Upstash Redis  │  │   Cloudinary    │
    │  PostgreSQL     │   │  Cache/Sessions │  │  Médias + DRM   │
    │  + PostGIS      │   │  Queues/Pub-Sub │  │  Watermarking   │
    └─────────────────┘   └─────────────────┘  └─────────────────┘
             │
    ┌────────▼────────┐
    │     Stripe      │
    │ Cashier/Connect │
    └─────────────────┘
```

### 1.2 Principes d'architecture

| Principe | Décision |
|---|---|
| **Pattern** | Monolithe modulaire (1 app, N modules métier) |
| **Communication interne** | Appels directs entre modules Laravel (pas d'HTTP interne) |
| **Communication externe** | REST JSON pour mobile + web, WSS pour temps réel |
| **Stateless** | API sans état, sessions stockées dans Redis |
| **Scalabilité horizontale** | Plusieurs instances Laravel derrière un load balancer |
| **Queue-first** | Tout traitement lourd (emails, push, watermark) passe par une queue |

---

## 2. Modèle de données

### 2.1 Diagramme Entité-Relation (ERD)

```
┌──────────────────┐         ┌──────────────────────┐
│      USERS       │         │       ARTISTS        │
├──────────────────┤         ├──────────────────────┤
│ id (UUID) PK     │◄────────│ id (UUID) PK         │
│ email            │  1    1 │ user_id FK           │
│ password_hash    │         │ studio_name          │
│ oauth_provider   │         │ bio                  │
│ oauth_id         │         │ is_certified         │
│ first_name       │         │ certification_date   │
│ last_name        │         │ specialties (JSON)   │
│ profile_pic_url  │         │ location (POINT)     │
│ skin_conditions* │         │ address              │
│ location (POINT) │         │ subscription_tier    │
│ created_at       │         │ rating_avg           │
│ updated_at       │         │ reviews_count        │
└────────┬─────────┘         └──────────┬───────────┘
         │                              │
         │ 1                            │ 1
         │                              │
         │ N                            │ N
┌────────▼─────────┐         ┌──────────▼───────────┐
│     BOOKINGS     │         │   PORTFOLIO_ITEMS    │
├──────────────────┤         ├──────────────────────┤
│ id (UUID) PK     │         │ id (UUID) PK         │
│ user_id FK       │         │ artist_id FK         │
│ artist_id FK     │         │ image_url            │
│ status           │         │ thumbnail_url        │
│ appointment_date │         │ category             │
│ duration_minutes │         │ is_before_after      │
│ price_estimate   │         │ created_at           │
│ deposit_amount   │         └──────────────────────┘
│ deposit_paid_at  │
│ final_paid_at    │         ┌──────────────────────┐
│ stripe_pi_id     │         │  ARTIST_SUBSCRIPTIONS│
│ notes            │         ├──────────────────────┤
│ created_at       │         │ id (UUID) PK         │
└────────┬─────────┘         │ artist_id FK         │
         │                   │ stripe_sub_id        │
         │ 1                 │ plan                 │
         │                   │ status               │
         │ 1                 │ current_period_end   │
┌────────▼─────────┐         └──────────────────────┘
│     REVIEWS      │
├──────────────────┤         ┌──────────────────────┐
│ id (UUID) PK     │         │    CONVERSATIONS     │
│ user_id FK       │         ├──────────────────────┤
│ artist_id FK     │         │ id (UUID) PK         │
│ booking_id FK    │         │ user_id FK           │
│ rating (1-5)     │         │ artist_id FK         │
│ comment          │         │ created_at           │
│ photos_urls JSON │         └──────────┬───────────┘
│ created_at       │                    │ 1
└──────────────────┘                    │
                                        │ N
                             ┌──────────▼───────────┐
                             │      MESSAGES        │
                             ├──────────────────────┤
                             │ id (UUID) PK         │
                             │ conversation_id FK   │
                             │ sender_id FK         │
                             │ content              │
                             │ media_url            │
                             │ read_at              │
                             │ created_at           │
                             └──────────────────────┘

┌──────────────────┐         ┌──────────────────────┐
│     PRODUCTS     │         │       ORDERS         │
├──────────────────┤         ├──────────────────────┤
│ id (UUID) PK     │◄────────│ id (UUID) PK         │
│ name             │  N    N │ user_id FK           │
│ description      │         │ stripe_pi_id         │
│ price            │         │ status               │
│ category         │         │ total_amount         │
│ recommended_by   │         │ created_at           │
│ images_urls JSON │         └──────────┬───────────┘
│ stock_quantity   │                    │
│ is_active        │         ┌──────────▼───────────┐
└──────────────────┘         │    ORDER_ITEMS       │
                             ├──────────────────────┤
                             │ id (UUID) PK         │
                             │ order_id FK          │
                             │ product_id FK        │
                             │ quantity             │
                             │ unit_price           │
                             └──────────────────────┘

* skin_conditions : chiffré au repos (AES-256 via pgcrypto)
```

### 2.2 Index critiques

| Table | Index | Type | Raison |
|---|---|---|---|
| artists | location | GIST (PostGIS) | Recherche géospatiale |
| users | location | GIST (PostGIS) | Géolocalisation utilisateur |
| artists | studio_name + bio | GIN (full-text) | Recherche textuelle |
| bookings | user_id | B-Tree | Requêtes fréquentes |
| bookings | artist_id | B-Tree | Requêtes fréquentes |
| messages | conversation_id + created_at | B-Tree | Pagination messagerie |
| portfolio_items | artist_id | B-Tree | Galerie tatoueur |

### 2.3 Données sensibles — Stratégie de chiffrement

| Champ | Table | Traitement |
|---|---|---|
| `skin_conditions` | users | Chiffré AES-256 (pgcrypto), déchiffré uniquement par l'utilisateur lui-même et le tatoueur du RDV |
| `notes` | bookings | Chiffré AES-256 si contenu médical |
| `content` | messages | Chiffrement TLS en transit, stockage chiffré au repos |
| `password_hash` | users | bcrypt (cost 12) |

---

## 3. Arborescence des applications

### 3.1 App Mobile (React Native + Expo Router)

```
APP MOBILE
│
├── (AUTH)
│   ├── /login                  Connexion email / OAuth
│   ├── /register               Inscription
│   └── /onboarding             Profil initial (type de besoin, localisation)
│
├── (TABS) — Barre de navigation principale
│   ├── / (index)               FYP — Fil de découverte
│   ├── /map                    Carte des tatoueurs
│   ├── /shop                   Boutique produits
│   ├── /messages               Liste des conversations
│   └── /profile                Profil utilisateur
│
├── /artist
│   ├── /[id]                   Profil tatoueur
│   ├── /[id]/portfolio         Galerie de réalisations
│   ├── /[id]/reviews           Avis clients
│   └── /[id]/book              Prise de rendez-vous
│
├── /booking
│   ├── /[id]                   Détail d'un RDV
│   ├── /[id]/deposit           Paiement de l'acompte
│   └── /[id]/payment           Paiement du solde
│
├── /conversation
│   └── /[id]                   Fil de messagerie
│
├── /ar
│   └── /preview                Visualisation AR du tatouage
│
├── /shop
│   ├── /[category]             Produits par catégorie
│   └── /product/[id]           Fiche produit
│
└── /settings
    ├── /account                Paramètres du compte
    ├── /rgpd                   Consentements & données
    └── /subscription           Abonnement tatoueur (si rôle tatoueur)
```

### 3.2 Web (Next.js App Router)

```
SITE WEB
│
├── / (Landing Page)            Hero, valeur, comment ça marche,
│                               témoignages, FAQ, newsletter
│
├── /shop
│   ├── /                       Catalogue produits
│   ├── /[category]             Filtres par catégorie
│   ├── /product/[slug]         Fiche produit détaillée
│   ├── /cart                   Panier
│   └── /checkout               Tunnel de paiement Stripe
│
├── /auth
│   ├── /login                  Connexion
│   └── /register               Inscription
│
└── /account
    └── /orders                 Historique commandes
```

### 3.3 Rôles & accès

| Rôle | App Mobile | Web |
|---|---|---|
| **Visiteur** | FYP, profils tatoueurs, map (lecture) | Landing, shop (consultation) |
| **Client** | FYP, map, RDV, paiement, messagerie, avis | Shop (achat), compte |
| **Tatoueur** | Tout client + gestion portfolio, planning, abonnement | Tableau de bord (V2) |
| **Admin** | Backoffice (Filament Laravel) | N/A |

---

## 4. Parcours utilisateurs & diagrammes de séquence

### 4.1 Inscription & Onboarding

```
Utilisateur          App Mobile            API Laravel          Supabase
    │                    │                      │                   │
    │── Ouvre l'app ────►│                      │                   │
    │                    │── GET /auth/check ──►│                   │
    │                    │◄── 401 Unauthenticated│                  │
    │                    │                      │                   │
    │── Choisit Google ─►│                      │                   │
    │                    │── OAuth Google ─────────────────────────►│
    │                    │◄── Token Google ─────────────────────────│
    │                    │─ POST /auth/google ─►│                  │
    │                    │                      │── Upsert user ───►│
    │                    │                      │◄── User créé ─────│
    │                    │◄── Sanctum Token ────│                   │
    │                    │                      │                   │
    │── Onboarding ─────►│                      │                   │
    │  (type de besoin,  │                      │                   │
    │   localisation)    │── PATCH /users/me ──►│                   │
    │                    │                      │── Update user ───►│
    │                    │◄── 200 OK ───────────│                   │
    │◄── Home (FYP) ─────│                      │                   │
```

### 4.2 Recherche & sélection d'un tatoueur

```
Utilisateur          App Mobile            API Laravel          Supabase (PostGIS)
    │                    │                      │                   │
    │── Ouvre Map ──────►│                      │                   │
    │                    │── GET /artists       │                   │
    │                    │   ?lat=&lng=         │                   │
    │                    │   &radius=50         │                   │
    │                    │   &specialty=        │                   │
    │                    │   &certified=true ──►│                   │
    │                    │                      │── ST_DWithin() ──►│
    │                    │                      │◄── Liste triée ───│
    │                    │◄── Artists[] ────────│                   │
    │                    │                      │                   │
    │── Clique profil ──►│                      │                   │
    │                    │── GET /artists/:id ─►│                   │
    │                    │── GET /artists/:id   │                   │
    │                    │        /portfolio ──►│                   │
    │                    │── GET /artists/:id   │                   │
    │                    │        /reviews ────►│                   │
    │◄── Profil complet ─│                      │                   │
```

### 4.3 Prise de rendez-vous & paiement de l'acompte

```
Utilisateur       App Mobile        API Laravel        Stripe          Supabase
    │                 │                  │                │                │
    │── Choisit ─────►│                  │                │                │
    │   date/heure   │                  │                │                │
    │                 │── POST /bookings►│                │                │
    │                 │                  │── Insert ──────────────────────►│
    │                 │◄── Booking créé ─│                │                │
    │                 │   (status:pending)                │                │
    │                 │                  │                │                │
    │── Payer ────────►│                 │                │                │
    │   acompte       │── POST /bookings │                │                │
    ��                 │   /:id/deposit ─►│                │                │
    │                 │                  │── Create ──────►│               │
    │                 │                  │   PaymentIntent │               │
    │                 │◄── client_secret ◄────────────────│               │
    │                 │                  │                │                │
    │── Confirme ─────►│                 │                │                │
    │   paiement      │── Stripe SDK ────────────────────►│               │
    │                 │                  │                │                │
    │                 │         Webhook Stripe            │                │
    │                 │                  │◄── payment_    │                │
    │                 │                  │    intent.     │                │
    │                 │                  │    succeeded ──│                │
    │                 │                  │── Update ──────────────────────►│
    │                 │                  │   booking      │                │
    │                 │                  │   (confirmed)  │                │
    │                 │                  │── Queue: ───────────────────────│
    │                 │                  │   Email + Push │                │
    │◄── Confirmation─│                  │                │                │
```

### 4.4 Messagerie entre client et tatoueur

```
Client            App Mobile         Laravel (Reverb)      Tatoueur App
  │                   │                    │                    │
  │── Envoie msg ────►│                    │                    │
  │                   │── POST /conversations                   │
  │                   │   /:id/messages ──►│                    │
  │                   │                    │── Persist message  │
  │                   │                    │── Broadcast        │
  │                   │                    │   channel privé ──►│
  │                   │◄── 201 Created ────│                    │
  │◄── Msg affiché ───│                    │◄── Echo reçu ──────│
  │                   │                    │                    │
  │                   │                    │── Queue: Push notif►│
  │                   │                    │   (si hors app)    │
```

### 4.5 Achat boutique (Web)

```
Utilisateur       Next.js (Web)       API Laravel          Stripe
    │                  │                   │                  │
    │── Ajoute ────────►│                  │                  │
    │   au panier       │ (localStorage)   │                  │
    │                   │                  │                  │
    │── Checkout ──────►│                  │                  │
    │                   │── POST /orders ─►│                  │
    │                   │                  │── Create ────────►│
    │                   │                  │   PaymentIntent  │
    │                   │◄── client_secret ◄──────────────────│
    │                   │                  │                  │
    │── Paye ──────────►│── Stripe.js ───────────────────────►│
    │                   │                  │                  │
    │                   │       Webhook Stripe                │
    │                   │                  │◄── succeeded ────│
    │                   │                  │── Update order   │
    │                   │                  │── Queue: Email   │
    │◄── Confirmation ──│                  │   confirmation   │
```

---

## 5. Diagrammes d'états

### 5.1 États d'un Booking

```
                    ┌─────────────┐
                    │   PENDING   │◄──── Créé par l'utilisateur
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │ Acompte payé            │ Annulé avant paiement
              ▼                         ▼
       ┌─────────────┐           ┌─────────────┐
       │  CONFIRMED  │           │  CANCELLED  │
       └──────┬──────┘           └─────────────┘
              │
   ┌──────────┴──────────┐
   │ RDV passé            │ Annulé après confirmation
   ▼                      ▼
┌──────────────┐    ┌─────────────┐
│  AWAITING_   │    │  CANCELLED  │
│  PAYMENT     │    │ (remboursé) │
└──────┬───────┘    └─────────────┘
       │
       │ Solde payé
       ▼
┌─────────────┐
│  COMPLETED  │──── Déclenche demande d'avis
└─────────────┘
```

### 5.2 États d'un Tatoueur

```
┌──────────────┐
│  REGISTERED  │◄──── Création du compte
└──────┬───────┘
       │
       │ Profil complété + demande
       ▼
┌──────────────┐
│  PENDING_    │◄──── En attente de vérification
│ CERTIFICATION│
└──────┬───────┘
       │
  ┌────┴────┐
  │         │
  ▼         ▼
┌──────┐  ┌──────────┐
│ACTIVE│  │ REJECTED │──► Peut re-soumettre
└──┬───┘  └──────────┘
   │
   ├── Abonnement actif ──► ACTIVE + SUBSCRIBED
   │
   └── Abonnement expiré ──► ACTIVE (limité)
```

### 5.3 États d'un Message

```
┌─────────┐     Envoyé      ┌──────────┐
│ PENDING │────────────────►│   SENT   │
└─────────┘                 └────┬─────┘
                                 │
                                 │ Reçu par le serveur
                                 ▼
                           ┌──────────┐
                           │DELIVERED │
                           └────┬─────┘
                                │
                                │ Ouvert par destinataire
                                ▼
                           ┌──────────┐
                           │   READ   │
                           └──────────┘
```

---

## 6. Architecture des modules Laravel

### 6.1 Structure modulaire

```
app/
│
├── Modules/
│   │
│   ├── Auth/                   Authentification & autorisation
│   │   ├── Controllers/        AuthController, SocialiteController
│   │   ├── Services/           AuthService, TokenService
│   │   ├── Requests/           LoginRequest, RegisterRequest
│   │   └── Policies/           UserPolicy
│   │
│   ├── Artist/                 Gestion des tatoueurs
│   │   ├── Controllers/        ArtistController, PortfolioController
│   │   ├── Services/           ArtistService, CertificationService
│   │   ├── Models/             Artist, PortfolioItem
│   │   └── Resources/          ArtistResource, ArtistCollection
│   │
│   ├── Booking/                Réservations & planning
│   │   ├── Controllers/        BookingController, AvailabilityController
│   │   ├── Services/           BookingService, AvailabilityService
│   │   ├── Models/             Booking
│   │   └── Jobs/               SendBookingConfirmation
│   │
│   ├── Payment/                Paiements & abonnements
│   │   ├── Controllers/        PaymentController, WebhookController
│   │   ├── Services/           DepositService, SubscriptionService
│   │   └── Listeners/          HandleStripeWebhook
│   │
│   ├── Messaging/              Messagerie temps réel
│   │   ├── Controllers/        ConversationController, MessageController
│   │   ├── Events/             MessageSent
│   │   ├── Models/             Conversation, Message
│   │   └── Channels/           ConversationChannel
│   │
│   ├── Shop/                   Boutique produits
│   │   ├── Controllers/        ProductController, OrderController
│   │   ├── Services/           OrderService, CartService
│   │   └── Models/             Product, Order, OrderItem
│   │
│   ├── Review/                 Avis & notes
│   │   ├── Controllers/        ReviewController
│   │   ├── Services/           ReviewService
│   │   └── Models/             Review
│   │
│   ├── Notification/           Notifications push, email, SMS
│   │   ├── Notifications/      BookingConfirmed, MessageReceived
│   │   └── Jobs/               SendPushNotification
│   │
│   └── Content/                Blog & articles
│       ├── Controllers/        ArticleController
│       └── Models/             Article
│
├── Http/
│   └── Middleware/
│       ├── Authenticate.php
│       ├── CheckRole.php        Vérification des rôles
│       └── ScreenshotProtection.php
│
└── Console/
    └── Commands/
        ├── ExpireSubscriptions.php   CRON quotidien
        └── SendReminderNotifications.php
```

### 6.2 Dépendances entre modules

```
         ┌─────────┐
         │  Auth   │◄──── Tous les modules dépendent de Auth
         └────┬────┘
              │
     ┌────────┼────────────────┐
     │        │                │
     ▼        ▼                ▼
┌─────────┐ ┌─────────┐ ┌──────────┐
│ Artist  │ │ Booking │ │  Shop    │
└────┬────┘ └────┬────┘ └────┬─────┘
     │           │            │
     │      ┌────┴────┐       │
     │      │ Payment │◄──────┘
     │      └────┬────┘
     │           │
     ▼           ▼
┌─────────┐ ┌──────────────┐
│ Review  │ │ Notification │◄── Tous les modules peuvent
└─────────┘ └──────────────┘    déclencher des notifications
```

---

## 7. Flux de paiement

### 7.1 Vue d'ensemble des flux financiers

```
                    PLATEFORME (Stripe Connect)
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  Acompte    │  │  Paiement   │  │ Abonnement  │
   │  (30%)      │  │  final      │  │  tatoueur   │
   │             │  │  (70%)      │  │  mensuel    │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Stripe     │
                    │  Account    │
                    │ (plateforme)│
                    └──────┬──────┘
                           │
              ┌────────────┴───────────┐
              │ Commission             │ Reversement
              │ plateforme (X%)        │ tatoueur (Y%)
              ▼                        ▼
       ┌─────────────┐          ┌─────────────┐
       │  Revenus    │          │  Stripe     │
       │  plateforme │          │  Connected  │
       └─────────────┘          │  Account    │
                                │ (tatoueur)  │
                                └─────────────┘
```

### 7.2 Gestion des cas d'erreur paiement

| Cas | Comportement |
|---|---|
| Paiement échoué (acompte) | Booking reste PENDING, notification à l'utilisateur, retry possible |
| Paiement échoué (solde) | Booking reste AWAITING_PAYMENT, relance J+1, J+3 |
| Remboursement annulation | Stripe Refund automatique via webhook, booking → CANCELLED |
| Abonnement impayé | Tatoueur → plan free après 3 jours de grâce (Cashier grace period) |
| Webhook non reçu | Vérification signature Stripe, idempotence sur l'event ID |

---

## 8. Flux médias & protection des photos

### 8.1 Upload d'une photo (tatoueur)

```
Tatoueur        App Mobile         API Laravel         Cloudinary
   │                │                   │                  │
   │── Sélectionne─►│                   │                  │
   │   photo        │                   │                  │
   │                │── POST /media/    │                  │
   │                │   upload ────────►│                  │
   │                │                   │── Upload + ─────►│
   │                │                   │   Watermark      │
   │                │                   │   auto           │
   │                │                   │◄── public_id ────│
   │                │                   │    + URLs        │
   │                │                   │── Persist ───────────► Supabase
   │                │◄── URLs signées ──│                  │
   │◄── Aperçu ─────│                   │                  │
```

### 8.2 Consultation d'une photo (client)

```
Client          App Mobile         API Laravel         Cloudinary
  │                 │                   │                  │
  │── Ouvre ───────►│                   │                  │
  │   galerie       │── GET /artists/   │                  │
  │                 │   :id/portfolio ─►│                  │
  │                 │                   │── Génère URL ────►│
  │                 │                   │   signée         │
  │                 │                   │   (15 min)       │
  │                 │◄── URLs signées ──│                  │
  │                 │── Charge images ────────────────────►│
  │◄── Galerie ─────│                   │                  │
  │   affichée      │                   │                  │
  │                 │                   │                  │
  │── Screenshot ──►│                   │                  │
  │   (tentative)   │── FLAG_SECURE  ───│                  │
  │                 │   bloque (Android)│                  │
  │◄── Bloqué ──────│                   │                  │
```

### 8.3 Niveaux de protection par contexte

| Contexte | Protection |
|---|---|
| Galerie tatoueur | URL signée 15 min + watermark visible |
| FYP (découverte) | Thumbnail basse résolution, URL signée 1h |
| Visualisation AR | Watermark + filigrane "Aperçu" |
| Photo profil | URL publique (pas de protection) |
| Photos médicales (avant/après) | URL signée 5 min + accès restreint au tatoueur du RDV |

---

## 9. Flux temps réel — Messagerie

### 9.1 Architecture Reverb

```
┌────────────────┐       ┌────────────────────────────────┐
│   App Mobile   │       │         Laravel                │
│  (Echo client) │       │                                │
│                │  WSS  │  ┌──────────┐  ┌───────────┐   │
│ echo.private() │◄─────►│  │  Reverb  │  │   Redis   │   │
│ .listen()      │       │  │  Server  │◄►│  Pub/Sub  │   │
└────────────────┘       │  └──────────┘  └───────────┘   │
                         │       ▲                        │
┌────────────────┐       │       │ Broadcast              │
│   App Mobile   │  WSS  │  ┌────┴─────┐                  │
│  (tatoueur)    │◄─────►│  │  API     │                  │
│                │       │  │  REST    │                  │
│ echo.private() │       │  └──────────┘                  │
│ .listen()      │       └────────────────────────────────┘
└────────────────┘
```

### 9.2 Sécurisation des channels

```
Règle d'accès channel privé "conversation.{id}" :

User authentifié ?
    │
    ├── NON → Connexion WebSocket refusée (401)
    │
    └── OUI → Est-il user_id OU artist_user_id de la conversation ?
                    │
                    ├── NON → Accès channel refusé (403)
                    │
                    └── OUI → Connexion autorisée ✅
```

---

## 10. Sécurité & RGPD

### 10.1 Modèle de menaces

| Menace | Vecteur | Contre-mesure |
|---|---|---|
| Vol de token | Interception réseau | HTTPS/TLS 1.3 uniquement, HSTS |
| Brute force login | API /auth/login | Rate limiting 5 tentatives / 15 min par IP |
| Accès photo non autorisé | URL directe | URLs signées avec expiration |
| Injection SQL | Paramètres API | Eloquent ORM (requêtes paramétrées) |
| Mass assignment | Données POST | Laravel `$fillable` strict sur tous les modèles |
| CORS non maîtrisé | Appels cross-origin | Liste blanche des origines autorisées |
| Screenshot | Écrans sensibles | FLAG_SECURE Android + API iOS |
| Accès données médicales | API | Row Level Security + rôles Laravel |

### 10.2 Flux d'authentification & tokens

```
Mobile                  API Laravel              Redis (Upstash)
  │                          │                         │
  │── POST /auth/login ─────►│                         │
  │                          │── Vérifie credentials   │
  │                          │── Génère Access Token   │
  │                          │── Génère Refresh Token  │
  │                          │── Store Refresh Token ─►│
  │◄── { access_token,       │                         │
  │      refresh_token }     │                         │
  │                          │                         │
  │  [15 min plus tard]      │                         │
  │── POST /auth/refresh ───►│                         │
  │   { refresh_token }      │── Vérifie token ───────►│
  │                          │◄── Valide ──────────────│
  │                          │─ Invalide ancien token ►│
  │                          │──Génère nouveaux tokens │
  │◄── Nouveaux tokens ──────│                         │
```

### 10.3 Conformité RGPD — Données de santé (Art. 9)

| Obligation | Implémentation |
|---|---|
| **Consentement explicite** | Écran dédié onboarding, stocké avec timestamp et version des CGU |
| **Finalité limitée** | `skin_conditions` visible uniquement par l'utilisateur et le tatoueur du RDV actif |
| **Droit d'accès** | `GET /users/me/data` → export JSON complet |
| **Droit à l'oubli** | `DELETE /users/me` → suppression en cascade + anonymisation des avis |
| **Portabilité** | Export JSON structuré sur demande |
| **Minimisation** | Seules les données strictement nécessaires sont collectées |
| **AIPD** | Analyse d'Impact obligatoire avant mise en production (données de santé) |
| **DPO** | Désignation obligatoire, coordonnées dans les CGU |

### 10.4 Politique de rétention des données

| Donnée | Durée de rétention | Action à expiration |
|---|---|---|
| Compte utilisateur actif | Indéfinie | — |
| Compte supprimé | 30 jours | Suppression définitive |
| Messages | 12 mois après la dernière activité | Archivage chiffré |
| Logs applicatifs | 90 jours | Suppression automatique |
| Données de paiement | 5 ans (obligation légale) | Anonymisation du reste |
| Photos portfolio | Jusqu'à suppression par le tatoueur | — |

---

## 11. Infrastructure & déploiement

### 11.1 Environnements

| Environnement | URL | Déclencheur | Base de données |
|---|---|---|---|
| **Local** | localhost | Manuel (Docker Compose) | PostgreSQL local |
| **Staging** | staging.app.com | Push sur `develop` | Supabase staging project |
| **Production** | app.com | Merge sur `main` (manuel) | Supabase production project |

### 11.2 Pipeline CI/CD (GitHub Actions)

```
Push / PR
    │
    ▼
┌───────────────┐
│  Lint & Tests │  PHPStan (analyse statique)
│               │  Pest (tests unitaires + feature)
│               │  Tests React Native (Jest)
└───────┬───────┘
        │ OK
        ▼
┌───────────────┐
│    Security   │  Audit dépendances (composer audit)
│    Checks     │  Scan SAST (Semgrep)
└───────┬───────┘
        │ OK
        ▼
┌───────────────┐
│     Build     │  Docker image Laravel + Octane
│               │  Build Next.js (Vercel preview)
└───────┬───────┘
        │
        ├── Branche develop ──► Deploy Staging (automatique)
        │
        └── Branche main ─────► Deploy Production
                                 (validation manuelle requise)
```

### 11.3 Variables d'environnement requises

| Variable | Service | Sensible |
|---|---|---|
| `DB_HOST`, `DB_PORT`, `DB_NAME` | Supabase | ✅ |
| `DB_USERNAME`, `DB_PASSWORD` | Supabase | ✅ |
| `REDIS_URL` | Upstash | ✅ |
| `STRIPE_SECRET_KEY` | Stripe | ✅ |
| `STRIPE_WEBHOOK_SECRET` | Stripe | ✅ |
| `CASHIER_KEY`, `CASHIER_SECRET` | Stripe Cashier | ✅ |
| `CLOUDINARY_URL` | Cloudinary | ✅ |
| `MAPBOX_TOKEN` | Mapbox | ✅ |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | OAuth | ✅ |
| `APPLE_CLIENT_ID`, `APPLE_CLIENT_SECRET` | OAuth | ✅ |
| `REVERB_APP_ID`, `REVERB_APP_KEY` | Laravel Reverb | ✅ |
| `FCM_SERVER_KEY` | Firebase (Push) | ✅ |
| `APP_KEY` | Laravel | ✅ |
| `APP_ENV`, `APP_DEBUG` | Laravel | ❌ |

> Toutes les variables sensibles sont stockées dans les secrets GitHub Actions et injectées au déploiement. Aucune valeur sensible dans le dépôt Git.

### 11.4 Stratégie de sauvegarde

| Donnée | Fréquence | Rétention | Méthode |
|---|---|---|---|
| Base PostgreSQL | Quotidienne | 30 jours | Supabase automated backups |
| Médias Cloudinary | Continue | Illimitée | Réplication Cloudinary |
| Redis (sessions) | Non critique | — | Upstash persistence optionnelle |
| Logs applicatifs | Continue | 90 jours | Railway log drain |

---