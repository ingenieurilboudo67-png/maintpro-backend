# Rendre MaintPro opérationnel — 3 façons

## Option A — En ligne, accessible de partout (gratuit, ~15 min)

On va héberger le **backend** (API + base de données) sur Render.com, et le
**frontend** (le fichier HTML) sur Netlify Drop. Les deux ont un tier gratuit
sans carte bancaire.

⚠️ Limite à connaître : la base PostgreSQL gratuite de Render est supprimée
au bout de 30 jours (il faudra la recréer ou passer sur un plan payant ~7$/mois
pour une vraie mise en production durable). Le service web gratuit se met en
veille après 15 min d'inactivité (premier chargement un peu plus lent après
une pause, ~30-60 sec).

### 1. Mettre le backend sur GitHub
```bash
cd backend
git init
git add .
git commit -m "MaintPro backend"
# Crée un repo vide sur github.com puis :
git remote add origin https://github.com/TON-COMPTE/maintpro-backend.git
git branch -M main
git push -u origin main
```

### 2. Créer la base de données sur Render
1. Va sur https://render.com → crée un compte (pas de CB requise)
2. **New +** → **PostgreSQL** → nom `maintpro-db` → plan **Free** → Create
3. Une fois créée, copie l'**Internal Database URL** (tu en auras besoin à l'étape 3)

### 3. Déployer l'API
1. **New +** → **Web Service** → connecte ton repo `maintpro-backend`
2. Root Directory : laisse vide (ou `.` si le repo contient directement `backend/`)
3. Build Command : `npm install`
4. Start Command : `npm start`
5. Plan : **Free**
6. Dans **Environment**, ajoute les variables :
   - `DATABASE_URL` = l'Internal Database URL copiée à l'étape 2
   - `JWT_SECRET` = une longue chaîne aléatoire (ex: génère avec `openssl rand -hex 32`)
   - `JWT_EXPIRES_IN` = `8h`
   - `CORS_ORIGINS` = (laisse vide pour l'instant, on le complètera à l'étape 5)
7. Create Web Service → attends la fin du déploiement
8. Note l'URL donnée par Render, ex : `https://maintpro-backend-xxxx.onrender.com`

### 4. Initialiser la base de données
1. Dans le dashboard du Web Service → onglet **Shell**
2. Lance : `npm run migrate`
3. Tu dois voir `✅ Schéma appliqué avec succès.`

### 5. Déployer le frontend
1. Ouvre `maintpro-full-stack.html` dans un éditeur de texte
2. Remplace la ligne :
   ```js
   window.MAINTPRO_API_URL = 'http://localhost:4000/api';
   ```
   par :
   ```js
   window.MAINTPRO_API_URL = 'https://maintpro-backend-xxxx.onrender.com/api';
   ```
   (avec ton URL exacte de l'étape 3.8)
3. Renomme le fichier en `index.html`
4. Va sur https://app.netlify.com/drop et glisse-dépose le fichier
5. Netlify te donne une URL du type `https://random-name-123.netlify.app`

### 6. Autoriser le frontend à parler au backend (CORS)
1. Retourne sur Render → ton Web Service → **Environment**
2. Modifie `CORS_ORIGINS` = l'URL Netlify obtenue à l'étape 5 (ex: `https://random-name-123.netlify.app`)
3. Save → le service redémarre automatiquement

### 7. C'est prêt
Ouvre ton URL Netlify → "Nouvelle organisation" → crée ton compte admin →
tu es en ligne, accessible depuis n'importe quel navigateur, n'importe où.

---

## Option B — Backend en local sur ton PC

```bash
cd backend
cp .env.example .env
# édite .env : DATABASE_URL, JWT_SECRET

# Base de données locale via Docker :
docker run --name maintpro-db -e POSTGRES_USER=maintpro -e POSTGRES_PASSWORD=maintpro \
  -e POSTGRES_DB=maintpro -p 5432:5432 -d postgres:16

npm install
npm run migrate
npm start
```

⚠️ **Important** : n'ouvre PAS le fichier HTML en double-cliquant dessus
(`file://...`), le navigateur bloquera les requêtes vers l'API (CORS). Sers-le
via un petit serveur local à la place :
```bash
npx serve -l 5500 .
```
Puis ouvre `http://localhost:5500/maintpro-full-stack.html`, et mets
`CORS_ORIGINS=http://localhost:5500` dans le `.env` du backend.

---

## Option C — Mode démo hors-ligne (déjà fonctionnel, zéro configuration)

Ouvre simplement `maintpro-full-stack.html` et clique sur **"Continuer en
mode démo hors-ligne"**. Aucune installation nécessaire. Les données restent
dans le navigateur (comme avant), pas de compte, pas de multi-site — utile
pour tester l'interface rapidement.
