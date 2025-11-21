# Linux

Le projet Paperless utilise les connaissances apprises dans le module Linux : gestion des services, manipulation du système de fichiers, création de scripts Bash, gestion des permissions, sécurité réseau et organisation du système.
 Paperless a besoin d’un environnement bien configuré (Python, base de données, services comme Redis, outils de traitement de documents…). Le cours nous a donc appris les bases nécessaires pour :
comprendre comment fonctionne un système Linux,
automatiser des installations répétitives,
sécuriser un service disponible sur le réseau,
assurer la maintenance grâce à des sauvegardes et à l’analyse des logs.
L’objectif du projet est de mettre en pratique ces notions en créant un processus d’installation automatique, fiable et reproductible pour installer Paperless sur Debian 13 uniquement avec des scripts Bash.
La question principale à laquelle ce projet répond est :
 Comment installer, configurer, sécuriser et maintenir automatiquement Paperless sous Debian 13 en utilisant uniquement des scripts Linux ?

Script d'instalation : 
````
#!/bin/bash

set -e

# DÉPENDANCES SYSTÈME
apt update
apt install -y \
  git \
  python3 python3-venv python3-dev python3-pip \
  build-essential \
  libpq-dev \
  redis-server \
  postgresql postgresql-contrib \
  tesseract-ocr \
  qpdf \
  poppler-utils \
  imagemagick \
  libmagic-dev \
  libzbar0 \
  pkg-config \
  fonts-liberation \
  gnupg

systemctl enable --now redis-server

#  UTILISATEUR PAPERLESS
adduser --system --group --home /opt/paperless paperless || true

# BASE DE DONNÉES POSTGRES
echo -n "Entrez le mot de passe PostgreSQL pour l'utilisateur paperless : "
read -s DB_PASS
sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='paperless'" | grep -q 1 \
  || sudo -u postgres psql -c "CREATE USER paperless WITH PASSWORD '$DB_PASS';"
sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname='paperless'" | grep -q 1 \
  || sudo -u postgres psql -c "CREATE DATABASE paperless OWNER paperless;"

#  RÉCUPÉRATION DU CODE PAPERLESS-NGX
mkdir -p /opt/temp
cd /opt/temp
git clone https://github.com/paperless-ngx/paperless-ngx
cp -a paperless-ngx/. /opt/paperless/
chown -R paperless:paperless /opt/paperless

cd /opt/paperless

# ENVIRONNEMENT PYTHON
sudo -u paperless python3 -m venv /opt/paperless/venv
source /opt/paperless/venv/bin/activate

pip install --upgrade pip wheel setuptools

# INSTALLATION DES DÉPENDANCES (uv + correctifs)
pip install uv

cd /opt/paperless

# installer via pyproject.toml
uv pip install -r pyproject.toml

# correctifs importants
pip install concurrent-log-handler==0.9.25
pip install --upgrade django-allauth
pip install psycopg2

pip install "gunicorn==22.*"
pip install "uvicorn[standard]==0.30.*"

deactivate

# ARBORESCENCE PAPERLESS
sudo -u paperless mkdir -p /opt/paperless/media
sudo -u paperless mkdir -p /opt/paperless/data
sudo -u paperless mkdir -p /opt/paperless/consume

sudo chown -R paperless:paperless /opt/paperless/media /opt/paperless/data /opt/paperless/consume

sudo chmod 755 /opt/paperless/media /opt/paperless/data /opt/paperless/consume

# FICHIER .ENV
SECRET_KEY=$(/opt/paperless/venv/bin/python <<'EOF'
from django.core.management.utils import get_random_secret_key
print(get_random_secret_key())
EOF
)

cat <<EOF > /opt/paperless/.env
SECRET_KEY=$SECRET_KEY
PAPERLESS_REDIS=redis://localhost:6379
PAPERLESS_DBHOST=localhost
PAPERLESS_DBNAME=paperless
PAPERLESS_DBUSER=paperless
PAPERLESS_DBPASS=$DB_PASS
PAPERLESS_TIME_ZONE=Europe/Paris
PAPERLESS_CONSUMPTION_DIR=/opt/paperless/consume
PAPERLESS_DATA_DIR=/opt/paperless/data
PAPERLESS_MEDIA_ROOT=/opt/paperless/media
PAPERLESS_STATICDIR=/opt/paperless/static
PAPERLESS_URL=http://localhost:8000
EOF

