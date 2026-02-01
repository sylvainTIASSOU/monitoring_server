Voici la configuration complète révisée avec **Traefik à la place de Nginx**, ainsi que les optimisations nécessaires sur votre machine hôte pour un accès simplifié à votre VPS. J'ai restructuré le guide pour une intégration fluide de Traefik avec Docker, Let's Encrypt, et une gestion centralisée des services.

---

## 🔧 Configuration Optimisée : Traefik + Accès Hôte Simplifié

### 📌 Principales modifications apportées :
- Remplacement complet de Nginx par **Traefik v3** (reverse proxy moderne avec dashboard intégré)
- Configuration Docker-native avec providers dynamiques
- Gestion automatique des certificats TLS via Let's Encrypt
- Sécurisation du dashboard Traefik avec authentification basique
- Optimisations SSH côté machine hôte pour accès simplifié
- Intégration avec vos projets existants (FastAPI, Flutter, etc.)

---

## 1. Initial System Setup *(inchangé mais sans Nginx)*

```bash
# Création utilisateur sécurisé
sudo adduser satya
sudo usermod -aG sudo satya
su - satya

# Mise à jour système
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git htop neofetch net-tools figlet lolcat -y

# MOTD personnalisé (inchangé)
sudo nano /etc/update-motd.d/00-custom
```
```bash
#!/bin/bash
echo
figlet "Traefik Server" | lolcat
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Utilisateur: $(whoami)"
echo "Hôte: $(hostname) @ $(hostname -I | awk '{print $1}')"
echo "Système: $(lsb_release -ds)"
echo "Kernel: $(uname -r)"
echo
echo "Mémoire: $(free -h | awk '/^Mem:/ {print $3 "/" $2 " (" $3/$2*100 "%)"}')"
echo "Disque: $(df -h / | awk 'NR==2 {print $3 "/" $2 " (" $5 ")"}')"
echo "Charge: $(cat /proc/loadavg | awk '{print $1, $2, $3}')"
echo
[ -f /var/run/reboot-required ] && echo "⚠️  Redémarrage requis !" | lolcat -p 0.3
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Traefik Dashboard: https://traefik.your-domain.com"
echo "Portainer: https://portainer.your-domain.com"
```
```bash
sudo chmod +x /etc/update-motd.d/00-custom
```

---

## 2. Sécurité SSH *(renforcée pour Traefik)*

```bash
sudo nano /etc/ssh/sshd_config
```
```ini
Port 2222  # Changer le port par défaut pour réduire les scans
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
AllowUsers satya
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```
```bash
sudo systemctl restart sshd
```

> ⚠️ **Important** : Avant de redémarrer SSH, testez la connexion dans un nouveau terminal pour éviter de vous verrouiller !

---

## 3. Configuration Machine Hôte (Votre PC Local)

Créez un fichier `~/.ssh/config` sur votre machine locale pour un accès simplifié :

```bash
# ~/.ssh/config
Host vps-togo
    HostName VOTRE_IP_SERVEUR
    User satya
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_togo  # Votre clé privée
    ServerAliveInterval 60
    ForwardAgent yes
    Compression yes
    LocalForward 8080 localhost:8080    # Pour accéder à Traefik dashboard localement

Host *.vps-togo
    User satya
    ProxyJump vps-togo
```

**Raccourcis utiles** (`~/.bashrc` ou `~/.zshrc` sur votre machine locale) :
```bash
# Accès rapide au serveur
alias vps='ssh vps-togo'

# Accès aux logs Traefik
alias traefik-logs='ssh vps-togo "docker logs traefik -f"'

# Redémarrage sécurisé de Traefik
alias traefik-restart='ssh vps-togo "cd /opt/traefik && docker compose restart traefik"'

# Tunnel sécurisé vers le dashboard (accès via http://localhost:8080)
alias traefik-tunnel='ssh -L 8080:localhost:8080 vps-togo -N'
```

```bash
source ~/.bashrc  # ou source ~/.zshrc
```

---

## 4. Docker Setup *(positionné avant Traefik)*

```bash
# Installation Docker (inchangée)
sudo apt update
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker satya
newgrp docker  # Appliquer le groupe immédiatement
```

---

## 5. Traefik v3 Setup *(remplace Nginx)*

### Structure des dossiers
```bash
sudo mkdir -p /opt/traefik/{rules,acme,logs}
sudo chown -R satya:satya /opt/traefik
```

### docker-compose.yml
```yaml
# /opt/traefik/docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"    # HTTP
      - "443:443"  # HTTPS
      - "8080:8080" # Dashboard (bloqué par défaut via middleware)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./acme:/acme:rw
      - ./logs:/logs:rw
      - ./rules:/rules:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.votre-domaine.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=${TRAEFIK_USER}:${TRAEFIK_HASH}"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

### traefik.yml
```yaml
# /opt/traefik/traefik.yml
global:
  checkNewVersion: true
  sendAnonymousUsage: false

