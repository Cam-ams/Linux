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

Script : 
`````
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
```

# Conclusion
Ce projet nous a permis d’appliquer les notions essentielles du module Linux : gestion des services, permissions, sécurité, scripts Bash, environnement système et maintenance.
Grâce à ces connaissances, nous avons pu créer un script complet capable d’automatiser l’installation et la configuration de Paperless sur Debian 13.
Ce travail montre l’importance de l’automatisation pour faciliter les déploiements, rendre les installations plus fiables et garantir la reproductibilité des environnements.
Il met aussi en évidence le rôle des bonnes pratiques d’administration système pour sécuriser et organiser un service.
En résumé, ce projet nous a permis de comprendre comment combiner théorie et pratique pour installer, sécuriser et gérer un service réel de manière entièrement automatique. C’est une étape importante pour apprendre à administrer un système Linux de façon professionnelle.





