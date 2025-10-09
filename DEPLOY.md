Déploiement en production (VPS + Docker)

Objectif: mettre le site en ligne sur neogend-frp.fr avec HTTPS. On y va pas à pas, très simplement, en suivant les étapes dans l’ordre. Si quelque chose échoue, on revient à l’étape juste avant.

## 0) Pré-requis (à préparer avant)

-   Tu as un serveur (VPS) accessible en SSH (Ubuntu/Debian conseillé)
-   Le domaine neogend-frp.fr pointe vers l’IP du VPS (DNS de type A). Attends la propagation DNS (ça peut prendre 5–30 min)
-   Les ports 80 (HTTP) et 443 (HTTPS) sont ouverts sur:
    -   le firewall du fournisseur (OVH/Scaleway/Azure/etc.)
    -   le firewall du VPS (ex: ufw). Exemple pour Ubuntu:

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 22
sudo ufw enable
```

-   Docker et Docker Compose sont installés sur le VPS
    -   Si besoin (Ubuntu):

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Déconnecte-toi/reconnecte-toi ensuite pour que le groupe docker soit pris en compte
```

## 1) Mettre le projet sur le VPS

Tu as deux options. Choisis UNE option.

Option A — Copier depuis ton PC Windows (PowerShell):

```powershell
# Remplace 1.2.3.4 par l’IP de ton VPS, et adapte le chemin si besoin
scp -r "D:\Dev\projetPerso\NeogendFRP\" root@1.2.3.4:/opt/neogend
```

Option B — Cloner un dépôt git directement sur le VPS:

```bash
ssh root@1.2.3.4
sudo mkdir -p /opt/neogend
cd /opt/neogend
git clone <URL_DU_DEPOT> .
```

Dans la suite, on suppose que les fichiers sont dans /opt/neogend sur le VPS.

## 2) Préparer le fichier d’environnement

Sur le VPS:

```bash
cd /opt/neogend
cp .env.prod .env
nano .env
```

Dans `.env`, remplis les valeurs importantes:

-   POSTGRES_PASSWORD = un bon mot de passe
-   SECRET_KEY = une longue chaîne aléatoire (clé JWT)
-   FRONTEND_ORIGINS = https://neogend-frp.fr
-   COOKIE_SECURE = true
-   COOKIE_SAMESITE = none
-   (facultatif) DEFAULT*ADMIN*\* si tu veux un admin créé automatiquement si aucun utilisateur n’existe
-   (facultatif) DOMAIN et CERTBOT_EMAIL si tu veux éviter de les taper à la main plus tard

Sauvegarde et ferme.

Astuce: tu peux vérifier rapidement que l’IP publique répond au ping DNS:

```bash
ping -c 1 neogend-frp.fr
```

## 3) Démarrer en HTTP (première mise en route)

On démarre l’infra en HTTP (port 80) pour permettre à Let’s Encrypt de vérifier le domaine.

Sur le VPS:

```bash
cd /opt/neogend
docker compose -f docker-compose.prod.yml --env-file .env up -d db backend frontend reverse-proxy
```

Vérifie que ça tourne:

```bash
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f reverse-proxy
```

Teste dans un navigateur: http://neogend-frp.fr doit afficher ton site (pas encore en HTTPS).

## 4) Émettre le certificat Let’s Encrypt (une seule fois)

Toujours sur le VPS (ou depuis ton PC si tu préfères, c’est pareil tant que ça exécute sur le VPS):

```bash
cd /opt/neogend
# Si DOMAIN et CERTBOT_EMAIL sont dans ton .env, tu peux juste lancer:
docker compose -f docker-compose.prod.yml run --rm certbot_init

# Sinon, passe-les en variables d’env à la volée:
docker compose -f docker-compose.prod.yml run --rm \
    -e DOMAIN=neogend-frp.fr \
    -e CERTBOT_EMAIL=votre@email \
    certbot_init
```

Si c’est bon, les certificats sont créés ici sur le VPS:
`/opt/neogend/data/certbot/conf/live/neogend-frp.fr/`

Si ça échoue:

-   Vérifie que neogend-frp.fr pointe bien vers l’IP du VPS
-   Vérifie que le port 80 est ouvert
-   Attends 5–10 minutes et réessaie

## 5) Basculer Nginx en HTTPS (activer 443)

Maintenant on passe en SSL. Deux mini-changements à faire dans `docker-compose.prod.yml` (service `reverse-proxy`):

1. Activer le port 443 (dé-commente la ligne):

```yaml
ports:
    - "80:80"
    - "443:443" # activer cette ligne
```

2. Utiliser la conf SSL au lieu de HTTP:

