# ToolShare

Plateforme web de réservation et de gestion de matériel professionnel, répartie entre plusieurs magasins.

Application **React** (responsive) connectée à l'API REST **Node.js / Express / MongoDB** développée au Sprint 2.

> Les éléments entre crochets `[...]` sont à remplacer par vos valeurs.

## Sommaire

1. [Fonctionnalités](#fonctionnalités)
2. [Technologies](#technologies)
3. [Architecture](#architecture)
4. [Prérequis](#prérequis)
5. [Configuration](#configuration)
6. [Installation et lancement](#installation-et-lancement)
7. [Tests](#tests)
8. [Docker](#docker)
9. [CI/CD](#cicd)
10. [Comptes et démonstration](#comptes-et-démonstration)
11. [Sécurité côté client](#sécurité-côté-client)
12. [Planification et conception](#planification-et-conception)

## Fonctionnalités

Il n'y a pas d'inscription publique : les comptes sont créés par les profils autorisés.

**Client**
- Consulter les magasins et leur matériel (pages de détail, recherche, filtres, pagination)
- Réserver un équipement (matériel, magasin, début, durée) avec récapitulatif avant confirmation
- Voir les conflits de réservation signalés par l'API
- Consulter son historique, filtrer par statut et annuler lorsque c'est permis

**Manager** (limité à son magasin)
- Tableau de bord du magasin
- Gérer le matériel et les réservations de son magasin
- Enregistrer la remise et le retour d'un équipement (retard, dommages)

**Administrateur**
- Tableau de bord global
- Gérer les magasins, les utilisateurs et les managers (création de comptes, affectation à un magasin)
- Superviser toutes les réservations

Statuts d'une réservation : **Confirmée**, **En cours**, **Terminée**, **Annulée**.

## Technologies

| Domaine | Outils |
|---|---|
| Frontend | React, Vite, React Router, Context API |
| Données distantes | Axios, TanStack Query (React Query) |
| Tests | Jest, React Testing Library |
| Backend | Node.js, Express, MongoDB |
| Conteneurs | Docker, Docker Compose, Nginx |
| CI/CD | GitHub Actions, Docker Hub |
| Gestion de projet | Jira, Figma |

## Architecture

```
toolshare/
├── backend/                 API REST Node.js / Express
├── frontend/
│   ├── src/
│   │   ├── api/             instance Axios (client.js) et intercepteurs
│   │   ├── auth/            AuthProvider, useAuth
│   │   ├── components/
│   │   │   ├── ui/          Button, Input, Card, Badge, Loader, ErrorMessage
│   │   │   └── layout/      Sidebar
│   │   ├── layouts/         ClientLayout, ManagerLayout, AdminLayout
│   │   ├── pages/           auth, client, manager, admin, NotFound, Forbidden
│   │   ├── services/        un service par ressource (stores, equipment, reservations, users)
│   │   ├── hooks/           hooks React Query (useStores, useReservations, ...)
│   │   ├── routes/          AppRouter, ProtectedRoute, RoleRedirect
│   │   └── utils/           validation, formatage des dates
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml
├── .github/workflows/       ci.yml et cd.yml
└── README.md
```

**Principes**
- Les pages ne contiennent que l'affichage ; les appels API sont dans `services/` et `hooks/`.
- La session (utilisateur, token) est gérée par `AuthContext` ; le token est envoyé par l'intercepteur Axios.
- React Query gère le cache, les états de chargement et d'erreur, et l'invalidation après chaque mutation.
- Les routes sont protégées par rôle via `ProtectedRoute`. Les contrôles côté client ne remplacent jamais les autorisations du backend.

## Prérequis

- Node.js `[version, ex. 20]` et npm
- Git
- MongoDB (local ou via Docker)
- Docker et Docker Compose (pour la partie conteneurs)

## Configuration

### Frontend

Copier `frontend/.env.example` vers `frontend/.env` :

```env
VITE_API_URL=http://localhost:[PORT_API]/api
```

### Backend

Copier `backend/.env.example` vers `backend/.env` et renseigner les variables du Sprint 2, par exemple :

```env
PORT=[PORT_API]
MONGODB_URI=mongodb://localhost:27017/toolshare
JWT_SECRET=[secret]
CORS_ORIGIN=http://localhost:5173
```

Aucun fichier `.env` ni secret ne doit être versionné.

## Installation et lancement

```bash
git clone [URL_DU_DEPOT] toolshare
cd toolshare

# Backend
cd backend
npm install
npm run dev          # ou npm start

# Frontend (dans un autre terminal)
cd frontend
npm install
npm run dev
```

- Frontend : http://localhost:5173
- API : http://localhost:[PORT_API]

Autres commandes frontend :

```bash
npm run lint         # ESLint
npm run build        # build de production
npm test             # tests Jest
```

## Tests

Les tests d'intégration utilisent **Jest** et **React Testing Library**. Les appels API sont simulés (`jest.mock` sur le client Axios) : aucun serveur ni MongoDB n'est nécessaire.

Scénarios couverts :
- Rendu du catalogue, état de chargement et état d'erreur
- Validation du formulaire de réservation et message en cas de conflit
- Protection des routes et redirection selon le rôle
- Remise / retour et changement de statut

```bash
cd frontend
npm test
```

## Docker

Deux images applicatives distinctes :

| Image | Contenu |
|---|---|
| `toolshare-frontend` | React compilé, servi par Nginx (rechargement des routes SPA géré par `try_files`) |
| `toolshare-backend` | API Node.js / Express |

MongoDB utilise l'image officielle et n'est pas publiée.

### Lancer l'ensemble avec Docker Compose

```bash
docker compose up --build
```

- Frontend : http://localhost:[PORT_FRONT]
- API : http://localhost:[PORT_API]
- MongoDB : service interne `mongo`

Arrêter et supprimer les conteneurs :

```bash
docker compose down
```

### Construire les images séparément

```bash
docker build -t toolshare-frontend ./frontend
docker build -t toolshare-backend ./backend
```

Les secrets sont fournis par variables d'environnement (fichier `.env` non versionné) et ne sont jamais copiés dans les images.

## CI/CD

### CI : `.github/workflows/ci.yml`

Déclenchée sur push et pull request vers `main`. Pour le frontend et le backend :
1. Installation des dépendances
2. ESLint
3. Tests
4. Build de production

Toute erreur fait échouer le workflow.

### CD : `.github/workflows/cd.yml`

Après une CI réussie, sur un push dans `main` (y compris après fusion d'une pull request) :
1. Build de l'image frontend et de l'image backend
2. Connexion à Docker Hub avec les secrets GitHub
3. Publication dans deux dépôts distincts avec deux tags : `latest` et le hash du commit

| Dépôt Docker Hub | Lien |
|---|---|
| Frontend | `[username]/toolshare-frontend` |
| Backend | `[username]/toolshare-backend` |

Une pull request non fusionnée ne publie rien.

### Secrets GitHub à créer

*Settings → Secrets and variables → Actions*

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | nom d'utilisateur Docker Hub |
| `DOCKERHUB_TOKEN` | jeton d'accès Docker Hub |

## Comptes et démonstration

| Rôle | Email | Mot de passe |
|---|---|---|
| Administrateur | `[email]` | `[à fournir hors dépôt]` |
| Manager | `[email]` | `[à fournir hors dépôt]` |
| Client | `[email]` | `[à fournir hors dépôt]` |

Parcours à démontrer :
1. Connexion et redirection selon le rôle
2. **Client** : catalogue, réservation, conflit de réservation, historique
3. **Manager** : réservations de son magasin, remise, retour
4. **Admin** : tableau de bord, magasins, utilisateurs, réservations

## Sécurité côté client

- Formulaires validés (champs, dates, durée)
- Aucun HTML utilisateur injecté : pas de `dangerouslySetInnerHTML`, React échappe l'affichage
- Erreur **401** : session supprimée et retour à la page de connexion
- Erreur **403** : page « Accès refusé »
- Aucun secret dans le code ni dans les images ; configuration par variables d'environnement
- Le token est stocké dans `sessionStorage` et supprimé à la déconnexion

## Planification et conception

- Jira : projet `TOOLF`, découpage en Epics et tâches, échéances du 23 novembre au 4 décembre 2026 : `[lien]`
- Figma : maquettes des parcours Client, Manager et Administrateur et charte graphique : `[lien]`

**Charte graphique** : bleu marine `#1A2238`, orange `#C94F0C` (boutons) et `#F26B21` (accent), gris `#E6E7E9` (fond), `#F7F7F8` (surfaces), `#D5D7DA` (bordures), police Inter.

## Auteur

`[Nom Prénom]` — projet individuel, Sprint 3, 2026.
