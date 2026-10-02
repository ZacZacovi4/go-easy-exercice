# Mémo d'examen — Déploiement sécurisé d'une application sur VM vierge

> Document personnel de révision, pensé pour être utilisable **sans IA**, avec accès à Internet et à ta propre documentation.
> Il reste volontairement générique : il ne liste pas les failles précises de l'exercice d'entraînement (GoEasy), mais des catégories et réflexes transposables à n'importe quelle application le jour de l'épreuve.

---

## 1. Méthodologie générale du jour J

1. **Prendre connaissance du projet** : README, schéma d'architecture, stack technique, variables d'environnement, routes exposées. Ne pas commencer à taper des commandes avant d'avoir compris ce que fait l'application.
2. **Faire tourner l'appli en local** (Docker Compose) pour l'explorer en conditions réelles avant de la déployer.
3. **Rédiger/adapter la procédure de déploiement** au fur et à mesure (ne pas attendre la fin) — ça sert aussi de pense-bête si une étape plante et doit être refaite.
4. **Déployer une première fois tel quel**, vérifier que tout fonctionne (health check, pages, API).
5. **Auditer le code avec la checklist ci-dessous**, corriger, documenter dans un changelog/rapport de sécurité.
6. **Redéployer le correctif** et re-vérifier.

Règle d'or à chaque étape : comprendre *pourquoi* une commande/config existe avant de la copier-coller. Une doc fournie peut contenir des erreurs (coquilles, commandes invalides) — toujours vérifier le résultat plutôt que supposer que "ça a marché parce que je l'ai tapé".

---

## 2. Checklist sécurité générique (à parcourir sur N'IMPORTE QUELLE appli)

Pour chaque catégorie : ce qu'il faut tester, pourquoi, et le principe de correction (pas une solution clé en main, le code change à chaque appli).

### Contrôle d'accès
- Une route qui agit sur "ma" ressource (ma commande, ma réservation, mon profil...) utilise-t-elle bien l'identité extraite du token/session, ou fait-elle confiance à un paramètre fourni par le client (`?userId=`, body, etc.) ?
- Une action d'écriture/suppression sur une ressource d'un autre utilisateur est-elle bloquée (IDOR) ?
- Les routes d'administration vérifient-elles le **rôle**, ou seulement le fait d'être authentifié (authn ≠ authz) ?
- Principe : toujours revérifier côté serveur l'identité ET les droits, jamais côté client seul.

### Injections (SQL, commande, etc.)
- Chercher les endroits où une entrée utilisateur est concaténée directement dans une requête/commande plutôt que passée en paramètre lié (requêtes préparées, query builder).
- Tester avec des caractères spéciaux (`'`, `--`, `;`, etc.) sur les champs de recherche/filtre.

### Validation des entrées côté serveur
- Prix, quantités, dates, rôles, statuts : sont-ils recalculés/revérifiés côté serveur, ou le serveur fait-il confiance à ce qu'envoie le client ?
- Un champ optionnel du formulaire peut-il donner plus de droits que prévu (ex: un rôle envoyé dans un payload d'inscription) ?

### Affichage des données utilisateur (XSS)
- Les champs saisis par les utilisateurs (nom, titre, commentaire...) sont-ils échappés avant d'être affichés dans les vues (back-office notamment, souvent moins testé que le front public) ?
- Vérifier les mécanismes de templating : certains ont une syntaxe "échappée" par défaut et une syntaxe "brute" à ne pas utiliser sur de la donnée utilisateur.

### Réponses API et fuite d'information
- Les réponses JSON contiennent-elles des champs sensibles qui ne devraient jamais sortir (mots de passe/hash, tokens internes, secrets) ?
- Les erreurs serveur renvoient-elles des détails internes (stack trace, chemins de fichiers) au client, surtout en production ?

### Configuration CORS
- `Access-Control-Allow-Origin` est-il restreint à une liste précise de domaines, ou à `*` (surtout combiné avec des cookies/credentials, ce qui est dangereux) ?
- Une variable d'environnement prévue pour ça est-elle vraiment utilisée dans le code, ou ignorée ?

### Authentification / tokens
- Algorithme de signature JWT restreint explicitement (éviter la confusion d'algorithme) ?
- Expiration courte + mécanisme de révocation (ex: version de token en base) ?
- Mots de passe hashés avec un algorithme adapté (bcrypt/argon2), jamais en clair ni avec un hash rapide (MD5/SHA1 seul).

### Secrets et configuration
- Secrets (JWT, session, DB) suffisamment longs/aléatoires, jamais commités en clair dans le dépôt.
- `.gitignore` couvre bien tous les fichiers `.env`.
- Variables de prod validées au démarrage (erreur explicite si secret trop court/absent).

### Concurrence / race conditions
- Une ressource à capacité limitée (stock, places, créneaux) : la vérification de disponibilité et l'écriture sont-elles protégées par une transaction/un verrou, ou vulnérables à deux requêtes simultanées ?

### Dépendances tierces
- Lancer l'équivalent d'un audit de dépendances (`npm audit` ou autre selon la stack) sur chaque sous-projet, et vérifier les CVEs réellement exploitables dans le contexte de l'appli (ne pas changer de version aveuglément si l'alerte ne concerne pas un chemin de code utilisé).

### Durcissement serveur (souvent déjà fourni, mais à savoir expliquer)
- Compte non-root dédié à l'application.
- Connexion SSH par clé plutôt que mot de passe.
- `fail2ban` sur SSH (anti brute-force).
- Pare-feu n'ouvrant que le strict nécessaire (SSH, puis 80/443).
- Mises à jour de sécurité automatiques.

### Exposition réseau des conteneurs
- Seuls les services qui doivent être atteints depuis l'extérieur publient un port sur l'hôte (ex: reverse proxy). Une base de données ne devrait jamais être exposée publiquement.

---

## 3. Procédure générique de déploiement sur VM vierge (grandes étapes + pourquoi)

1. **Connexion initiale** : accepter l'empreinte SSH (TOFU — on fait confiance au serveur la première fois, l'empreinte est ensuite épinglée dans `known_hosts`).
2. **Locale** (`en_US.UTF-8`) : évite des erreurs d'encodage sur les caractères spéciaux dans les logs/outils.
3. **Fuseau horaire** : cohérence des horodatages (logs, dates en base, certificats).
4. **Mise à jour système** (`apt update && apt upgrade`) : partir des derniers correctifs connus.
5. **Mises à jour automatiques de sécurité** (`unattended-upgrades`) : patcher même en l'absence de l'administrateur.
6. **Utilisateur non-privilégié dédié** : principe de moindre privilège, l'appli ne tourne pas avec un compte ayant tous les droits.
7. **Clé SSH** (si demandé) : remplace l'authentification par mot de passe, plus résistante au brute-force.
8. **`fail2ban`** (si demandé) : bannit les IP après plusieurs échecs de connexion.
9. **Pare-feu `ufw`** (si demandé) : ferme tout sauf SSH (puis 80/443 une fois l'appli exposée). Toujours autoriser SSH *avant* d'activer le pare-feu, sous peine de se couper l'accès.
10. **Docker** : moteur de conteneurisation. Ajouter l'utilisateur au groupe `docker` pour éviter `sudo` à chaque commande.
11. **Git** : nécessaire pour cloner le dépôt de l'application.
12. **Cloner le dépôt**, configurer les variables d'environnement (`.env`), premier lancement (`docker compose up --build`), seed de données si fourni.
13. **Vérification** : health check, accès aux différents services (front, API, back-office), logs sans erreur.
14. **Déploiement en production** : build/publication des images (CI/CD) → la VM va chercher (`pull`) les images plutôt que l'inverse (modèle pull, aucun identifiant d'accès à la VM côté CI).

