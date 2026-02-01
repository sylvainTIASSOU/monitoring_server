# Configuration Serveur VPS avec Traefik et Configuration Hôte

## 1. Configuration Initiale du Système

### Création d'utilisateur et SSH
```bash
# Créer l'utilisateur
sudo adduser satya
sudo usermod -aG sudo satya
su - satya

# Générer une clé SSH sur la machine hôte (LOCAL)
ssh-keygen -t ed25519 -C "your@email.com"

# Copier la clé vers le VPS
ssh-copy-id satya@server_ip

# Sur le VPS, configurer les permissions SSH
mkdir -p ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Configuration SSH Renforcée
```bash
sudo nano /etc/ssh/sshd_config
```
```bash
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers satya
```

### Mises à jour système
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git htop neofetch net-tools ufw fail2ban -y
```

## 2. Configuration Traefik (Remplacent Nginx)

### Installation Docker (prérequis)
```bash
# Installer Docker
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker $USER
```

### Configuration Traefik avec Docker Compose
```bash
# Créer la structure de répertoires
mkdir -p ~/traefik/{config,certificates}
cd ~/traefik

# Créer le fichier docker-compose.yml
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - web
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard (protégé)
    environment:
      - CF_API_EMAIL=${CF_API_EMAIL}
      - CF_API_KEY=${CF_API_KEY}
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/traefik.yml:ro
      - ./config/dynamic.yml:/dynamic.yml:ro
      - ./certificates:/certificates
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.middlewares.traefik-auth.basicauth.users=${TRAEFIK_USER}"
      - "traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.traefik.middlewares=traefik-https-redirect"
      - "traefik.http.routers.traefik-secure.entrypoints=https"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.traefik-secure.middlewares=traefik-auth"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=cloudflare"
      - "traefik.http.routers.traefik-secure.service=api@internal"

  # Exemple: Service Whoami
  whoami:
    image: traefik/whoami
    container_name: whoami
    restart: unless-stopped
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.${DOMAIN}`)"
      - "traefik.http.routers.whoami.entrypoints=https"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=cloudflare"

networks:
  web:
    external: true
```

### Configuration Traefik
```bash
# Créer le fichier de configuration principal
nano ~/traefik/config/traefik.yml
```

```yaml
api:
  dashboard: true
  debug: true

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
          permanent: true
  https:
    address: ":443"

serversTransport:
  insecureSkipVerify: true

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: web
  file:
    filename: /dynamic.yml
    watch: true

certificatesResolvers:
  cloudflare:
    acme:
      email: ${CF_API_EMAIL}
      storage: /certificates/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
```

### Configuration Dynamique
```bash
nano ~/traefik/config/dynamic.yml
```

```yaml
http:
  middlewares:
    # Middleware de sécurité
    security-headers:
      headers:
        customFrameOptionsValue: "SAMEORIGIN"
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "strict-origin-when-cross-origin"
        permissionsPolicy: "camera=(), microphone=(), geolocation=()"
        customResponseHeaders:
          X-Robots-Tag: "none"
    
    # Rate limiting
    ratelimit:
      rateLimit:
        average: 100
        burst: 50
    
    # Compression
    compress:
      compress: {}

tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
        - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
        - "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"
```

### Fichier d'environnement
```bash
nano ~/traefik/.env
```

```bash
# Variables d'environnement Traefik
DOMAIN=votre-domaine.com
CF_API_EMAIL=votre-email@domaine.com
CF_API_KEY=votre-api-key-cloudflare

# Générer le mot de passe pour le dashboard: 
# echo $(htpasswd -nb admin votre-mot-de-passe) | sed -e s/\\$/\\$\\$/g
TRAEFIK_USER=admin:$$apr1$$xxxxxxxx$$yyyyyyyyyyyyyyyyyyyyyyyy
```

### Créer le réseau Docker
```bash
docker network create web
```

### Lancer Traefik
```bash
cd ~/traefik
docker-compose up -d
```

## 3. Configuration Machine Hôte (LOCAL)

### Configuration SSH pour accès facile
```bash
# Sur votre machine locale, éditer ~/.ssh/config
nano ~/.ssh/config
```

Ajouter:
```bash
# Configuration VPS
Host vps
    HostName votre-ip-serveur
    User satya
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
    TCPKeepAlive yes
    Compression yes
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 1h
    
# Tunnel pour services spécifiques
Host vps-tunnel-*
    HostName votre-ip-serveur
    User satya
    IdentityFile ~/.ssh/id_ed25519
    LocalForward 127.0.0.1:8888 127.0.0.1:80
    
