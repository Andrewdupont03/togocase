# Guide de mise en production — TogoCase

Checklist complète à suivre **dans l'ordre**, avant de lancer `npm start` en production. Les dépendances actuelles ont été vérifiées : **0 vulnérabilité connue** (`npm audit`), Node >=18 requis (testé avec Node 22).

---

## 0. Remplacer les fichiers corrigés

Avant tout, copiez dans votre projet réel les 7 fichiers livrés précédemment, en écrasant les originaux :

`routes.js`, `server.js`, `.env.example`, `app.js`, `brands.html`, `models.html`, `components.html`.

---

## 1. Base de données PostgreSQL

**Option recommandée (la plus simple) : un service managé** — Railway, Render, ou Supabase. Créez une base Postgres chez l'un d'eux, ils vous donnent une `DATABASE_URL` toute prête (ex: `postgres://user:pass@host:5432/dbname`).

**Option auto-hébergée** (VPS avec Postgres installé) :

```bash
sudo -u postgres psql -c "CREATE DATABASE togocase;"
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'un_mot_de_passe_fort';"
```

---

## 2. Le fichier `.env` — le cœur de la configuration production

```bash
cp .env.example .env
```

Remplissez `.env` avec les **vraies valeurs de production** :

| Variable | Valeur en production | Pourquoi |
|---|---|---|
| `DATABASE_URL` | fournie par Railway/Render/Supabase | connexion tout-en-un (sinon utilisez les `PGHOST/PGUSER/...` séparés) |
| `PGSSL` | `true` si base managée | la plupart des fournisseurs cloud l'exigent — sans ça, la connexion échoue souvent avec une erreur SSL |
| `PORT` | laissez tel quel | la plateforme (Render/Railway) injecte son propre PORT automatiquement |
| `NODE_ENV` | `production` | masque les détails d'erreur internes aux visiteurs, active les bons logs |
| `CORS_ORIGIN` | `https://votre-domaine-frontend.com` (**jamais `*`**) | sécurité : sans ça, votre site ne pourra même pas appeler l'API depuis le navigateur, ou pire, n'importe quel site pourrait le faire |
| `ADMIN_API_KEY` | générée aléatoirement (commande ci-dessous) | protège toutes les routes de gestion (catalogue, commandes, clients) |

Générez la clé admin :

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Copiez le résultat dans `ADMIN_API_KEY=` du `.env`. **Ne partagez cette clé qu'avec vous-même** (Postman, futur outil d'admin).

⚠️ **`.env` ne doit JAMAIS être commité sur Git.** Vérifiez qu'il est bien dans votre `.gitignore` (avec `node_modules/`) :

```bash
echo -e ".env\nnode_modules/" >> .gitignore
```

---

## 3. Installer les dépendances

```bash
npm ci
```

(`npm ci` plutôt que `npm install` : installation reproductible à partir de `package-lock.json`, recommandé en production.)

---

## 4. Peupler la base — ⚠️ ZONE DANGEREUSE

```bash
npm run seed
```

**Ne lancez cette commande QUE lors du tout premier déploiement, sur une base vide.** `seed.js` fait un `TRUNCATE ... CASCADE` qui **efface définitivement** marques, modèles, produits, promotions, **commandes et clients**. Si votre base contient déjà de vraies commandes/clients, **ne relancez jamais `npm run seed`** — vous perdriez tout. Les mises à jour du schéma se font automatiquement au démarrage du serveur (`initSchema()` dans `server.js`), sans toucher aux données existantes.

---

## 5. Pointer le front-end vers le VRAI backend

Par défaut, `app.js` appelle `http://localhost:3000/api` — ça ne fonctionnera pas en production. Dans **chacun** des 4 fichiers HTML (`pages.html`, `brands.html`, `models.html`, `components.html`), juste **avant** la ligne `<script src="assets/app.js"></script>`, ajoutez :

```html
<script>window.TOGOCASE_API_BASE = "https://votre-backend.example.com/api";</script>
<script src="assets/app.js"></script>
```

Remplacez par l'URL réelle de votre backend déployé (https, pas localhost).

---

## 6. Où héberger quoi

Ce backend (`server.js`) ne sert **que** les routes `/api/*` — il ne sert pas vos fichiers HTML/CSS/JS. Deux options :

