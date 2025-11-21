

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
sudo -u postgres psql -c "CREATE DATABASE paperless;"
sudo -u postgres psql -c "CREATE USER paperless WITH ENCRYPTED PASSWORD 'YOUR_SECRET_PASS';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;"
sudo -u postgres psql -c "ALTER USER paperless WITH SUPERUSER;"
```

**Explication :**

* `sudo -u postgres` exécute la commande avec l’utilisateur système PostgreSQL.
* Le rôle `SUPERUSER` n’est pas recommandé en production (principe du moindre privilège).

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
#!/usr/bin/env bash
set -e

apt update
apt install -y \
    python3 \
    python3-venv \
    python3-pip \
    python3-dev \
    build-essential \
    git \
    libmagic1 \
    tesseract-ocr \
    tesseract-ocr-eng \
    imagemagick \
    unpaper \
    poppler-utils \
    libxml2-dev \
    libxslt1-dev \
    libpq-dev \
    zlib1g-dev \
    libjpeg-dev \
    liblcms2-dev \
    libtiff-dev \
    libffi-dev \
    libcairo2-dev \
    libpango1.0-dev \
    libglib2.0-dev \
    libgirepository1.0-dev \
    gir1.2-pango-1.0

if ! id "paperless" >/dev/null 2>&1; then
    adduser --system --group --home /var/lib/paperless paperless
fi

mkdir -p /opt/paperless
chown paperless:paperless /opt/paperless


sudo -u paperless git clone https://github.com/paperless-ngx/paperless-ngx.git /opt/paperless/paperless-ngx

sudo -u paperless python3 -m venv /opt/paperless/venv
source /opt/paperless/venv/bin/activate

pip install -U pip wheel setuptools

cd /opt/paperless/paperless-ngx
sudo -u paperless /opt/paperless/venv/bin/pip install --upgrade pip wheel setuptools

# Install dependencies directly from pyproject.toml
sudo -u paperless /opt/paperless/venv/bin/pip install .

mkdir -p \
    /var/lib/paperless/media \
    /var/lib/paperless/data \
    /var/lib/paperless/consume

chown -R paperless:paperless /var/lib/paperless

cat >/etc/systemd/system/paperless.service <<'EOF'
[Unit]
Description=Paperless-NGX Document Management
After=network.target redis.service postgresql.service

[Service]
Type=simple
User=paperless
Group=paperless
Environment="PAPERLESS_REDIS=redis://localhost:6379"
Environment="PAPERLESS_DBHOST=localhost"
Environment="PAPERLESS_DBPORT=5432"
Environment="PAPERLESS_DBNAME=paperless"
Environment="PAPERLESS_DBUSER=paperless"
Environment="PAPERLESS_DBPASS=YourSecurePasswordHere"
Environment="PAPERLESS_BIND=0.0.0.0:8000"
WorkingDirectory=/opt/paperless/paperless-ngx/src
ExecStart=/opt/paperless/venv/bin/python3 manage.py runserver 0.0.0.0:8000
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and start service
systemctl daemon-reload
systemctl enable --now paperless
systemctl status paperless



systemctl daemon-reload
systemctl enable --now paperless

echo "Paperless-NGX installed successfully!"
echo "Accessible at: http://localhost:8000"
echo "Default user creation:"
echo "  source /opt/paperless/venv/bin/activate && paperless createsuperuser"


```