Host vps-traefik
    HostName votre-ip-serveur
    User satya
    IdentityFile ~/.ssh/id_ed25519
    LocalForward 127.0.0.1:8080 127.0.0.1:8080
```

### Script d'accès rapide
```bash
# Créer un script sur votre machine locale
nano ~/bin/vps-connect
```

```bash
#!/bin/bash

# Script pour gérer la connexion au VPS
VPS_HOST="votre-ip-serveur"
VPS_USER="satya"
SSH_KEY="~/.ssh/id_ed25519"

case "$1" in
    connect)
        ssh -i $SSH_KEY $VPS_USER@$VPS_HOST
        ;;
    traefik)
        ssh -i $SSH_KEY -L 8080:localhost:8080 $VPS_USER@$VPS_HOST
        ;;
    sync)
        rsync -avz -e "ssh -i $SSH_KEY" ~/projects/ $VPS_USER@$VPS_HOST:~/projects/
        ;;
    backup)
        rsync -avz -e "ssh -i $SSH_KEY" $VPS_USER@$VPS_HOST:~/backups/ ~/backups/vps/
        ;;
    docker-logs)
        ssh -i $SSH_KEY $VPS_USER@$VPS_HOST "docker logs -f $2"
        ;;
    status)
        ssh -i $SSH_KEY $VPS_USER@$VPS_HOST "docker ps && echo '---' && df -h && echo '---' && free -h"
        ;;
    *)
        echo "Usage: $0 {connect|traefik|sync|backup|docker-logs|status}"
        exit 1
        ;;
esac
```

```bash
# Rendre le script exécutable
chmod +x ~/bin/vps-connect

# Ajouter un alias dans ~/.bashrc ou ~/.zshrc
echo "alias vps='~/bin/vps-connect'" >> ~/.bashrc
source ~/.bashrc
```

### Configuration du fichier hosts local
```bash
# Sur votre machine locale, éditer /etc/hosts
sudo nano /etc/hosts
```

Ajouter:
```bash
# VPS Services
votre-ip-serveur    traefik.votre-domaine.com
votre-ip-serveur    whoami.votre-domaine.com
votre-ip-serveur    portainer.votre-domaine.com
```

### Script de monitoring local
```bash
nano ~/bin/vps-monitor
```

```bash
#!/bin/bash

# Monitorer le VPS depuis la machine locale
VPS_HOST="votre-ip-serveur"
VPS_USER="satya"

echo "=== VPS Status Monitor ==="
echo "Last checked: $(date)"
echo ""

# Vérifier la connectivité
if ping -c 1 $VPS_HOST &> /dev/null; then
    echo "✅ VPS is reachable"
    
    # Récupérer les stats via SSH
    ssh $VPS_USER@$VPS_HOST << 'EOF'
    echo "=== System Information ==="
    echo "Uptime: $(uptime -p)"
    echo "Load: $(cat /proc/loadavg | awk '{print $1, $2, $3}')"
    echo ""
    
    echo "=== Memory Usage ==="
    free -h | awk 'NR==2{printf "Used: %s/%s (%.2f%%)\n", $3, $2, $3/$2*100}'
    echo ""
    
    echo "=== Disk Usage ==="
    df -h / | awk 'NR==2{printf "Used: %s/%s (%.2f%%)\n", $3, $2, $3/$2*100}'
    echo ""
    
    echo "=== Docker Containers ==="
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    echo ""
    
    echo "=== Traefik Services ==="
    curl -s http://localhost:8080/api/http/routers | jq -r '.[].rule' 2>/dev/null || echo "Cannot access Traefik API"
EOF
else
    echo "❌ VPS is not reachable"
fi
```

## 4. Configuration Sécurité et Firewall

### Configuration UFW
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser SSH
sudo ufw allow 22/tcp

# Autoriser Traefik (HTTP/HTTPS)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Ne pas ouvrir le port du dashboard publiquement
# Il sera accessible via SSH tunnel seulement

sudo ufw enable
sudo ufw status verbose
```

### Fail2ban Configuration
```bash
sudo nano /etc/fail2ban/jail.local
```

```bash
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3

[traefik-auth]
enabled = true
port = http,https
filter = traefik-auth
logpath = /var/log/traefik/access.log
maxretry = 10
```

## 5. Scripts d'automatisation

### Script de déploiement automatique
```bash
nano ~/scripts/deploy-app.sh
```