chmod 600 /opt/paperless/.env
chown paperless:paperless /opt/paperless/.env

# CONF
cd /opt/paperless/
cp paperless.conf.example paperless.conf

sed -i -e '5,12s/^#//' -e '16,17s/^#//' -e '19,20s/^#//' -e "s/^PAPERLESS_DBPASS=.*/PAPERLESS_DBPASS=$DB_PASS/" paperless.conf

# MIGRATIONS DJANGO + ADMIN
source /opt/paperless/venv/bin/activate
cd /opt/paperless/src

sudo -u paperless /opt/paperless/venv/bin/python manage.py migrate --noinput
sudo -u paperless /opt/paperless/venv/bin/python manage.py collectstatic --noinput

echo "=== CRÉATION SUPERUTILISATEUR PAPERLESS ==="
sudo -u paperless /opt/paperless/venv/bin/python manage.py createsuperuser

deactivate

#  Vérification et Migrations
echo "=== APPLICATION DES MIGRATIONS DJANGO ==="
source /opt/paperless/venv/bin/activate
cd /opt/paperless/src

# Appliquer toutes les migrations
sudo -u paperless /opt/paperless/venv/bin/python manage.py migrate --noinput

# Vérifier l'état des migrations
sudo -u paperless /opt/paperless/venv/bin/python manage.py showmigrations

deactivate

# SYSTEMD : SERVEUR WEB
cat <<EOF > /etc/systemd/system/paperless-web.service
[Unit]
Description=Paperless-NGX Web Server
After=network.target redis-server.service postgresql.service

[Service]
User=paperless
EnvironmentFile=/opt/paperless/.env
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/gunicorn \
    --bind 0.0.0.0:8000 \
    paperless.asgi:application \
    -k uvicorn.workers.UvicornWorker
Restart=always

[Install]
WantedBy=multi-user.target
EOF


# SYSTEMD : CONSUMER


cat <<EOF > /etc/systemd/system/paperless-consumer.service
[Unit]
Description=Paperless-NGX Document Consumer

[Service]
User=paperless
EnvironmentFile=/opt/paperless/.env
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/python manage.py document_consumer
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# ACTIVATION DES SERVICES
systemctl daemon-reload
systemctl enable --now paperless-web.service
systemctl enable --now paperless-consumer.service

echo ""
echo "=============================================="
echo " INSTALLATION TERMINÉE !"
echo " Paperless est disponible sur : http://localhost:8000"
echo "=============================================="
````
## Lien avec le cours



## Conclusion

Ce projet nous a permis d’appliquer les notions essentielles du module Linux : gestion des services, permissions, sécurité, scripts Bash, environnement système et maintenance.
Grâce à ces connaissances, nous avons pu créer un script complet capable d’automatiser l’installation et la configuration de Paperless sur Debian 13.

Ce travail montre l’importance de l’automatisation pour faciliter les déploiements, rendre les installations plus fiables et garantir la reproductibilité des environnements.

Il met aussi en évidence le rôle des bonnes pratiques d’administration système pour sécuriser et organiser un service.

En résumé, ce projet nous a permis de comprendre comment combiner théorie et pratique pour installer, sécuriser et gérer un service réel de manière entièrement automatique.

C’est une étape importante pour apprendre à administrer un système Linux de façon professionnelle.


________________________________________________________________________________________________________________________________________________________________________

# Rapport Projet Paperless-NGX - Administration Linux

**Auteurs :** [Vos noms]  
**Date :** Novembre 2025  
**Service déployé :** Paperless-NGX  
**Système :** Debian 13 (Trixie)

---

## Table des matières

