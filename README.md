# 🛡️ Projet Bastion – Apache Guacamole

![Debian](https://img.shields.io/badge/Debian-13-A81D33)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED)
![Nginx](https://img.shields.io/badge/Nginx-Reverse%20Proxy-009639)
![Guacamole](https://img.shields.io/badge/Apache-Guacamole-FF6600)
![VMware](https://img.shields.io/badge/VMware-Workstation-607078)
![License](https://img.shields.io/badge/License-MIT-blue)

> Centralisation et sécurisation des accès distants à des machines virtuelles via Apache Guacamole, accessible depuis un simple navigateur web.

---

## 📋 Sommaire

- [Présentation](#-présentation)
- [Architecture](#-architecture)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Accès](#-accès)
- [Sécurisation](#-sécurisation)
- [Groupes & Utilisateurs](#-groupes--utilisateurs)
- [MFA / TOTP](#-mfa--totp)
- [Compétences](#-compétences)

---

## 📋 Présentation

Apache Guacamole est une solution d'accès distant **clientless** open-source. Elle permet de se connecter à des machines distantes via un **simple navigateur web** sans installation de logiciel client, en supportant les protocoles **SSH**, **RDP** et **VNC**.

Ce projet déploie un bastion sécurisé centralisant l'accès à 3 machines cibles :

| Machine | Protocole | Port |
|---------|-----------|------|
| Debian CLI | SSH | 22 |
| Debian GUI (XFCE) | VNC | 5902 |
| Windows 10 | RDP | 3389 |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Host Ubuntu                           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              VMware Workstation                  │   │
│  │                                                  │   │
│  │  ┌─────────────┐    VMnet2 (192.168.50.0/24)    │   │
│  │  │   BASTION   │◄──────────────────────────────►│   │
│  │  │ ens34: .50.1│    ┌──────────────┐            │   │
│  │  │ ens37: NAT  │    │  Debian CLI  │            │   │
│  │  │             │    │  192.168.50.30│ SSH        │   │
│  │  │ Guacamole   │    └──────────────┘            │   │
│  │  │ (Docker)    │    ┌──────────────┐            │   │
│  │  │ Nginx HTTPS │    │  Debian GUI  │            │   │
│  │  │             │    │  192.168.50.20│ VNC:5902  │   │
│  │  └─────────────┘    └──────────────┘            │   │
│  │        ▲            ┌──────────────┐            │   │
│  │        │            │  Windows 10  │            │   │
│  └────────┼────────────│  192.168.50.10│ RDP:3389 ─┘   │
│           │            └──────────────┘                 │
│    HTTPS :443                                           │
└─────────────────────────────────────────────────────────┘
         ▲
    Navigateur
```

---

## 📦 Prérequis

- VMware Workstation (26H1 ou supérieur)
- 4 VMs : Bastion Debian 13, Debian CLI, Debian GUI (XFCE), Windows 10
- Docker + Docker Compose installés sur le Bastion
- Nginx installé sur le Bastion
- OpenSSL pour la génération du certificat SSL

---

## 🚀 Installation

### 1. Déploiement de Guacamole via Docker Compose

```bash
mkdir -p /opt/guacamole && cd /opt/guacamole
```

Créer le fichier `docker-compose.yml` :

```yaml
services:
  guacd:
    image: guacamole/guacd
    restart: always

  guacamole-db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole
      MYSQL_PASSWORD: guacpass
    restart: always

  guacamole:
    image: guacamole/guacamole
    depends_on:
      - guacd
      - guacamole-db
    volumes:
      - /opt/guacamole/extensions/guacamole-auth-totp/guacamole-auth-totp.jar:/etc/guacamole/extensions/guacamole-auth-totp.jar
    environment:
      GUACD_HOSTNAME: guacd
      MYSQL_HOSTNAME: guacamole-db
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole
      MYSQL_PASSWORD: guacpass
    ports:
      - "127.0.0.1:8080:8080"
    restart: always
```

```bash
# Initialiser la base de données
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > /opt/guacamole/initdb.sql
docker compose up -d
docker exec -i guacamole-guacamole-db-1 mysql -u guacamole -pguacpass guacamole_db < /opt/guacamole/initdb.sql
```

---

### 2. Configuration VNC sur la Debian GUI

```bash
# Installer TigerVNC + XFCE + dbus-x11
apt install tigervnc-standalone-server xfce4 xfce4-goodies dbus-x11 -y

# Créer le script xstartup
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x ~/.vnc/xstartup

# Lancer VNC
vncserver :2 -geometry 1280x800 -depth 24 -localhost no -xstartup ~/.vnc/xstartup
```

**Service systemd pour démarrage automatique :**

```bash
cat > /etc/systemd/system/vncserver@.service << 'EOF'
[Unit]
Description=TigerVNC server
After=network.target

[Service]
Type=forking
User=alexis
WorkingDirectory=/home/alexis
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill :%i > /dev/null 2>&1 || true'
ExecStart=/usr/bin/vncserver :%i -geometry 1280x800 -depth 24 -localhost no -xstartup /home/alexis/.vnc/xstartup
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable vncserver@2.service
systemctl start vncserver@2.service
```

---

### 3. Reverse Proxy HTTPS avec Nginx

```bash
# Générer le certificat SSL auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/guacamole.key \
  -out /etc/ssl/certs/guacamole.crt \
  -subj "/CN=bastion"

# Configurer Nginx
cat > /etc/nginx/sites-available/guacamole << 'EOF'
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/guacamole.crt;
    ssl_certificate_key /etc/ssl/private/guacamole.key;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_cookie_path /guacamole/ /;
    }
}
EOF

ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx
```

---

### 4. MFA / TOTP

```bash
# Extraire l'extension TOTP depuis le container
mkdir -p /opt/guacamole/extensions
docker cp guacamole-guacamole-1:/opt/guacamole/extensions/guacamole-auth-totp \
  /opt/guacamole/extensions/

# Recréer le container avec le volume monté
docker compose up -d --force-recreate guacamole
```

---

## 🌐 Accès

Depuis le navigateur sur le réseau NAT :

```
https://<IP_BASTION>/guacamole/
```

Identifiants par défaut : `guacadmin` / `guacadmin`

> ⚠️ Changer le mot de passe par défaut après la première connexion.

---

## 🔒 Sécurisation

| Mesure | Description |
|--------|-------------|
| HTTPS | Certificat SSL, redirection HTTP → HTTPS |
| HSTS | Header `Strict-Transport-Security` |
| Isolation Tomcat | Port 8080 bindé sur `127.0.0.1` uniquement |
| Réseau isolé | VMnet2 sans accès internet direct |
| MFA TOTP | Double authentification via application OTP |

---

## 👥 Groupes & Utilisateurs

| Groupe | Accès |
|--------|-------|
| Admin | Debian-CLI + Debian-GUI + Windows 10 |
| Windaube | Windows 10 uniquement |
| Linux | Debian-CLI + Debian-GUI |
| UI | Windows 10 + Debian-GUI |

---

## 🔐 MFA / TOTP

L'extension `guacamole-auth-totp` est activée. Lors de la première connexion, un QR Code est affiché — le scanner avec **Google Authenticator** ou **Aegis** pour enrôler le compte. Un code à 6 chiffres sera demandé à chaque connexion.

---

## 🧠 Compétences démontrées

- **Virtualisation** : VMware Workstation, réseaux virtuels NAT/Host-Only
- **Linux** : Debian 13, systemd, ifupdown, VNC/TigerVNC, XFCE
- **Docker** : Docker Compose, volumes, bind mounts, port binding sécurisé
- **Sécurité réseau** : Nginx reverse proxy, SSL/TLS, HSTS, isolation réseau, MFA TOTP
- **Protocoles** : SSH, RDP, VNC, WebSocket, HTTP/HTTPS
- **Administration** : Gestion des droits, groupes Guacamole, authentification multi-facteurs

---

## 👤 Auteur

**Alexis Noiret** – Étudiant en Bachelor IT Cybersécurité @ La Plateforme, Cannes

[![GitHub](https://img.shields.io/badge/GitHub-alexis--noiret-181717?style=flat-square&logo=github)](https://github.com/alexis-noiret)
[![Portfolio](https://img.shields.io/badge/Portfolio-alexis--noiret.students--laplateforme.io-00B4D8?style=flat-square&logo=google-chrome&logoColor=white)](https://alexis-noiret.students-laplateforme.io)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Alexis%20Noiret-0077B5?style=flat-square&logo=linkedin)](https://linkedin.com/in/alexis-noiret-726019349)
