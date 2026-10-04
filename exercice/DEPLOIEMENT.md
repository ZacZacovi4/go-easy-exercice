# Procédure de déploiement

> Vous pouvez suivre la procédure de [configuration](../docs/environnement-de-production/1-configuration.md), [d’installation](../docs/environnement-de-production/2-installation.md) et de [déploiement](../docs/deploiement/) présentée en cours, ou opter pour votre propre démarche.

Cette procédure déploie l'application GoEasy sur une VM Debian 13 vierge. Chaque étape est suivie de sa **vérification** (`Vérif`) : ne passer à l'étape suivante que si elle est concluante.

## Variables à remplacer

| Variable          | Signification                                                   |
| ----------------- | --------------------------------------------------------------- |
| `<user>`          | utilisateur applicatif non-privilégié créé sur la VM            |
| `<compte-github>` | compte GitHub propriétaire du dépôt, **en minuscules**          |
| `<depot-url>`     | URL HTTPS du dépôt GitHub (`https://github.com/<compte>/<repo>.git`) |
| `<repo>`          | nom du dépôt (= nom du dossier créé par `git clone`)            |
| `<domaine>`       | nom de domaine routé vers la VM (fourni par l'organisme)        |

## 0 - Prérequis sur le poste de développement

### 0.1 Dépôt GitHub

Créer un dépôt vide sur GitHub (sans README, sans `.gitignore`, sans licence), puis, dans le dossier du projet :

```bash
git init
git branch -M main
git add .
git commit -m "Initial commit"
git remote add origin <depot-url>
git push -u origin main
```

Vérif : tous les fichiers sont visibles sur GitHub, et `git status` indique `working tree clean`.

### 0.2 Pipeline CI/CD

Le push sur `main` déclenche le workflow `.github/workflows/ci-cd.yml` (build, tests, publication des images sur `ghcr.io`).

Vérif : onglet **Actions** du dépôt, le workflow `CI/CD` est terminé avec une coche verte.

### 0.3 Visibilité des packages

Pour chacun des deux packages (`go-easy-backend`, `go-easy-front-office`) : onglet **Packages** du compte GitHub, puis **Package settings**, **Change visibility**, **Public**.

Vérif (réalisée depuis la VM à l'étape 3.4) : `docker manifest inspect` retourne un JSON sans authentification.

## 1 - Connexion à la VM

Depuis l'extérieur du réseau de l'organisme : portail Guacamole `https://guacamole.stagiairesmns.fr/guacamole/`. Depuis le réseau de l'organisme : `ssh <utilisateur>@<ip>` (répondre `yes` à l'empreinte la première fois).

Vérif : une invite de commande `<utilisateur>@<nom-de-la-vm>:~$` s'affiche.

## 2 - Configuration du serveur

### 2.1 Locale `en_US.UTF-8`

```bash
sudo nano /etc/locale.gen
```

Décommenter la ligne `# en_US.UTF-8 UTF-8`, enregistrer (`CTRL+O`, `Entrée`, `CTRL+X`), puis :

```bash
sudo locale-gen
sudo update-locale LANG=en_US.UTF-8
exit
```

Se reconnecter à la VM.

Vérif :

```bash
locale -a | grep -i en_US
locale
```

`en_US.utf8` est listée et `locale` n'affiche aucun message `Cannot set LC_*`.

### 2.2 Fuseau horaire

```bash
sudo timedatectl set-timezone Europe/Paris
```

Vérif : `timedatectl` affiche `Time zone: Europe/Paris`.

### 2.3 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade
```

Vérif : la commande se termine sans erreur ; `sudo apt update` ne signale plus de mise à jour en attente après coup.

### 2.4 Mises à jour de sécurité automatiques

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Sélectionner `Yes`, puis `Entrée`.

Vérif : `systemctl status unattended-upgrades` indique le service `active`.

### 2.5 Utilisateur applicatif non-privilégié

```bash
sudo adduser <user>
sudo usermod -aG sudo <user>
```

Vérif : `getent passwd <user>` retourne une ligne, et `groups <user>` contient `sudo`.

### 2.6 Étapes facultatives de durcissement

Clé SSH (depuis le poste du développeur) :

```bash
ssh-keygen -t ed25519 -a 100
ssh-copy-id <nom-du-serveur-ou-du-projet>
```

Vérif : `ssh <nom-du-serveur-ou-du-projet>` se connecte sans mot de passe de compte.

`fail2ban` :

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Dans la section `[sshd]` déjà présente, ajouter `enabled = true`, puis :

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

Vérif : `sudo fail2ban-client status sshd` affiche `Status for the jail: sshd`.

Pare-feu `ufw` (toujours autoriser SSH avant d'activer) :

```bash
sudo apt install ufw
sudo ufw allow OpenSSH
sudo ufw enable
```

Vérif : `sudo ufw status` affiche `Status: active` avec la règle OpenSSH.

### 2.7 Docker

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Vérif intermédiaire : `cat /etc/apt/sources.list.d/docker.sources` affiche 6 lignes distinctes (un champ par ligne).

Vérif : `sudo systemctl status docker` indique `active (running)`, et `sudo docker run hello-world` affiche `Hello from Docker!`.

Utiliser Docker sans `sudo` :

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

Vérif : `docker run hello-world` (sans `sudo`) affiche `Hello from Docker!`.

### 2.8 Git

```bash
sudo apt install git
```

Vérif : `git --version`.

## 3 - Installation de l'application

### 3.1 Prendre l'utilisateur applicatif

```bash
sudo su <user>
cd
```

Vérif : `whoami` retourne `<user>` et `pwd` retourne `/home/<user>`.

### 3.2 Cloner le dépôt

```bash
git clone <depot-url>
cd <repo>
```

Vérif : `ls` affiche `backend`, `front-office`, `db`, `nginx`, `docs`, `docker-compose.yml`, `docker-compose.prod.yml`.

### 3.3 Variables d'environnement

```bash
cp backend/.env.example backend/.env
cp db/.env.example db/.env
nano backend/.env
nano db/.env
```

Dans `backend/.env` : renseigner `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME`, `JWT_SECRET`, `SESSION_SECRET`, `SEED_ADMIN_PASSWORD`, `SEED_TOURIST_PASSWORD`. Ne pas modifier `DB_HOST` ni `CORS_ORIGINS`. Générer chaque secret avec `openssl rand -base64 32` (deux valeurs distinctes pour `JWT_SECRET` et `SESSION_SECRET`).

Dans `db/.env` : renseigner `MARIADB_DATABASE`, `MARIADB_USER`, `MARIADB_PASSWORD`, `MARIADB_ROOT_PASSWORD`. Les trois premières valeurs doivent être **identiques** à `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`.

Enregistrer chaque fichier (`CTRL+O`, `Entrée`, `CTRL+X`). Ne jamais écrire de secret dans un fichier `.env.example` (versionné dans Git).

Vérif :

```bash
cat backend/.env
cat db/.env
diff backend/.env.example backend/.env
grep -nE '[[:space:]]+$' backend/.env db/.env
```

Les valeurs saisies sont présentes (plus de valeurs d'exemple), les valeurs communes sont identiques entre les deux fichiers, et le dernier `grep` n'affiche rien (aucun espace en fin de ligne).

### 3.4 Images Docker (`docker-compose.prod.yml`)

```bash
nano docker-compose.prod.yml
```

Sur les deux lignes `#ADAPTER UNIQUEMENT ICI`, remplacer `<compte-github>` par le compte GitHub **en minuscules** :

```yaml
image: ghcr.io/<compte-github>/go-easy-backend:latest
image: ghcr.io/<compte-github>/go-easy-front-office:latest
```

Vérif :

```bash
grep -n "ghcr.io" docker-compose.prod.yml
docker manifest inspect ghcr.io/<compte-github>/go-easy-backend:latest
docker manifest inspect ghcr.io/<compte-github>/go-easy-front-office:latest
```

Le `grep` affiche les deux images sans chevrons, et chaque `docker manifest inspect` retourne un JSON (image publique et accessible).

### 3.5 Configuration NGINX

```bash
nano nginx/nginx.conf
```

Remplacer le nom de domaine sur la ligne `#ADAPTER UNIQUEMENT ICI` :

```nginx
server_name <domaine>;#ADAPTER UNIQUEMENT ICI
```

Vérif : `grep -n "server_name" nginx/nginx.conf` affiche le domaine, sans chevrons.

### 3.6 Ports 80 et 443 libres

```bash
sudo ss -tlnp | grep -E ':80|:443'
```

Vérif : la commande n'affiche rien. Sinon, libérer les ports (par exemple `sudo systemctl stop apache2` puis `sudo systemctl disable apache2`).

### 3.7 Pare-feu (uniquement si `ufw` est installé)

```bash
sudo ufw allow 80,443/tcp
```

Vérif : `sudo ufw status` affiche la règle `80,443/tcp ALLOW`.

### 3.8 Démarrer les services

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Vérif : `docker compose -f docker-compose.yml -f docker-compose.prod.yml ps` affiche `db`, `backend`, `front-office`, `nginx` et `watchtower` à l'état `Up` (`db` et `watchtower` en `healthy`).

### 3.9 Initialiser la base de données

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec backend npm run seed
```

Vérif : `curl -s http://localhost/api/activities | head -c 400` retourne du JSON contenant des activités.

## 4 - Vérifications finales

Depuis la VM :

```bash
curl -i http://localhost/api/health
curl -i -H "Host: <domaine>" http://localhost/
```

Le premier retourne `200 OK` avec `"status":"healthy"` et `"environment":"production"` ; le second retourne la page HTML de l'application.

Depuis un navigateur :

- `https://<domaine>/` affiche le catalogue de l'application ;
- `https://<domaine>/api/health` retourne un état sain ;
- `https://<domaine>/admin` affiche l'authentification du back-office ;
- la connexion fonctionne avec le compte client et le compte admin créés par le seed (identifiants de `backend/.env`).

Contrôle que le domaine désigne bien sa propre VM : lancer en rapprochant les deux commandes `curl -sk https://<domaine>/api/health` et `curl -s http://localhost/api/health` ; les valeurs `uptime` doivent être quasi identiques.

## 5 - Mises à jour (déploiement d'un correctif)

1. Commiter et pousser les modifications sur `main` :

```bash
git add <fichiers>
git commit -m "<message>"
git push origin main
```

Vérif : onglet **Actions** du dépôt, le workflow est vert (les images sont reconstruites et publiées sur `ghcr.io`).

2. `watchtower` détecte les nouvelles images (interrogation toutes les 60 s) et recrée les conteneurs concernés.

Vérif :

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f watchtower
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
```

Les logs de `watchtower` mentionnent la mise à jour, et la colonne `CREATED` des conteneurs `backend` et `front-office` est récente.

3. Si `watchtower` ne met pas à jour, forcer manuellement :

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