```yaml
volumes:
    - ./nginx/prod.ssl.conf:/etc/nginx/conf.d/default.conf:ro
    - ./data/certbot/www:/var/www/certbot
    - ./data/certbot/conf:/etc/letsencrypt
```

Puis redémarre uniquement le reverse proxy:

```bash
cd /opt/neogend
docker compose -f docker-compose.prod.yml up -d reverse-proxy
```

## 6) Lancer le renouvellement automatique des certificats

Toujours sur le VPS:

```bash
docker compose -f docker-compose.prod.yml up -d certbot
```

Ce service va tenter un `certbot renew` toutes les 12h.

## 7) Tester que tout est OK

-   Visite: http://neogend-frp.fr → doit rediriger vers https
-   Visite: https://neogend-frp.fr → le site doit s’afficher
-   API: https://neogend-frp.fr/api/health → doit renvoyer 200
-   Docs: https://neogend-frp.fr/api/docs → doit s’ouvrir

Tu peux aussi tester en ligne de commande:

```bash
curl -I http://neogend-frp.fr
curl -I https://neogend-frp.fr
curl -sS https://neogend-frp.fr/api/health
```

## 8) Opérations courantes (petit pense-bête)

-   Voir l’état:

```bash
docker compose -f docker-compose.prod.yml ps
```

-   Voir les logs:

```bash
docker compose -f docker-compose.prod.yml logs -f reverse-proxy
docker compose -f docker-compose.prod.yml logs -f backend
docker compose -f docker-compose.prod.yml logs -f db
```

-   Mettre à jour le code et redéployer (les migrations Alembic sont lancées automatiquement au démarrage du backend):

```bash
git pull            # si tu utilises git
docker compose -f docker-compose.prod.yml build --no-cache backend frontend
docker compose -f docker-compose.prod.yml up -d backend frontend
```

-   Sauvegarder la base Postgres (backup):

```bash
docker compose -f docker-compose.prod.yml exec -T db \
    pg_dump -U neogend_user neogend_db > backup_$(date +%F).sql
```

-   Restaurer une sauvegarde:

```bash
cat backup_YYYY-MM-DD.sql | docker compose -f docker-compose.prod.yml exec -T db \
    psql -U neogend_user -d neogend_db
```

## 9) Problèmes fréquents (et solutions simples)

-   Le certificat ne s’émet pas:

    -   DNS pas encore propagé → attendre 10–30 min
    -   Port 80 fermé → ouvrir sur le VPS et chez le fournisseur
    -   Mauvais domaine → vérifier `DOMAIN` et server_name dans `nginx/prod.http.conf`/`prod.ssl.conf`

-   Erreur 502 Bad Gateway:

    -   Le backend n’est pas démarré ou a crash → `docker compose logs -f backend`
    -   Le frontend n’est pas prêt → `docker compose logs -f frontend`
    -   502 avec Nginx et message `connect() failed (111: Connection refused) while connecting to upstream`:
        -   Cause fréquente: l’IP du conteneur `frontend`/`backend` a changé après un restart et Nginx pointe encore vers l’ancienne IP (DNS Docker « stale »).
        -   Correctif: les configs Nginx `nginx/prod.http.conf` et `nginx/prod.ssl.conf` intègrent désormais un resolver Docker (`resolver 127.0.0.11`) et utilisent des variables d’upstream dynamiques. Assurez-vous d’utiliser ces fichiers et redémarrez le reverse proxy:

        ```bash
        docker compose -f docker-compose.prod.yml up -d reverse-proxy
        # ou simplement recharger la conf sans couper:
        docker compose -f docker-compose.prod.yml exec reverse-proxy nginx -s reload
        ```
        -   Vérifiez aussi que `frontend` et `backend` sont `healthy`:
        ```bash
        docker compose -f docker-compose.prod.yml ps
        ```

-   Swagger (docs) ne charge pas:

    -   Vérifier https://neogend-frp.fr/api/docs et que `API_ROOT_PATH=/api` est bien pris en compte (c’est déjà configuré)

-   CORS / Cookies:
    -   FRONTEND_ORIGINS doit contenir exactement `https://neogend-frp.fr`
    -   Cookies: `COOKIE_SECURE=true` et `COOKIE_SAMESITE=none` (déjà dans `.env.prod`)

## 10) En résumé (vraiment très simple)

1. Copier le projet sur le VPS → /opt/neogend
2. Copier `.env.prod` → `.env` et remplir les secrets
3. Démarrer en HTTP: `docker compose up -d db backend frontend reverse-proxy`
4. Émettre les certs: `docker compose run --rm certbot_init`
5. Activer 443 + conf SSL, redémarrer `reverse-proxy`
6. Lancer `certbot` (renouvellement automatique)
7. Tester `/`, `/api/health`, `/api/docs`

Carré dans les carrés, rond dans les ronds 🙂