- **Séparé (le plus simple avec le code actuel)** : backend sur Render/Railway/Heroku/VPS (process Node persistant), front-end sur un hébergeur statique (Netlify, Vercel, GitHub Pages...). Pensez alors à mettre l'URL exacte du front dans `CORS_ORIGIN`.
- **Même origine** : je peux ajouter `express.static()` à `server.js` pour servir aussi vos pages HTML depuis le même serveur (plus simple à déployer, un seul service) — dites-le-moi si vous préférez cette option, je vous fournirai le fichier modifié.

---

## 7. HTTPS obligatoire

Render/Railway/Heroku fournissent HTTPS automatiquement. Sur un VPS auto-géré, ajoutez un reverse proxy (Nginx ou Caddy) + certificat Let's Encrypt — ne servez jamais de commandes/données clients en HTTP nu.

---

## 8. Garder le serveur en vie (VPS uniquement)

Ne lancez jamais `node server.js` directement dans un terminal en production (il meurt à la fermeture du terminal). Utilisez un gestionnaire de process :

```bash
npm install -g pm2
pm2 start server.js --name togocase-api
pm2 save && pm2 startup
```

(Sur Render/Railway/Heroku, la plateforme gère déjà ça — rien à faire.)

---

## 9. Checklist finale avant de couper le ruban

- [ ] `.env` rempli avec les vraies valeurs, **jamais commité**
- [ ] `CORS_ORIGIN` = domaine(s) réel(s) exact(s), **jamais `*`**
- [ ] `NODE_ENV=production`
- [ ] `ADMIN_API_KEY` générée, forte, gardée privée
- [ ] `PGSSL=true` si base managée
- [ ] `window.TOGOCASE_API_BASE` mis à jour dans les 4 pages HTML
- [ ] `npm run seed` lancé **une seule fois**, sur base vide

---

## 10. Test de fumée après déploiement

Remplacez `https://votre-backend.example.com` par votre vraie URL :

```bash
curl https://votre-backend.example.com/api/health
curl https://votre-backend.example.com/api/brands
# Doit renvoyer 401 (sécurité active) :
curl -o /dev/null -w "%{http_code}\n" https://votre-backend.example.com/api/orders
# Doit fonctionner avec votre clé :
curl https://votre-backend.example.com/api/orders -H "Authorization: Bearer VOTRE_ADMIN_API_KEY"
```

---

## 11. Enfin

```bash
npm start
```# Guide de mise en production — TogoCase

Checklist complète à suivre **dans l'ordre**, avant de lancer `npm start` en production. Les dépendances actuelles ont été vérifiées : **0 vulnérabilité connue** (`npm audit`), Node >=18 requis (testé avec Node 22).

---

## 0. Remplacer les fichiers corrigés

Avant tout, copiez dans votre projet réel les 7 fichiers livrés précédemment, en écrasant les originaux :

`routes.js`, `server.js`, `.env.example`, `app.js`, `brands.html`, `models.html`, `components.html`.

---

## 1. Base de données PostgreSQL

**Option recommandée (la plus simple) : un service managé** — Railway, Render, ou Supabase. Créez une base Postgres chez l'un d'eux, ils vous donnent une `DATABASE_URL` toute prête (ex: `postgres://user:pass@host:5432/dbname`).

**Option auto-hébergée** (VPS avec Postgres installé) :

```bash
sudo -u postgres psql -c "CREATE DATABASE togocase;"
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'un_mot_de_passe_fort';"
```

---

## 2. Le fichier `.env` — le cœur de la configuration production

```bash
cp .env.example .env
```

Remplissez `.env` avec les **vraies valeurs de production** :

| Variable | Valeur en production | Pourquoi |
|---|---|---|
| `DATABASE_URL` | fournie par Railway/Render/Supabase | connexion tout-en-un (sinon utilisez les `PGHOST/PGUSER/...` séparés) |
| `PGSSL` | `true` si base managée | la plupart des fournisseurs cloud l'exigent — sans ça, la connexion échoue souvent avec une erreur SSL |
| `PORT` | laissez tel quel | la plateforme (Render/Railway) injecte son propre PORT automatiquement |
| `NODE_ENV` | `production` | masque les détails d'erreur internes aux visiteurs, active les bons logs |
| `CORS_ORIGIN` | `https://votre-domaine-frontend.com` (**jamais `*`**) | sécurité : sans ça, votre site ne pourra même pas appeler l'API depuis le navigateur, ou pire, n'importe quel site pourrait le faire |
| `ADMIN_API_KEY` | générée aléatoirement (commande ci-dessous) | protège toutes les routes de gestion (catalogue, commandes, clients) |

