# Projet Paperless-NGX — Administration Linux

 **Auteurs :** MENOURY Ethan & DJENID Camélia

 **Date :** 21 Novembre 2025

 **Système :** Debian 13 (Trixie)

 **Service déployé :** Paperless-NGX
 
Projet disponible sur GitHub : https://github.com/Cam-ams/Linux/edit/main/rendu.md
---

## Table des matières

1. [Présentation](#présentation)
2. [Installation complète](#installation-complète)

   * [Préparation du système](#préparation-du-système)
   * [Build du frontend](#build-du-frontend)
   * [Backend Django](#installation-et-configuration-python)
   * [Services systemd](#services-systemd)
3. [Diagnostic des erreurs](#diagnostic-des-erreurs)
4. [Backup automatisé (théorique)](#backup-automatisé-théorique)
5. [Scripts d'installation et d'automatisation](#scripts-dinstallation-et-dautomatisation)
   

---

# Présentation

Le but du projet est d’installer, configurer, sécuriser et automatiser Paperless-NGX sous Linux grâce à des scripts Bash.

Les objectifs couvrent :

* Installation système
* Base de données PostgreSQL
* Redis
* Services systemd
* Gunicorn + Django
* Build du frontend
* Diagnostic système
* Automatisation des sauvegardes

---


# Installation complète

## Préparation du système

### Mise à jour

```bash
apt update && apt upgrade -y
```

### Installation des dépendances

```bash
apt install -y python3 python3-pip python3-dev \
 imagemagick fonts-liberation gnupg libpq-dev default-libmysqlclient-dev \
 pkg-config libmagic-dev mime-support libzbar0 poppler-utils \
 unpaper ghostscript icc-profiles-free qpdf liblept5 \
 libxml2 pngquant zlib1g tesseract-ocr \
 tesseract-ocr-deu tesseract-ocr-eng \
 lsb-release curl gpg software-properties-common apt-transport-https \
 redis postgresql nodejs npm
```

---

## Configuration de Redis

```bash
systemctl enable redis-server --now
```

---

## Configuration de PostgreSQL

```bash
sudo -u postgres psql -c "CREATE DATABASE paperless;"
sudo -u postgres psql -c "CREATE USER paperless WITH ENCRYPTED PASSWORD 'YOURPASS';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;"
```

---

## Création d'un utilisateur système

```bash
adduser --system --home /opt/paperless --shell /bin/bash --group paperless
passwd paperless
```

---

## Récupération du code source

```bash
mkdir -p /opt/paperless
chown paperless:paperless /opt/paperless
sudo -u paperless git clone https://github.com/paperless-ngx/paperless-ngx /opt/paperless/paperless-ngx
```

---

## Build du frontend

```bash
cd /opt/paperless/paperless-ngx
sudo -u paperless npm install
sudo -u paperless npm run build
```

---

## Installation et configuration Python

```bash
sudo -Hu paperless pip3 install --upgrade pip setuptools wheel
sudo -Hu paperless pip3 install /opt/paperless/paperless-ngx --break-system-packages
sudo -Hu paperless python3 /opt/paperless/paperless-ngx/src/manage.py migrate
```

---

# Services systemd

## Document Consumer

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

---

## Celery Beat

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

---

## Celery Worker

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

---

## Gunicorn (Webserver)

```ini
[Unit]
Description=Paperless Webserver (Gunicorn)
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

## Activation des services

```bash
systemctl daemon-reload
systemctl enable --now paperless-consumer.service
systemctl enable --now paperless-scheduler.service
systemctl enable --now paperless-task-queue.service
systemctl enable --now paperless-webserver.service
```

---

# Diagnostic des erreurs

## Symptômes observés

* Login fonctionnel mais interface inaccessible
* Message "Something might be wrong"
* L’API ne répond pas :

  ```bash
  curl -I http://127.0.0.1:8000/api/
  ```

## Outils de diagnostic

```bash
journalctl -u paperless-webserver.service -f
journalctl -u paperless-webserver.service --no-pager | tail -50
systemctl status paperless-webserver.service
```

## Causes probables

 ->  Frontend non construit
 -> Permissions incorrectes
 -> Mauvaise configuration Django
 -> API backend non accessible

---

# Backup automatisé (théorique)

## Installation de Restic et Rclone

```bash
apt install -y restic rclone
```

## Script de backup (Restic)

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
    "$BACKUP_SOURCE/media" \
    "$BACKUP_SOURCE/data" \
    "$BACKUP_SOURCE/consume" \
    /tmp/paperless_db_backup.sql

rm -f /tmp/paperless_db_backup.sql
```

---

## Cron pour sauvegarde automatique

```bash
crontab -e
0 2 * * * /opt/scripts/backup_paperless.sh >> /var/log/paperless_backup.log 2>&1
```

## Scripts d'installation et d'automatisation