- [1. Introduction](#1-introduction)
- [2. Installation du logiciel](#2-installation-du-logiciel)
- [3. Backup automatisé](#3-backup-automatisé)
- [4. Sécurité réseau](#4-sécurité-réseau)
- [5. Monitoring](#5-monitoring)
- [6. Lien avec les concepts théoriques](#6-lien-avec-les-concepts-théoriques)
- [7. Conclusion](#7-conclusion)

---

## 1. Introduction

### 1.1. Contexte du projet

Paperless-NGX est une application open-source de gestion électronique de documents (GED) permettant de numériser, indexer et archiver des documents PDF de manière intelligente. Elle utilise l'OCR (reconnaissance optique de caractères) pour rendre les documents scannés recherchables et propose un système de tags, correspondants et types de documents pour faciliter l'organisation.

### 1.2. Problématique

> **Comment installer, configurer, sécuriser et maintenir automatiquement Paperless-NGX sous Debian 13 en utilisant uniquement des scripts Bash, tout en assurant la haute disponibilité, la sauvegarde des données et le monitoring du système ?**

### 1.3. Architecture technique

L'architecture de Paperless-NGX repose sur plusieurs composants interconnectés :

```
┌─────────────────────────────────────────────────────────┐
│                    Client Web (Port 8000)                │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│              Gunicorn + Uvicorn (ASGI)                   │
│                 (paperless-web.service)                  │
└────────────┬───────────────────────────┬─────────────────┘
             │                           │
┌────────────▼──────────┐   ┌───────────▼──────────────┐
│   PostgreSQL          │   │   Redis                   │
│   (Base de données)   │   │   (Cache & Queue)         │
└───────────────────────┘   └──────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────┐
│         Document Consumer (paperless-consumer.service)   │
│         + Tesseract OCR + ImageMagick + Poppler          │
└──────────────────────────────────────────────────────────┘
```

**Composants principaux :**
- **Python 3** avec environnement virtuel (isolation des dépendances)
- **PostgreSQL** : Base de données relationnelle pour stocker les métadonnées
- **Redis** : Gestion du cache et des files d'attente asynchrones
- **Tesseract OCR** : Reconnaissance optique de caractères
- **Gunicorn + Uvicorn** : Serveurs web ASGI pour Django
- **Systemd** : Gestion des services et démarrage automatique

---

## 2. Installation du logiciel

### 2.1. Préparation du système

#### 2.1.1. Installation des dépendances système

```bash
apt update
apt install -y \
  git \
  python3 python3-venv python3-dev python3-pip \
  build-essential \
  libpq-dev \
  redis-server \
  postgresql postgresql-contrib \
  tesseract-ocr \
  qpdf \
  poppler-utils \
  imagemagick \
  libmagic-dev \
  libzbar0 \
  pkg-config \
  fonts-liberation \
  gnupg
```

**Lien avec le cours :**  
Cette étape illustre la **gestion des dépendances système** (cours "Le système d'exploitation", slide 9). Le gestionnaire de paquets `apt` (spécifique à Debian) résout automatiquement les dépendances transitives et évite les conflits de versions.

Les bibliothèques partagées comme `libpq-dev` (PostgreSQL) ou `libmagic-dev` sont nécessaires pour la compilation. Ces bibliothèques suivent le principe de **liaison dynamique** (fichiers `.so`), permettant :
-  Économie d'espace disque (factorisation)
-  Mises à jour de sécurité centralisées
- Partage en mémoire entre processus

#### 2.1.2. Activation du service Redis

```bash
systemctl enable --now redis-server
```

**Lien avec le cours :**  
`systemd` est le gestionnaire de services de Linux moderne (cours "Le système d'exploitation", slide 3). Redis fonctionne comme un **daemon** (`redis-server`), suivant la convention de nommage avec le suffixe "d".

- `enable` : Configure le démarrage automatique au boot (via `/etc/systemd/system/`)
- `--now` : Démarre immédiatement le service sans attendre le redémarrage

### 2.2. Création d'un utilisateur système dédié

```bash
adduser --system --group --home /opt/paperless paperless || true
```

**Lien avec le cours :**  
Création d'un **utilisateur système** (cours "Administration Linux", slide 6). Cet utilisateur :
- N'a pas de shell interactif
- N'a pas de mot de passe utilisable pour se connecter
- Est dédié uniquement à l'exécution du service Paperless
- Limite les dégâts en cas de compromission (principe du **moindre privilège**)

Si un attaquant exploite une vulnérabilité dans Paperless, il sera confiné aux permissions de l'utilisateur `paperless` et ne pourra pas :
- Accéder aux fichiers d'autres utilisateurs
- Modifier les configurations système dans `/etc`
- Compromettre d'autres services

### 2.3. Configuration de PostgreSQL

```bash
echo -n "Entrez le mot de passe PostgreSQL pour l'utilisateur paperless : "
read -s DB_PASS

sudo -u postgres psql -tc "SELECT 1 FROM pg_roles WHERE rolname='paperless'" | grep -q 1 \
  || sudo -u postgres psql -c "CREATE USER paperless WITH PASSWORD '$DB_PASS';"

sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname='paperless'" | grep -q 1 \
  || sudo -u postgres psql -c "CREATE DATABASE paperless OWNER paperless;"
```

**Lien avec le cours :**  
- **Gestion des utilisateurs** : PostgreSQL possède son propre utilisateur système `postgres` (cours "Administration Linux", slide 5-6)
- **sudo -u postgres** : Exécute les commandes avec les privilèges de l'utilisateur `postgres`, illustrant la commutation d'identité
- **Idempotence** : Les vérifications avec `grep -q` permettent de relancer le script sans erreur
- **Sécurité** : `read -s` masque le mot de passe lors de la saisie (pas d'affichage, pas d'enregistrement dans l'historique bash)

### 2.4. Récupération du code source

```bash
mkdir -p /opt/temp
cd /opt/temp
git clone https://github.com/paperless-ngx/paperless-ngx
cp -a paperless-ngx/. /opt/paperless/
chown -R paperless:paperless /opt/paperless
```

**Lien avec le cours :**  
**Gestion des permissions** (cours "Administration Linux", slide 9) :
- `chown -R` : Modifie récursivement le propriétaire ET le groupe
- Format `utilisateur:groupe` : Définit simultanément les deux attributs
- Garantit que seul l'utilisateur `paperless` contrôle les fichiers de l'application

### 2.5. Environnement virtuel Python

```bash
sudo -u paperless python3 -m venv /opt/paperless/venv
source /opt/paperless/venv/bin/activate
pip install --upgrade pip wheel setuptools
```

**Lien avec le cours :**  
L'environnement virtuel Python (`venv`) crée un espace **isolé** pour les dépendances Python, évitant les conflits avec les paquets système. C'est l'équivalent Python du concept de **bibliothèques partagées**, mais avec une isolation par projet.

**Avantages :**
- Versions de bibliothèques spécifiques au projet
- Pas de pollution du système global
- Reproductibilité entre environnements

L'utilisation de `sudo -u paperless` garantit que l'environnement appartient à l'utilisateur `paperless` et non à `root`.

### 2.6. Installation des dépendances Python

```bash
pip install uv
uv pip install -r pyproject.toml
pip install concurrent-log-handler==0.9.25
pip install --upgrade django-allauth
pip install psycopg2 "gunicorn==22.*" "uvicorn[standard]==0.30.*"
```

**Dépendances clés :**
- `uv` : Installeur de paquets Python ultra-rapide (écrit en Rust)
- `psycopg2` : Adaptateur PostgreSQL pour Python
- `gunicorn` : Serveur WSGI/ASGI production-ready
- `uvicorn` : Serveur ASGI asynchrone (pour WebSockets et HTTP/2)

### 2.7. Configuration de l'arborescence

```bash
sudo -u paperless mkdir -p /opt/paperless/media
sudo -u paperless mkdir -p /opt/paperless/data
sudo -u paperless mkdir -p /opt/paperless/consume

sudo chown -R paperless:paperless /opt/paperless/media /opt/paperless/data /opt/paperless/consume
sudo chmod 755 /opt/paperless/media /opt/paperless/data /opt/paperless/consume
```

**Structure des dossiers :**
- `media/` : Stockage des documents PDF originaux et miniatures
- `data/` : Base de données SQLite (non utilisée avec PostgreSQL) et index de recherche
- `consume/` : Dossier surveillé pour l'ajout automatique de documents

**Lien avec le cours :**  
Les **permissions 755** (cours "Administration Linux", slide 10) signifient :

| Utilisateur | Groupe | Autres |
|-------------|--------|--------|
| `rwx` (7)   | `r-x` (5) | `r-x` (5) |
| Lecture, écriture, exécution | Lecture, exécution | Lecture, exécution |

**En octal :**
- **7** = 4 (read) + 2 (write) + 1 (execute)
- **5** = 4 (read) + 1 (execute)
- **5** = 4 (read) + 1 (execute)

### 2.8. Configuration de l'environnement (.env)

```bash
SECRET_KEY=$(/opt/paperless/venv/bin/python <<'EOF'
from django.core.management.utils import get_random_secret_key
print(get_random_secret_key())
EOF
)

cat <<EOF > /opt/paperless/.env
SECRET_KEY=$SECRET_KEY
PAPERLESS_REDIS=redis://localhost:6379
PAPERLESS_DBHOST=localhost
PAPERLESS_DBNAME=paperless
PAPERLESS_DBUSER=paperless
PAPERLESS_DBPASS=$DB_PASS
PAPERLESS_TIME_ZONE=Europe/Paris
PAPERLESS_CONSUMPTION_DIR=/opt/paperless/consume
PAPERLESS_DATA_DIR=/opt/paperless/data
PAPERLESS_MEDIA_ROOT=/opt/paperless/media
PAPERLESS_STATICDIR=/opt/paperless/static
PAPERLESS_URL=http://localhost:8000
EOF

chmod 600 /opt/paperless/.env
chown paperless:paperless /opt/paperless/.env
```

**Lien avec le cours :**  
**Variables d'environnement** (cours "Rappels sur le terminal", slide 6) :
- Le fichier `.env` contient des variables sensibles chargées au démarrage
- `chmod 600` : `rw-------` garantit que SEUL le propriétaire peut lire/écrire
- Protection des secrets : mot de passe DB, clé de chiffrement Django

**Structure des permissions 600 :**
```
6 = rw- (lecture + écriture pour le propriétaire)
0 = --- (aucun accès pour le groupe)
0 = --- (aucun accès pour les autres)
```

### 2.9. Migrations Django et collecte des fichiers statiques

```bash
source /opt/paperless/venv/bin/activate
cd /opt/paperless/src

sudo -u paperless /opt/paperless/venv/bin/python manage.py migrate --noinput
sudo -u paperless /opt/paperless/venv/bin/python manage.py collectstatic --noinput
sudo -u paperless /opt/paperless/venv/bin/python manage.py createsuperuser
sudo -u paperless /opt/paperless/venv/bin/python manage.py showmigrations

deactivate
```

**Commandes Django :**
- `migrate` : Applique les migrations (création/modification des tables PostgreSQL)
- `collectstatic` : Rassemble les fichiers CSS/JS pour la production
- `createsuperuser` : Crée le compte administrateur initial (interactif)
- `showmigrations` : Vérifie que toutes les migrations sont appliquées

### 2.10. Configuration des services systemd

#### 2.10.1. Service Web (paperless-web.service)

```ini
[Unit]
Description=Paperless-NGX Web Server
After=network.target redis-server.service postgresql.service

[Service]
User=paperless
EnvironmentFile=/opt/paperless/.env
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/gunicorn \
    --bind 0.0.0.0:8000 \
    paperless.asgi:application \
    -k uvicorn.workers.UvicornWorker
Restart=always

[Install]
WantedBy=multi-user.target
```

**🔗 Lien avec le cours :**  
Ce fichier `.service` configure un **daemon systemd** (cours "Le système d'exploitation", slide 3) :

| Directive | Signification |
|-----------|--------------|
| `After=` | Dépendances : attend réseau, Redis, PostgreSQL |
| `User=` | Exécution avec l'utilisateur système dédié |
| `EnvironmentFile=` | Charge les variables depuis `.env` |
| `Restart=always` | Redémarrage automatique en cas de crash |
| `WantedBy=multi-user.target` | Activation au démarrage du système |

#### 2.10.2. Service Consumer (paperless-consumer.service)

```ini
[Unit]
Description=Paperless-NGX Document Consumer

[Service]
User=paperless
EnvironmentFile=/opt/paperless/.env
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/python manage.py document_consumer
Restart=always

[Install]
WantedBy=multi-user.target
```

Le **consumer** surveille le dossier `consume/` et traite automatiquement les nouveaux documents :
1. Détection de nouveaux fichiers (inotify)
2. Extraction du texte via OCR (Tesseract)
3. Indexation dans PostgreSQL
4. Génération de miniatures
5. Déplacement dans `media/`

#### 2.10.3. Activation des services

```bash
systemctl daemon-reload
systemctl enable --now paperless-web.service
systemctl enable --now paperless-consumer.service
```

**🔗 Lien avec le cours :**  
- `daemon-reload` : Recharge la configuration systemd après ajout de fichiers `.service`
- `enable` : Crée un lien symbolique dans `/etc/systemd/system/multi-user.target.wants/`
- Vérification : `systemctl status paperless-web.service`

### 2.11. Validation de l'installation

**Commandes de vérification :**

```bash
# Statut des services
systemctl status paperless-web.service
systemctl status paperless-consumer.service

# Logs en temps réel
journalctl -u paperless-web.service -f

# Test d'accès
curl http://localhost:8000
```

**Accès à l'interface web :**  
`http://localhost:8000` (ou `http://<IP_SERVEUR>:8000`)

---

## 3. Backup automatisé

### 3.1. Installation de Restic et Rclone

```bash
apt update
apt install -y restic rclone
```

### 3.2. Script de backup avec Restic

```bash
#!/bin/bash
# /opt/scripts/backup_paperless.sh

set -e

RESTIC_REPOSITORY="/opt/backups/paperless-restic"
RESTIC_PASSWORD_FILE="/root/.restic_password"
BACKUP_SOURCE="/opt/paperless"

# Initialisation du dépôt (première fois uniquement)
if [ ! -d "$RESTIC_REPOSITORY" ]; then
    echo "Initialisation du dépôt Restic..."
    restic init --repo "$RESTIC_REPOSITORY" --password-file "$RESTIC_PASSWORD_FILE"
fi

# Backup PostgreSQL
echo "Backup de la base de données PostgreSQL..."
sudo -u postgres pg_dump paperless > /tmp/paperless_db_backup.sql

# Backup avec Restic
echo "Création du snapshot Restic..."
restic backup \
    --repo "$RESTIC_REPOSITORY" \
    --password-file "$RESTIC_PASSWORD_FILE" \
    --tag "paperless" \
    "$BACKUP_SOURCE/media" \
    "$BACKUP_SOURCE/data" \
    "$BACKUP_SOURCE/consume" \
    "$BACKUP_SOURCE/.env" \
    /tmp/paperless_db_backup.sql

# Nettoyage
rm -f /tmp/paperless_db_backup.sql

# Conservation : 7 derniers snapshots quotidiens, 4 hebdomadaires, 6 mensuels
restic forget \
    --repo "$RESTIC_REPOSITORY" \
    --password-file "$RESTIC_PASSWORD_FILE" \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --prune

echo "Backup terminé avec succès - $(date)"
```

**🔗 Lien avec le cours :**  
**Scripting Bash** (cours "Rappels sur le terminal", slide 6-8) :
- `set -e` : Arrête le script dès qu'une commande échoue (gestion des erreurs)
- Variables : `RESTIC_REPOSITORY`, `BACKUP_SOURCE`
- Redirection : `> /tmp/paperless_db_backup.sql` (STDOUT vers fichier)
- Flux de texte : tout est du texte pour compatibilité maximale

**Rendre le script exécutable :**

```bash
chmod +x /opt/scripts/backup_paperless.sh
```

### 3.3. Configuration de Cron

```bash
# Éditer le crontab root
crontab -e

# Ajouter la ligne suivante (backup toutes les heures)
0 * * * * /opt/scripts/backup_paperless.sh >> /var/log/paperless_backup.log 2>&1
```

**Format Cron :**
```
┌───────────── minute (0 - 59)
│ ┌───────────── heure (0 - 23)
│ │ ┌───────────── jour du mois (1 - 31)
│ │ │ ┌───────────── mois (1 - 12)
│ │ │ │ ┌───────────── jour de la semaine (0 - 6) (Dimanche = 0)
│ │ │ │ │
│ │ │ │ │
0 * * * * /opt/scripts/backup_paperless.sh
```

**🔗 Lien avec le cours :**  
`cron` est un **daemon** système (cours "Le système d'exploitation", slide 3) qui exécute des tâches planifiées. Il vérifie toutes les minutes s'il y a des commandes à lancer selon le planning défini dans les crontabs.

### 3.4. Synchronisation vers Google Drive avec Rclone

```bash
# Configuration de Rclone (interactive)
rclone config

# Nom: gdrive
# Type: Google Drive
# Suivre les instructions pour authentifier
```

**Script de synchronisation :**

```bash
#!/bin/bash
# /opt/scripts/sync_to_gdrive.sh

set -e

RESTIC_REPOSITORY="/opt/backups/paperless-restic"
REMOTE_NAME="gdrive"
REMOTE_PATH="Paperless-Backups"

echo "Synchronisation vers Google Drive - $(date)"

rclone sync \
    "$RESTIC_REPOSITORY" \
    "$REMOTE_NAME:$REMOTE_PATH" \
    --progress \
    --log-file /var/log/rclone_sync.log

echo "Synchronisation terminée - $(date)"
```

**Ajout au crontab (toutes les 6 heures) :**

```bash
0 */6 * * * /opt/scripts/sync_to_gdrive.sh >> /var/log/gdrive_sync.log 2>&1
```

### 3.5. Restauration d'un backup

```bash
# Lister les snapshots disponibles
restic snapshots --repo /opt/backups/paperless-restic --password-file /root/.restic_password

# Restaurer un snapshot spécifique
restic restore <snapshot_id> \
    --repo /opt/backups/paperless-restic \
    --password-file /root/.restic_password \
    --target /opt/paperless-restore

# Restaurer la base de données
sudo -u postgres psql paperless < /opt/paperless-restore/tmp/paperless_db_backup.sql
```

---

## 4. Sécurité réseau

### 4.1. Configuration du pare-feu avec UFW

```bash
# Installation
apt install -y ufw

# Politique par défaut : bloquer tout le trafic entrant
ufw default deny incoming
ufw default allow outgoing

# Autoriser SSH (IMPORTANT avant d'activer UFW !)
ufw allow 22/tcp

# Autoriser Paperless
ufw allow 8000/tcp

# Activation du pare-feu
ufw enable

# Vérification
ufw status verbose
```

**🔗 Lien avec le cours :**  
Le pare-feu filtre le trafic réseau au niveau du **kernel Linux** (cours "Fonctionnement d'un ordinateur"). UFW (Uncomplicated Firewall) est une interface simplifiée pour `iptables`, le système de filtrage de paquets du noyau.

**Principe de sécurité :** "Deny by default, allow by exception"
- ❌ Tout est bloqué par défaut
- ✅ Seuls les ports nécessaires sont ouverts

### 4.2. Configuration de Fail2Ban

#### 4.2.1. Installation

```bash
apt install -y fail2ban
```

#### 4.2.2. Configuration pour Paperless

```bash
# /etc/fail2ban/filter.d/paperless.conf
[Definition]
failregex = ^.*Failed login attempt.*from <HOST>.*$
            ^.*"POST /api/token/ HTTP.*" (401|403).*$
ignoreregex =
```

```bash
# /etc/fail2ban/jail.d/paperless.conf
[paperless]
enabled = true
port = 8000
filter = paperless
logpath = /var/log/paperless/paperless.log
maxretry = 5
bantime = 3600
findtime = 600
action = iptables-multiport[name=paperless, port="8000", protocol=tcp]
```

**🔗 Lien avec le cours :**  
Fail2Ban analyse les **logs système** (cours "Le système d'exploitation") via des expressions régulières pour détecter :
- **Bruteforce** : 5 tentatives de connexion échouées en 10 minutes
- **Énumération web** : Tentatives d'accès à des URLs inexistantes

**Action :** Bannissement de l'IP pendant 1 heure via `iptables`

#### 4.2.3. Filtre pour énumération web

```bash
# /etc/fail2ban/filter.d/paperless-enum.conf
[Definition]
failregex = ^<HOST>.*"(GET|POST).*HTTP.*" 404.*$
ignoreregex =
```

```bash
# Ajouter dans /etc/fail2ban/jail.d/paperless.conf
[paperless-enum]
enabled = true
port = 8000
filter = paperless-enum
logpath = /var/log/nginx/access.log
maxretry = 10
bantime = 7200
findtime = 300
action = iptables-multiport[name=paperless-enum, port="8000", protocol=tcp]
```

#### 4.2.4. Activation et vérification

```bash
systemctl enable --now fail2ban
systemctl status fail2ban

# Voir les IPs bannies
fail2ban-client status paperless

# Débannir une IP
fail2ban-client set paperless unbanip <IP>
```

### 4.3. Renforcement supplémentaire

#### 4.3.1. Désactivation de root SSH

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no

# Redémarrage SSH
systemctl restart sshd
```

#### 4.3.2. Configuration d'un reverse proxy Nginx (optionnel)

```nginx
# /etc/nginx/sites-available/paperless
server {
    listen 80;
    server_name paperless.example.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Limitation du taux de requêtes
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
    limit_req zone=one burst=20 nodelay;
}
```

**Avantages du reverse proxy :**
- ✅ HTTPS/TLS (avec Let's Encrypt)
- ✅ Limitation de taux (rate limiting)
- ✅ Compression gzip
- ✅ Cache des fichiers statiques

---

## 5. Monitoring

### 5.1. Installation de Prometheus et Grafana

#### 5.1.1. Installation de Prometheus

```bash
# Téléchargement
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
mv prometheus-2.45.0.linux-amd64 /opt/prometheus

# Création utilisateur
adduser --system --group --no-create-home prometheus

# Configuration
cat <<EOF > /opt/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'paperless'
    static_configs:
      - targets: ['localhost:8000']
EOF

# Service systemd
cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
    --config.file=/opt/prometheus/prometheus.yml \
    --storage.tsdb.path=/opt/prometheus/data \
    --web.listen-address=127.0.0.1:9090

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now prometheus
```

**🔗 Lien avec le cours :**  
`--web.listen-address=127.0.0.1:9090` écoute uniquement sur **localhost** (cours "Sécurité"). Le service n'est pas exposé sur le réseau externe, on y accède via **port forwarding SSH**.

#### 5.1.2. Installation de Node Exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
mv node_exporter-1.6.0.linux-amd64/node_exporter /usr/local/bin/

# Service systemd
cat <<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
```

**Métriques collectées :**
- CPU, RAM, disque
- I/O réseau
- Processus système

#### 5.1.3. Installation de Grafana

```bash
apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | tee /etc/apt/sources.list.d/grafana.list

apt update
apt install -y grafana

# Configuration pour localhost uniquement
sed -i 's/;http_addr =/http_addr = 127.0.0.1/' /etc/grafana/grafana.ini

systemctl enable --now grafana-server
```

### 5.2. Accès sécurisé via Port Forwarding SSH

```bash
# Depuis votre machine locale
ssh -L 3000:localhost:3000 -L 9090:localhost:9090 user@<IP_SERVEUR>

# Accès web :
# Grafana : http://localhost:3000 (admin / admin par défaut)
# Prometheus : http://localhost:9090