```bash
#!/bin/bash

# Script pour déployer une application sur le VPS
APP_NAME=$1
DOMAIN=$2

if [ -z "$APP_NAME" ] || [ -z "$DOMAIN" ]; then
    echo "Usage: $0 <app_name> <domain>"
    exit 1
fi

# Créer la configuration Docker Compose
cat > ~/apps/$APP_NAME/docker-compose.yml << EOF
version: '3.8'

services:
  $APP_NAME:
    image: your-image/$APP_NAME
    container_name: $APP_NAME
    restart: unless-stopped
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.$APP_NAME.rule=Host(\`$DOMAIN\`)"
      - "traefik.http.routers.$APP_NAME.entrypoints=https"
      - "traefik.http.routers.$APP_NAME.tls=true"
      - "traefik.http.routers.$APP_NAME.tls.certresolver=cloudflare"
      - "traefik.http.services.$APP_NAME.loadbalancer.server.port=3000"
    environment:
      - NODE_ENV=production
    volumes:
      - ./data:/app/data

networks:
  web:
    external: true
EOF

# Déployer l'application
cd ~/apps/$APP_NAME
docker-compose up -d

echo "✅ Application $APP_NAME déployée sur https://$DOMAIN"
```

## 6. Configuration Backup Automatique

```bash
nano ~/scripts/backup.sh
```

```bash
#!/bin/bash

# Script de backup du VPS
BACKUP_DIR="/home/satya/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Créer le dossier de backup
mkdir -p $BACKUP_DIR/$DATE

# Backup des configurations Docker
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $BACKUP_DIR/$DATE:/backup alpine tar czf /backup/docker-configs.tar.gz /etc/docker

# Backup des volumes Docker
docker ps -q | while read CONTAINER; do
    NAME=$(docker inspect --format='{{.Name}}' $CONTAINER | sed 's/\///')
    docker run --rm --volumes-from $CONTAINER -v $BACKUP_DIR/$DATE:/backup alpine tar czf /backup/$NAME-volumes.tar.gz /config /data 2>/dev/null || true
done

# Backup des configurations Traefik
tar czf $BACKUP_DIR/$DATE/traefik-config.tar.gz ~/traefik

# Synchroniser avec le stockage externe (exemple: AWS S3)
# aws s3 sync $BACKUP_DIR/$DATE s3://your-backup-bucket/vps/$DATE/

echo "✅ Backup complet créé dans $BACKUP_DIR/$DATE"
```

## 7. Accès aux Services

### Pour accéder au dashboard Traefik depuis votre machine locale:
```bash
# Sur votre machine locale, créer un tunnel SSH
ssh -L 8080:localhost:8080 vps

# Maintenant, accédez à:
# http://localhost:8080
# Identifiants: admin / votre-mot-de-passe
```

### Commandes utiles sur le VPS:
```bash
# Voir les logs Traefik
docker logs -f traefik

# Voir les services actifs
curl http://localhost:8080/api/http/routers | jq .

# Redémarrer Traefik
cd ~/traefik && docker-compose restart

# Mettre à jour tous les conteneurs
docker-compose pull && docker-compose up -d
```

## 8. Configuration du Monitoring

### Installer cAdvisor pour monitoring Docker
```bash
nano ~/monitoring/docker-compose.yml
```

```yaml
version: '3.8'

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    networks:
      - web
    ports:
      - "8081:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.cadvisor.rule=Host(`cadvisor.${DOMAIN}`)"
      - "traefik.http.routers.cadvisor.entrypoints=https"
      - "traefik.http.routers.cadvisor.tls=true"
      - "traefik.http.routers.cadvisor.tls.certresolver=cloudflare"
      - "traefik.http.middlewares.cadvisor-auth.basicauth.users=${CADVISOR_USER}"

networks:
  web:
    external: true
```

## Résumé des Accès

### Depuis votre machine locale:
1. **Connexion SSH**: `ssh vps` ou `vps connect`
2. **Dashboard Traefik**: `vps traefik` puis http://localhost:8080
3. **Monitoring**: `vps-monitor`
4. **Fichiers**: `vps sync` et `vps backup`

### Services exposés:
1. **Traefik Dashboard**: https://traefik.votre-domaine.com (protégé)
2. **Applications**: Configurées automatiquement avec Let's Encrypt
3. **Monitoring**: https://cadvisor.votre-domaine.com

### Avantages de cette configuration:
- ✅ Traefik gère automatiquement les certificats SSL
- ✅ Configuration déclarative avec Docker Compose
- ✅ Dashboard protégé accessible uniquement via tunnel SSH
- ✅ Automatisation complète du déploiement
- ✅ Monitoring intégré
- ✅ Backup automatique
- ✅ Accès simplifié depuis la machine locale

### Prochaines étapes:
1. Configurer Cloudflare pour votre domaine
2. Ajouter vos applications dans `~/apps/`
3. Configurer les alertes de monitoring
4. Mettre en place un système de backup cloud