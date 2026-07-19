# MaintPro — API Backend (Node.js + Express + PostgreSQL)

Backend multi-organisations / multi-sites pour remplacer le stockage `localStorage`
de l'application MaintPro par une vraie base de données partagée entre plusieurs
sites et équipes.

## 1. Démarrage rapide

```bash
cp .env.example .env
# éditez .env : DATABASE_URL, JWT_SECRET, CORS_ORIGINS

npm install
npm run migrate   # crée les tables dans PostgreSQL
npm start         # démarre l'API sur http://localhost:4000
```

Prérequis : Node.js 18+, une base PostgreSQL 14+ accessible (locale, Docker, RDS, Supabase...).

Exemple rapide avec Docker pour une base locale :
```bash
docker run --name maintpro-db -e POSTGRES_USER=maintpro -e POSTGRES_PASSWORD=maintpro \
  -e POSTGRES_DB=maintpro -p 5432:5432 -d postgres:16
```

## 2. Modèle de données

- **organizations** : le client (une entreprise).
- **sites** : les sites/usines/ateliers d'une organisation (plusieurs sites par organisation).
- **users** : les comptes utilisateurs, rattachés à une organisation.
- **user_sites** : donne à un utilisateur un rôle (`admin`, `manager`, `technician`, `viewer`) sur un site précis. Un utilisateur peut avoir accès à plusieurs sites avec des rôles différents.
- **equipments, work_orders, stock_items, technicians, history_events, notifications** : toutes rattachées à un `site_id` — les données d'un site ne sont jamais visibles par un autre site.

Ce modèle permet : plusieurs équipes, plusieurs sites, et un contrôle d'accès par rôle **sans tout mélanger**.

## 3. Authentification

- `POST /api/auth/register-organization` — crée une nouvelle organisation + un premier compte admin + un site par défaut.
- `POST /api/auth/login` — retourne un JWT (`Authorization: Bearer <token>` à envoyer sur toutes les requêtes suivantes).
- `GET /api/auth/my-sites` — liste les sites accessibles à l'utilisateur connecté avec son rôle sur chacun.

## 4. Routes principales (toutes sous `/api/sites/:siteId/...`, JWT requis)

| Ressource | Endpoints |
|---|---|
| Équipements | `GET/POST /equipments`, `PATCH/DELETE /equipments/:id` |
| Ordres de travail | `GET/POST /work-orders`, `PATCH /work-orders/:id/complete`, `DELETE /work-orders/:id` |
| Stock | `GET /stock`, `POST /stock/movement` (entrée/sortie, crée la pièce si besoin), `DELETE /stock/:id` |
| Techniciens | `GET/POST /technicians`, `DELETE /technicians/:id` |
| Historique | `GET /history` |
| Notifications | `GET /notifications`, `PATCH /notifications/:id/read`, `PATCH /notifications/read-all` |

Chaque route vérifie le rôle minimum requis sur le site via `requireSiteAccess('manager')` etc.

## 5. Sécurité déjà en place

- Mots de passe hashés avec bcrypt (12 rounds).
- JWT signé, expiration configurable.
- `helmet` (en-têtes de sécurité HTTP) + CORS restreint à une liste blanche de domaines.
- Rate limiting global + limite stricte sur `/auth/login` contre le brute-force.
- Validation stricte des entrées avec `zod` sur toutes les routes de création.
- Requêtes SQL paramétrées partout (`$1, $2...`) → pas d'injection SQL.
- Isolation stricte par `site_id` sur chaque requête.

## 6. Migrer le frontend de localStorage vers l'API

Le fichier `maintpro-xss-fixed.html` actuel stocke tout dans `localStorage` (fonctions `saveData()` / `loadData()`) et fait toute la logique côté client. Pour brancher ce backend, il faut remplacer :

1. **L'écriture locale par des appels `fetch`** : chaque action (`addWorkOrder`, `addEquipment`, `addStockItem`, etc.) doit `POST`/`PATCH` vers l'API au lieu de modifier `this.workOrders` directement puis `saveData()`.
2. **Le chargement initial** : `loadData()` doit être remplacé par des `fetch` vers `GET /equipments`, `GET /work-orders`, etc. au démarrage (`App.init()`), après connexion (login).
3. **Ajouter un écran de connexion** (email + mot de passe) qui appelle `/api/auth/login`, stocke le JWT (en mémoire ou `sessionStorage`, pas `localStorage` pour un token sensible idéalement avec expiration courte), puis charge le(s) site(s) accessibles via `/api/auth/my-sites`.
4. **Un sélecteur de site** si l'utilisateur a accès à plusieurs sites (visible dans la sidebar), qui recharge les données pour le site sélectionné.

C'est un chantier de réécriture significatif de la couche données du frontend (le reste — UI, rendu, PDF — peut rester quasi identique). Je peux m'en charger dans une prochaine étape si tu veux : soit une réécriture progressive (fonction par fonction), soit une passe complète.

## 7. Ce qui reste à ajouter selon vos besoins

- Gestion des rôles côté UI (masquer les boutons Supprimer pour un `viewer`, etc.)
- Upload de photos (actuellement `photo_url` en base — nécessite un stockage fichiers : S3, Cloudinary, ou disque local + serveur statique)
- Invitations d'utilisateurs / gestion des comptes (actuellement seul `register-organization` crée un admin)
- Rafraîchissement de token (le JWT actuel expire après `JWT_EXPIRES_IN` sans renouvellement automatique)
- Tests automatisés