---

## 4. Pièges rencontrés en entraînement (à ne pas refaire)

- **Sauvegarder un fichier dans `nano`** : `CTRL+X` pour quitter, puis la touche **Y** (lettre seule, pas `Ctrl+Y` — qui fait autre chose dans nano), puis **Entrée** pour confirmer le nom de fichier affiché.
- **Blocs de commandes multi-lignes (heredoc `<<EOF ... EOF`)** : si le copier/coller dans un terminal web (type Guacamole) ne préserve pas les retours à la ligne, tout le contenu se retrouve sur une seule ligne et casse le fichier généré (ex: fichier de sources `apt` illisible). **Toujours rouvrir le fichier après coup** (`cat` ou `nano`) pour vérifier qu'il a la forme attendue, ligne par ligne.
- **Accès réseau à la VM** : un accès SSH direct peut être restreint au réseau interne de l'organisme ; depuis l'extérieur (chez soi), passer par le portail web fourni (ex: Guacamole) qui fait le lien en interne.
- **Ne jamais faire confiance aveuglément à une doc fournie** : vérifier chaque commande avant de l'exécuter, surtout si le résultat ne correspond pas à ce qui est annoncé — une doc peut contenir une coquille (commande qui n'existe pas, etc.).
- **Exécuter les commandes une par une plutôt qu'en bloc** quand la doc le recommande : ça permet de repérer immédiatement laquelle échoue.

---

## 5. Aide-mémoire commandes

### Docker / Compose
```bash
docker compose up --build -d                  # démarrer (reconstruire les images si besoin)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up --build -d
docker compose ps                             # état des conteneurs
docker compose logs -f <service>              # logs en continu d'un service
docker compose exec <service> <commande>      # exécuter une commande dans un conteneur
docker compose down                           # arrêter et supprimer les conteneurs
docker run hello-world                        # tester que Docker fonctionne
sudo systemctl status docker                  # état du service Docker
```

### systemctl (services en général)
```bash
sudo systemctl status <service>
sudo systemctl restart <service>
sudo systemctl enable <service>     # démarrage automatique au boot
sudo systemctl enable --now <service>
```

### Pare-feu ufw
```bash
sudo ufw allow OpenSSH        # autoriser SSH AVANT d'activer le pare-feu
sudo ufw allow 80,443/tcp     # une fois l'appli exposée en HTTP/HTTPS
sudo ufw enable
sudo ufw status
```

### fail2ban
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd      # état de la prison SSH
sudo grep -n '^\[sshd\]' /etc/fail2ban/jail.local   # repérer les doublons de section
```

### Git
```bash
git init
git remote add origin <url>
git add <fichiers>
git commit -m "message"
git push -u origin main
git clone <url>
```

### nano
```text
CTRL + X        # quitter
Y               # confirmer la sauvegarde (lettre seule)
Entrée          # valider le nom de fichier proposé
CTRL + O        # sauvegarder sans quitter (Write Out)
```

### Locale / fuseau horaire
```bash
sudo nano /etc/locale.gen       # décommenter la ligne voulue (ex: en_US.UTF-8 UTF-8)
sudo locale-gen
sudo update-locale LANG=en_US.UTF-8
sudo timedatectl set-timezone Europe/Paris
timedatectl                     # vérifier
```

### Audit de dépendances (exemple Node/npm)
```bash
npm audit
npm audit fix
```

### Nginx (vérification rapide de config)
```bash
sudo nginx -t                   # teste la syntaxe de la config
sudo systemctl reload nginx     # recharge sans coupure
```

---

*Document à compléter au fil de l'entraînement — ajoute tes propres pièges rencontrés et commandes utiles au fur et à mesure.*