api:
  dashboard: true
  debug: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: proxy
  file:
    directory: /rules
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: votre@email.com
      storage: /acme/acme.json
      keyType: EC384
      httpChallenge:
        entryPoint: web

log:
  level: WARN
  filePath: /logs/traefik.log
  format: json

accessLog:
  filePath: /logs/access.log
  format: json
  bufferingSize: 100
```

### Génération du mot de passe pour le dashboard
```bash
# Sur votre machine locale ou serveur
echo $(htpasswd -nb admin VOTRE_MOT_DE_PASSE_SECURE | openssl passwd -apr1 -stdin)
# Résultat à placer dans .env : TRAEFIK_HASH=...
```

### .env (à créer dans /opt/traefik/.env)
```ini
TRAEFIK_USER=admin
TRAEFIK_HASH=$apr1$xyz...  # Résultat de la commande ci-dessus
```

### Création du réseau Docker
```bash
docker network create proxy
```

### Permissions ACME
```bash
touch /opt/traefik/acme/acme.json
chmod 600 /opt/traefik/acme/acme.json
```

### Démarrage
```bash
cd /opt/traefik
docker compose up -d
```

---

## 6. Configuration Pare-feu (UFW)
```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH personnalisé'
sudo ufw allow 80/tcp comment 'HTTP Traefik'
sudo ufw allow 443/tcp comment 'HTTPS Traefik'
sudo ufw enable
```

---

## 7. Exemple d'utilisation avec vos projets

### Pour votre API FastAPI (docker-compose.yml)
```yaml
services:
  api:
    image: votre-api:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.votre-domaine.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.services.api.loadbalancer.server.port=8000"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

### Pour Portainer (gestion Docker)
```bash
docker run -d \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -l "traefik.enable=true" \
  -l "traefik.http.routers.portainer.rule=Host(`portainer.votre-domaine.com`)" \
  -l "traefik.http.routers.portainer.entrypoints=websecure" \
  -l "traefik.http.routers.portainer.tls.certresolver=letsencrypt" \
  portainer/portainer-ce:latest
```

---

## 8. Surveillance & Maintenance

### Script de surveillance mémoire adapté pour Traefik
```bash
sudo nano /usr/local/bin/memcheck.sh
```
```bash
#!/bin/bash
THRESHOLD=85
MEMORY_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')

if [ $MEMORY_USAGE -gt $THRESHOLD ]; then
    echo "[$(date)] Mémoire critique ($MEMORY_USAGE%) - Nettoyage" >> /var/log/memcheck.log
    sync; echo 3 > /proc/sys/vm/drop_caches
    docker restart traefik 2>/dev/null
fi
```
```bash
sudo chmod +x /usr/local/bin/memcheck.sh
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/memcheck.sh") | crontab -
```

### Commandes utiles
```bash
# Vérifier les certificats Let's Encrypt
docker exec traefik cat /acme/acme.json | jq '.letsencrypt.Certificates'

# Logs en temps réel
docker logs -f traefik --tail 100

# Redémarrage sécurisé
cd /opt/traefik && docker compose restart traefik
```

---

## ✅ Points clés à retenir

| Fonctionnalité          | Accès                                  | Notes                                  |
|-------------------------|----------------------------------------|----------------------------------------|
| Dashboard Traefik       | `https://traefik.votre-domaine.com`    | Authentification basique requise       |
| Vos APIs/services       | `https://votre-service.domaine.com`    | Certificats TLS automatiques           |
| Portainer               | `https://portainer.votre-domaine.com`  | Gestion visuelle de Docker             |
| Connexion SSH           | `ssh vps-togo`                         | Depuis votre machine hôte configurée   |
| Tunnel dashboard local  | `traefik-tunnel` + `http://localhost:8080` | Pour accès sécurisé sans DNS        |

---

## 🔐 Bonnes pratiques de sécurité

1. **Ne jamais exposer le port 8080 publiquement** – utilisez toujours le middleware d'authentification
2. **Renouvelez régulièrement les mots de passe** du dashboard Traefik
3. **Surveillez les logs** : `docker logs traefik --since 1h | grep -i error`
4. **Backup ACME** : sauvegardez `/opt/traefik/acme/acme.json` pour éviter les limites Let's Encrypt
5. **Mettez à jour Traefik** régulièrement : `docker compose pull && docker compose up -d`

Cette configuration vous donne une stack moderne, sécurisée et évolutive, parfaitement adaptée à vos projets TissuMode, Le Coin Étude et autres applications nécessitant un reverse proxy intelligent avec gestion TLS automatique. Traefik s'intègre naturellement avec Docker Compose – idéal pour votre écosystème technique existant.