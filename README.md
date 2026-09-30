# Ghinel — Onboarding membres

Bienvenue dans l'équipe. Ce guide te donne les bases pour être opérationnel rapidement.

---

## Architecture

```
                        Internet
                           │
                      nginx-gateway
                    (port 80 / 443)
                    /      |       \
                   /       |        \
          ghinel-website  auth    papyrus-backend
          (Next.js :3003) (:8000)   (:8001)
                           │
                        MariaDB
                    (ghinel-docker)
```

Tous les services web partagent le réseau Docker `ghinel_network`.  
`kondo` (mobile) tourne sur `kondo_default` — le gateway y est aussi connecté.

### Rôle de chaque repo

| Repo | Rôle |
|------|------|
| `ghinel-website` | Site public, SSO BFF — les pages `/connexion` et `/inscription` sont **le point d'entrée d'authentification pour tout l'écosystème**. Pose les cookies `gh_access` / `gh_refresh` sur `.ghinel.com`. |
| `ghinel-auth-service` | Seule source d'identité. Signe les JWT RS256. Ne valide pas lui-même les tokens pour les autres services — ils vérifient localement via JWKS. |
| `papyrus-backend` | API livres / bibliothèque. Consomme les JWT émis par l'auth-service (vérification locale). |
| `nginx-gateway` | Reverse proxy avec SSL. Routage par vhost. Pas de logique métier. |
| `ghinel-docker` | MariaDB + volumes partagés (static files). C'est ce repo qui crée le réseau `ghinel_network`. |
| `kondo` | App mobile React Native / Expo. Consomme les mêmes API. |
| `Behanzin` | Projet éditorial — Dada of Dahomey. |
| `GriotBot` | Assistant IA griots béninois. |

---

## Mise en place locale

### Prérequis

- Docker 24+ et Docker Compose v2
- Node.js 20+ (pour `ghinel-website` en dev local sans Docker)
- Python 3.12+ (pour les services Django en dev local sans Docker)
- `gh` CLI configuré sur ton compte (pour cloner les repos privés)

### 1. Créer le réseau Docker

```bash
docker network create ghinel_network
```

C'est la première chose à faire — tous les services en ont besoin.

### 2. Cloner les repos

```bash
# Structure recommandée
mkdir ghinel && cd ghinel
gh repo clone Ghinel/ghinel-docker
gh repo clone Ghinel/ghinel-auth-service
gh repo clone Ghinel/papyrus-backend
gh repo clone Ghinel/ghinel-website
```

### 3. Démarrer l'infrastructure de base (BDD + nginx)

```bash
cd ghinel-docker
cp .env.example .env   # remplir les variables
docker compose up -d --build
```

### 4. Démarrer l'auth-service

```bash
cd ../ghinel-auth-service
cp .env.example .env   # remplir les variables
docker compose up -d --build
```

Swagger disponible sur `http://localhost:8000/swagger/` (uniquement si `DEBUG=True`).

### 5. Démarrer papyrus-backend

```bash
cd ../papyrus-backend
cp .env.example .env
docker compose up -d --build
```

Swagger : `http://localhost:8001/swagger/`

### 6. Démarrer le site web

```bash
cd ../ghinel-website
cp .env.example .env.local   # remplir les variables
docker compose up -d --build
# ou en dev natif :
npm install && npm run dev
```

Site : `http://localhost:3003`

---

## Variables d'environnement clés

### `ghinel-docker`

| Variable | Description |
|----------|-------------|
| `DB_ROOT_PASSWORD` | Mot de passe root MariaDB |
| `DB_USER` / `DB_PASSWORD` | Utilisateur des bases Ghinel |

### `ghinel-auth-service`

Voir `.env.example` dans le repo. Points importants :
- La clé privée RSA (`PRIVATE_KEY_PATH`) est générée une seule fois et ne doit jamais être committée.
- `GOOGLE_CLIENT_IDS` : liste des Client IDs Google autorisés (web, Android, iOS) — à compléter pour chaque nouveau client.
- `FRONTEND_VERIFY_EMAIL_URL` : URL de ton frontend pour la confirmation d'email.
- `CORS_ALLOWED_ORIGINS` : ajouter ton origine locale (ex. `http://localhost:3003`).

### `ghinel-website`

| Variable | Description |
|----------|-------------|
| `AUTH_API_URL` | URL serveur de l'auth-service (jamais exposée au navigateur) |
| `AUTH_COOKIE_DOMAIN` | Vide en dev, `.ghinel.com` en prod |
| `AUTH_ALLOWED_REDIRECT_HOSTS` | Hôtes autorisés pour `redirect_uri` (anti open-redirect) |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Client ID Google — **inliné au build**, nécessite `--build` si changé |

---

## Authentification — l'essentiel

Le flux complet est documenté dans [`ghinel-auth-service/INTEGRATION.md`](https://github.com/Ghinel/ghinel-auth-service/blob/main/INTEGRATION.md). En résumé :

**Pour un nouveau service qui consomme des tokens :**

1. Récupère la clé publique via JWKS : `GET /api/auth/.well-known/jwks.json`
2. Vérifie localement, algorithme `RS256`, force `token_type === 'access'`
3. Utilise `sub` (UUID) comme identifiant utilisateur — jamais l'email
4. Sur 401 → tente un refresh une fois, remplace les deux tokens (rotation activée)

**Pour ajouter un nouveau client Google OAuth :**
→ Communiquer le Client ID Google à l'équipe auth pour l'ajouter à `GOOGLE_CLIENT_IDS`.

---

## Commandes Docker utiles

```bash
# Voir les logs en live
docker compose logs -f

# Logs d'un seul service
docker compose logs -f ghinel-auth

# Redémarrer un service sans reconstruire
docker compose restart ghinel-auth

# Reconstruire et relancer
docker compose up -d --build

# Arrêter sans supprimer les données
docker compose down

# Arrêter et supprimer les volumes (⚠ perte de données)
docker compose down -v
```

---

## Conventions

- **Branches** : `main` est la branche de production. Travaille sur une branche feature et ouvre une PR.
- **Commits** : messages en français ou en anglais, impératif, pas de point final.
- **Secrets** : jamais dans git. Utiliser `.env` (ignoré) ou les secrets Docker / CI.
- **Identifiant utilisateur** : toujours `sub` (UUID) dans les services, jamais l'email.

---

## Liens utiles

- [Guide d'intégration auth](https://github.com/Ghinel/ghinel-auth-service/blob/main/INTEGRATION.md) — référence complète pour consommer l'auth-service
- [ghinel-docker README](https://github.com/Ghinel/ghinel-docker/blob/main/README.md) — déploiement VPS Contabo
- **contact@ghinel.io** — pour toute question d'accès ou de configuration
