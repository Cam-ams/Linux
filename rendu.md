

# Rapport Projet Paperless-NGX – Administration Linux

**Auteurs :** MENOURY Ethan & Djenid Camélia

**Date :** 21 Novembre 2025

**Service déployé :** Paperless-NGX

**Système :** Debian 13 (Trixie)


1. [Introduction](#1-introduction)
2. [Installation du logiciel](#2-installation-du-logiciel)
3. [Analyse des erreurs rencontrées](#3-analyse-des-erreurs-rencontrées)
4. [Backup automatisé (théorique)](#4-backup-automatisé-théorique)
5. [Sécurité réseau (théorique)](#5-sécurité-réseau-théorique)
6. [Script d'installation](#6-script-dinstallation)

Le rapport est également sur GitHub : https://github.com/Cam-ams/Linux/blob/main/rendu.md


## 1. Introduction

### Problématique

Comment installer, configurer, sécuriser et maintenir automatiquement Paperless-NGX sous Debian 13 en utilisant uniquement des scripts Bash, tout en assurant la sauvegarde des données et la sécurité du système ?

###  Composants principaux

* **Python 3** : Langage de programmation principal
nd

### Lien avec le cours

Ce projet mobilise l'ensemble des concepts d'administration Linux vus en cours :

* Gestion des processus via systemd
* Manipulation du système de fichiers
* Scripting Bash avancé
* Gestion des utilisateurs et permissions
* Configuration de services réseau

---

## 2. Installation du logiciel

### Préparation du système

####  Mise à jour

```bash
apt update && apt upgrade -y
```

**Explication :**
`apt` est le gestionnaire de paquets de Debian. Le flag `-y` automatise la confirmation, utile dans les scripts non-interactifs.

####  Installation des dépendances système

```bash
apt install -y python3 python3-pip python3-dev \
imagemagick fonts-liberation gnupg libpq-dev default-libmysqlclient-dev \
pkg-config libmagic-dev mime-support libzbar0 poppler-utils \
unpaper ghostscript icc-profiles-free qpdf liblept5 \
libxml2 pngquant zlib1g tesseract-ocr \
tesseract-ocr-deu tesseract-ocr-eng \
lsb-release curl gpg software-properties-common apt-transport-https ca-certificates \
redis postgresql nodejs npm git
```

**Explication :**

* Ces bibliothèques utilisent la liaison dynamique (`.so` sous Linux).
* Les bibliothèques `-dev` contiennent les headers nécessaires à la compilation de modules natifs Python.
* `/lib`, `/lib64`, `/usr/lib` : stockage des bibliothèques.
* Avantages : économie d’espace, mises à jour centralisées, factorisation en mémoire.

---

###  Configuration de Redis

```bash
systemctl enable redis-server --now
```

**Explication :**

Redis est un daemon (`-server`).
`systemd` gère le service, ses dépendances et le redémarrage automatique.
 `enable --now` : active le service au boot et le démarre immédiatement.

---

###  Configuration de PostgreSQL

```bash
apt install -y postgresql
sudo -u postgres psql <<EOF
CREATE DATABASE paperless;
CREATE USER paperless WITH ENCRYPTED PASSWORD 'YourSecurePasswordHere';
GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;
```

**Explication :**

* `sudo -u postgres` exécute la commande avec l’utilisateur système PostgreSQL.


---

### 2.4 Création de l'utilisateur système

```bash
adduser --system --home /opt/paperless --shell /bin/bash --group paperless
passwd paperless
```

**Explication :**

* Utilisateur dédié pour isoler Paperless-NGX.
* Limite les dégâts en cas de compromission.

---

###  Récupération du code source

```bash
mkdir -p /opt/paperless
chown paperless:paperless /opt/paperless
sudo -u paperless git clone https://github.com/paperless-ngx/paperless-ngx /opt/paperless/paperless-ngx
```

**Explication :**

* `chown user:group` : ajuste les permissions sur le dossier.
* Indispensable pour respecter le modèle de sécurité Linux.

---

### Build du frontend

```bash
cd /opt/paperless/paperless-ngx
sudo -u paperless npm install
sudo -u paperless npm run build
```

**Explication :**

* Compilation JavaScript moderne (transpilation + minification + bundling).
* Génère les fichiers statiques nécessaires au navigateur.

---
###  Configuration de l’arborescence

```bash
mkdir -p /opt/paperless/media /opt/paperless/data /opt/paperless/consume
chown -R paperless:paperless /opt/paperless/media
chown -R paperless:paperless /opt/paperless/data
chown -R paperless:paperless /opt/paperless/consume
```

**Explication :**

* `-R` applique la modification récursivement.
* `media/` : documents, `data/` : base de données, `consume/` : dossier surveillé par inotify.

---

###  Installation des dépendances Python

```bash
sudo -Hu paperless pip3 install --upgrade pip setuptools wheel
sudo -Hu paperless pip3 install /opt/paperless/paperless-ngx --break-system-packages
```

**Explication :**

* `-H` : définit la variable HOME de l’utilisateur cible.
* `--break-system-packages` : permet l’installation hors d’un environnement virtuel.

---

### Initialisation de la base de données

```bash
sudo -Hu paperless python3 /opt/paperless/paperless-ngx/src/manage.py migrate
```

**Explication :**

* Crée et met à jour le schéma PostgreSQL automatiquement via Django.

---

### Configuration des services systemd

####  Service Document Consumer

```ini
[Unit]
Description=Paperless document consumer
Requires=redis-server.service
After=redis-server.service

[Service]
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless/paperless-ngx/src
ExecStart=/usr/bin/env python3 manage.py document_consumer

[Install]
WantedBy=multi-user.target
```

**Explication :**

* Services dépendants de Redis.
* Isolation via l’utilisateur `paperless`.

#### Service Celery Beat 

```ini
[Unit]
Description=Paperless Celery Beat 
Requires=redis-server.service
After=redis-server.service

[Service]
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless/paperless-ngx/src
ExecStart=/opt/paperless/.local/bin/celery --app paperless beat --loglevel INFO

[Install]
WantedBy=multi-user.target
```

####  Service Celery Workers

```ini
[Unit]
Description=Paperless Celery Workers
Requires=redis-server.service
After=redis-server.service

[Service]
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless/paperless-ngx/src
ExecStart=/opt/paperless/.local/bin/celery --app paperless worker --loglevel INFO

[Install]
WantedBy=multi-user.target
```

#### Service Gunicorn

```ini
[Unit]
Description=Paperless webserver 
After=network.target
Requires=redis-server.service

[Service]
User=paperless
Group=paperless
WorkingDirectory=/opt/paperless/paperless-ngx/src
ExecStart=/opt/paperless/.local/bin/gunicorn -c /opt/paperless/paperless-ngx/gunicorn.conf.py paperless.asgi:application

[Install]
WantedBy=multi-user.target
```

---

### Activation des services

```bash
systemctl daemon-reload
systemctl enable --now paperless-consumer.service
systemctl enable --now paperless-scheduler.service
systemctl enable --now paperless-task-queue.service
systemctl enable --now paperless-webserver.service
```

**Explication :**

* `daemon-reload` recharge la configuration systemd.

---

###  Vérification de l’installation

```bash
systemctl status paperless-*.service
journalctl -u paperless-webserver.service -f
curl http://localhost:8000
```

---

## 3. Analyse des erreurs rencontrées

###  Problème principal : "Something might be wrong"

* La page de login s’affiche mais l’API échoue.
* `curl -I http://127.0.0.1:8000/api/` ne répond pas.
* Gunicorn semble actif.

### Causes probables

1. Frontend non construit correctement
2. Permissions incorrectes
3. Mauvaise configuration Django (`STATIC_URL`, `ALLOWED_HOSTS`)
4. API backend inaccessible

###  Vérification des permissions

```bash
ls -l /opt/paperless/paperless-ngx/src/documents/static/
chmod -R 755 /opt/paperless/paperless-ngx/src/documents/static/
chown -R paperless:paperless /opt/paperless/paperless-ngx/
```

---

## 4. Backup automatisé (théorique)

###  Installation de Restic et Rclone

```bash
apt update
apt install -y restic rclone
```

###  Script de backup Restic

```bash
#!/bin/bash
set -e

RESTIC_REPOSITORY="/opt/backups/paperless-restic"
RESTIC_PASSWORD_FILE="/root/.restic_password"
BACKUP_SOURCE="/opt/paperless"

if [ ! -d "$RESTIC_REPOSITORY" ]; then
    restic init --repo "$RESTIC_REPOSITORY" --password-file "$RESTIC_PASSWORD_FILE"
fi

sudo -u postgres pg_dump paperless > /tmp/paperless_db_backup.sql

restic backup \
    --repo "$RESTIC_REPOSITORY" \
    --password-file "$RESTIC_PASSWORD_FILE" \
    --tag "paperless" \
    "$BACKUP_SOURCE/media" \
    "$BACKUP_SOURCE/data" \
    "$BACKUP_SOURCE/consume" \
    /tmp/paperless_db_backup.sql

rm -f /tmp/paperless_db_backup.sql
restic forget --repo "$RESTIC_REPOSITORY" \
    --password-file "$RESTIC_PASSWORD_FILE" \
    --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

### Cron pour automatisation

```bash
0 2 * * * /opt/scripts/backup_paperless.sh >> /var/log/paperless_backup.log 2>&1
```

---

## 5. Sécurité réseau (théorique)

* Limiter l’accès aux ports 5432 (PostgreSQL), 6379 (Redis), 8000 (Gunicorn) via firewall (`ufw` ou `iptables`).
* Configurer HTTPS (certbot ou reverse proxy Nginx) pour le serveur web.
* Créer des utilisateurs avec des droits limités (`principle of least privilege`).

---

## 6. Script d'installation complet

```bash


# CONFIGURATION


PAPERLESS_USER=paperless
PAPERLESS_HOME=/opt/paperless
VENV_DIR=$PAPERLESS_HOME/venv
DB_NAME=paperless
DB_USER=paperless
PAPERLESS_URL="http://127.0.0.1:8000"
TIMEZONE="Europe/Paris"
CONSUME_DIR="$PAPERLESS_HOME/consume"
DATA_DIR="$PAPERLESS_HOME/data"
MEDIA_DIR="$PAPERLESS_HOME/media"
STATIC_DIR="$PAPERLESS_HOME/static"
WORKDIR="$PAPERLESS_HOME/src"


# INSTALLATION DES DEPENDANCES


apt update
apt install -y git curl wget ca-certificates gnupg python3 python3-venv python3-dev python3-pip build-essential libpq-dev redis-server postgresql postgresql-contrib tesseract-ocr qpdf poppler-utils imagemagick libmagic-dev libzbar0 pkg-config fonts-liberation npm

systemctl enable --now redis-server


# UTILISATEUR PAPERLESS


adduser --system --group --home "$PAPERLESS_HOME" --shell /bin/bash "$PAPERLESS_USER"

mkdir -p "$PAPERLESS_HOME"
chown -R "$PAPERLESS_USER:$PAPERLESS_USER" "$PAPERLESS_HOME"


# POSTGRESQL


read -rsp "Entrez le mot de passe PostgreSQL pour l'utilisateur $DB_USER : " DB_PASS
echo

sudo -u postgres psql -c "CREATE USER $DB_USER WITH PASSWORD '$DB_PASS';"
sudo -u postgres psql -c "CREATE DATABASE $DB_NAME OWNER $DB_USER;"


# CLONER LE REPO

sudo -u "$PAPERLESS_USER" mkdir -p /opt/temp
sudo -u "$PAPERLESS_USER" cd /opt/temp
sudo -u "$PAPERLESS_USER" git clone https://github.com/paperless-ngx/paperless-ngx
sudo -u "$PAPERLESS_USER" cp -a paperless-ngx/. /opt/paperless/
sudo -u "$PAPERLESS_USER" chown -R paperless:paperless /opt/paperless

# VIRTUALENV


sudo -u "$PAPERLESS_USER" python3 -m venv "$VENV_DIR"
sudo -u "$PAPERLESS_USER" "$VENV_DIR/bin/pip" install --upgrade pip setuptools wheel
sudo -u "$PAPERLESS_USER" "$VENV_DIR/bin/pip" install django redis gunicorn psycopg2-binary django-allauth concurrent-log-handler==0.9.25


# ARBORESCENCE ET PERMISSIONS


mkdir -p "$CONSUME_DIR" "$DATA_DIR" "$MEDIA_DIR" "$STATIC_DIR"
chown -R "$PAPERLESS_USER:$PAPERLESS_USER" "$PAPERLESS_HOME" "$CONSUME_DIR" "$DATA_DIR" "$MEDIA_DIR" "$STATIC_DIR"
chmod 750 "$CONSUME_DIR" "$DATA_DIR" "$MEDIA_DIR"


# CREATION DU .ENV


SECRET_KEY=$(sudo -u "$PAPERLESS_USER" "$VENV_DIR/bin/python" -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())")

cat > "$PAPERLESS_HOME/.env" <<EOF
SECRET_KEY=$SECRET_KEY
PAPERLESS_REDIS=redis://localhost:6379
PAPERLESS_DBHOST=localhost
PAPERLESS_DBNAME=$DB_NAME
PAPERLESS_DBUSER=$DB_USER
PAPERLESS_DBPASS=$DB_PASS
PAPERLESS_TIME_ZONE=$TIMEZONE
PAPERLESS_CONSUMPTION_DIR=$CONSUME_DIR
PAPERLESS_DATA_DIR=$DATA_DIR
PAPERLESS_MEDIA_ROOT=$MEDIA_DIR
PAPERLESS_STATICDIR=$STATIC_DIR
PAPERLESS_URL=$PAPERLESS_URL
EOF

chmod 600 "$PAPERLESS_HOME/.env"
chown "$PAPERLESS_USER:$PAPERLESS_USER" "$PAPERLESS_HOME/.env"


# FRONT-END BUILD


UI_DIR="$PAPERLESS_HOME/src"

cd "$UI_DIR"
chown -R "$PAPERLESS_USER:$PAPERLESS_USER" "$UI_DIR"

sudo -u "$PAPERLESS_USER" npm install --no-audit --no-fund
sudo -u "$PAPERLESS_USER" npm run build || echo "Frontend build échoué"

BUILD_DIR="$UI_DIR/dist"
rm -rf "$STATIC_DIR"/*
cp -a "$BUILD_DIR"/. "$STATIC_DIR"/
chown -R "$PAPERLESS_USER:$PAPERLESS_USER" "$STATIC_DIR"


# MIGRATIONS & COLLECTSTATIC


cd "$WORKDIR"
chown -R "$PAPERLESS_USER:$PAPERLESS_USER" "$WORKDIR"

sudo -u "$PAPERLESS_USER" "$VENV_DIR/bin/python" manage.py migrate --noinput
sudo -u "$PAPERLESS_USER" "$VENV_DIR/bin/python" manage.py collectstatic --noinput


# SERVICES SYSTEMD


cat > /etc/systemd/system/paperless-web.service <<EOF
[Unit]
Description=Paperless-NGX Web Server (gunicorn)
After=network.target redis-server.service postgresql.service
Wants=redis-server.service postgresql.service

[Service]
User=$PAPERLESS_USER
Group=$PAPERLESS_USER
EnvironmentFile=$PAPERLESS_HOME/.env
WorkingDirectory=$WORKDIR
ExecStart=$VENV_DIR/bin/gunicorn --bind 0.0.0.0:8000 paperless.wsgi:application
Restart=on-failure
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/paperless-consumer.service <<EOF
[Unit]
Description=Paperless-NGX Document Consumer
After=network.target redis-server.service postgresql.service

[Service]
User=$PAPERLESS_USER
Group=$PAPERLESS_USER
EnvironmentFile=$PAPERLESS_HOME/.env
WorkingDirectory=$WORKDIR
ExecStart=$VENV_DIR/bin/python manage.py document_consumer
Restart=on-failure
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now paperless-web.service
systemctl enable --now paperless-consumer.service


# VERIFICATIONS


sleep 2
ss -tulpn | grep 8000 || echo "Port 8000 libre"
systemctl status paperless-web --no-pager
systemctl status paperless-consumer --no-pager

echo "Installation terminée"


```