Générez la clé admin :

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Copiez le résultat dans `ADMIN_API_KEY=` du `.env`. **Ne partagez cette clé qu'avec vous-même** (Postman, futur outil d'admin).

⚠️ **`.env` ne doit JAMAIS être commité sur Git.** Vérifiez qu'il est bien dans votre `.gitignore` (avec `node_modules/`) :

```bash
echo -e ".env\nnode_modules/" >> .gitignore
```

---

## 3. Installer les dépendances

```bash
npm ci
```

(`npm ci` plutôt que `npm install` : installation reproductible à partir de `package-lock.json`, recommandé en production.)

---

## 4. Peupler la base — ⚠️ ZONE DANGEREUSE

```bash
npm run seed
```

**Ne lancez cette commande QUE lors du tout premier déploiement, sur une base vide.** `seed.js` fait un `TRUNCATE ... CASCADE` qui **efface définitivement** marques, modèles, produits, promotions, **commandes et clients**. Si votre base contient déjà de vraies commandes/clients, **ne relancez jamais `npm run seed`** — vous perdriez tout. Les mises à jour du schéma se font automatiquement au démarrage du serveur (`initSchema()` dans `server.js`), sans toucher aux données existantes.

---

## 5. Pointer le front-end vers le VRAI backend

Par défaut, `app.js` appelle `http://localhost:3000/api` — ça ne fonctionnera pas en production. Dans **chacun** des 4 fichiers HTML (`pages.html`, `brands.html`, `models.html`, `components.html`), juste **avant** la ligne `<script src="assets/app.js"></script>`, ajoutez :

```html
<script>window.TOGOCASE_API_BASE = "https://votre-backend.example.com/api";</script>
<script src="assets/app.js"></script>
```

Remplacez par l'URL réelle de votre backend déployé (https, pas localhost).

---

## 6. Où héberger quoi

Ce backend (`server.js`) ne sert **que** les routes `/api/*` — il ne sert pas vos fichiers HTML/CSS/JS. Deux options :

- **Séparé (le plus simple avec le code actuel)** : backend sur Render/Railway/Heroku/VPS (process Node persistant), front-end sur un hébergeur statique (Netlify, Vercel, GitHub Pages...). Pensez alors à mettre l'URL exacte du front dans `CORS_ORIGIN`.
- **Même origine** : je peux ajouter `express.static()` à `server.js` pour servir aussi vos pages HTML depuis le même serveur (plus simple à déployer, un seul service) — dites-le-moi si vous préférez cette option, je vous fournirai le fichier modifié.

---

## 7. HTTPS obligatoire

Render/Railway/Heroku fournissent HTTPS automatiquement. Sur un VPS auto-géré, ajoutez un reverse proxy (Nginx ou Caddy) + certificat Let's Encrypt — ne servez jamais de commandes/données clients en HTTP nu.

---

## 8. Garder le serveur en vie (VPS uniquement)

Ne lancez jamais `node server.js` directement dans un terminal en production (il meurt à la fermeture du terminal). Utilisez un gestionnaire de process :

```bash
npm install -g pm2
pm2 start server.js --name togocase-api
pm2 save && pm2 startup
```

(Sur Render/Railway/Heroku, la plateforme gère déjà ça — rien à faire.)

---

## 9. Checklist finale avant de couper le ruban

- [ ] `.env` rempli avec les vraies valeurs, **jamais commité**
- [ ] `CORS_ORIGIN` = domaine(s) réel(s) exact(s), **jamais `*`**
- [ ] `NODE_ENV=production`
- [ ] `ADMIN_API_KEY` générée, forte, gardée privée
- [ ] `PGSSL=true` si base managée
- [ ] `window.TOGOCASE_API_BASE` mis à jour dans les 4 pages HTML
- [ ] `npm run seed` lancé **une seule fois**, sur base vide

---

## 10. Test de fumée après déploiement

Remplacez `https://votre-backend.example.com` par votre vraie URL :

```bash
curl https://votre-backend.example.com/api/health
curl https://votre-backend.example.com/api/brands
# Doit renvoyer 401 (sécurité active) :
curl -o /dev/null -w "%{http_code}\n" https://votre-backend.example.com/api/orders
# Doit fonctionner avec votre clé :
curl https://votre-backend.example.com/api/orders -H "Authorization: Bearer VOTRE_ADMIN_API_KEY"
```

---

## 11. Enfin

```bash
npm start
```
