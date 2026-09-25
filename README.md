# Instalasi-Odoo-14-With-Docker
1. Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin git curl

Aktifkan Docker:

sudo systemctl enable --now docker

Cek:

docker --version
docker compose version

Tambahkan user ubuntu ke Docker:

sudo usermod -aG docker $USER

Logout/login SSH lagi, kemudian:

docker ps
2. Buat folder Odoo
sudo mkdir -p /mnt/storage/odoo14
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14

Masuk:

cd /mnt/storage/odoo14

Buat struktur:

mkdir -p config
mkdir -p addons
mkdir -p custom_addons
mkdir -p odoo-data
mkdir -p postgres
mkdir -p scripts

Cek:

tree -L 2 /mnt/storage/odoo14

Kalau tree belum ada:

sudo apt install -y tree
3. Buat Dockerfile
nano /mnt/storage/odoo14/Dockerfile

Paste semua ini:

FROM python:3.8-slim-bullseye

# ---------------------------------------------------------
# Debian Bullseye snapshot
# ---------------------------------------------------------

RUN printf '%s\n' \
    'deb http://snapshot.debian.org/archive/debian/20240926T000000Z bullseye main' \
    'deb http://snapshot.debian.org/archive/debian-security/20240926T000000Z bullseye-security main' \
    'deb http://snapshot.debian.org/archive/debian/20240926T000000Z bullseye-updates main' \
    > /etc/apt/sources.list && \
    printf 'Acquire::Check-Valid-Until "false";\n' \
    > /etc/apt/apt.conf.d/99snapshot

# ---------------------------------------------------------
# Environment
# ---------------------------------------------------------

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# ---------------------------------------------------------
# System dependencies
# ---------------------------------------------------------

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        gnupg \
        git \
        build-essential \
        gcc \
        g++ \
        wkhtmltopdf \
        make \
        libpq-dev \
        libldap2-dev \
        libsasl2-dev \
        libxml2-dev \
        libxslt1-dev \
        libjpeg62-turbo-dev \
        zlib1g-dev \
        libffi-dev \
        libssl-dev \
        libfreetype6-dev \
        liblcms2-dev \
        libopenjp2-7-dev \
        libtiff5-dev \
        libwebp-dev \
        libharfbuzz-dev \
        libfribidi-dev \
        libxcb1 \
        libx11-6 \
        libxext6 \
        libxrender1 \
        xfonts-75dpi \
        xfonts-base \
        fonts-dejavu \
        node-less \
        npm \
    && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------
# PostgreSQL client 14
# Must match PostgreSQL server 14
# ---------------------------------------------------------

RUN echo "deb [signed-by=/usr/share/keyrings/postgresql-archive-keyring.gpg] http://apt.postgresql.org/pub/repos/apt bullseye-pgdg main" \
    > /etc/apt/sources.list.d/pgdg.list && \
    curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
    | gpg --dearmor -o /usr/share/keyrings/postgresql-archive-keyring.gpg && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        postgresql-client-14 \
    && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------
# Odoo 14
# ---------------------------------------------------------

RUN git clone \
    --depth 1 \
    --branch 14.0 \
    https://github.com/odoo/odoo.git \
    /opt/odoo

WORKDIR /opt/odoo

# ---------------------------------------------------------
# Python dependencies
# ---------------------------------------------------------

RUN python -m pip install --no-cache-dir --upgrade \
    pip \
    setuptools \
    wheel

RUN pip install --upgrade \
    "pip<24.1" \
    "setuptools<70" \
    wheel \
    "Cython<3"

RUN pip install --no-cache-dir \
    "Cython==0.29.21"

RUN pip install \
    --no-cache-dir \
    --no-build-isolation \
    -r requirements.txt

# ---------------------------------------------------------
# RTL CSS
# ---------------------------------------------------------

RUN npm install -g rtlcss

# ---------------------------------------------------------
# Odoo directories
# ---------------------------------------------------------

RUN mkdir -p \
    /var/lib/odoo \
    /mnt/extra-addons \
    /mnt/custom-addons \
    /etc/odoo

# ---------------------------------------------------------
# Odoo user
# ---------------------------------------------------------

RUN useradd \
    --system \
    --home /var/lib/odoo \
    --shell /bin/bash \
    odoo

RUN chown -R odoo:odoo \
    /var/lib/odoo \
    /opt/odoo \
    /mnt/extra-addons \
    /mnt/custom-addons

# ---------------------------------------------------------
# Configuration
# ---------------------------------------------------------

COPY config/odoo.conf /etc/odoo/odoo.conf

RUN chown odoo:odoo /etc/odoo/odoo.conf

# ---------------------------------------------------------
# Runtime
# ---------------------------------------------------------

USER odoo

EXPOSE 8069

CMD [
    "python3",
    "/opt/odoo/odoo-bin",
    "-c",
    "/etc/odoo/odoo.conf"
]

Save:

Ctrl + O
Enter
Ctrl + X
4. Buat docker-compose.yml
nano /mnt/storage/odoo14/docker-compose.yml

Paste:

services:

  db:
    image: postgres:14
    container_name: odoo14-db
    restart: unless-stopped

    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo14
      POSTGRES_PASSWORD: odoo

    volumes:
      - ./postgres:/var/lib/postgresql/data

  odoo:
    build:
      context: .
      dockerfile: Dockerfile

    image: odoo14-odoo
    container_name: odoo14
    restart: unless-stopped

    depends_on:
      - db

    ports:
      - "8069:8069"

    environment:
      HOST: db
      USER: odoo14
      PASSWORD: odoo

    volumes:
      - ./odoo-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
      - ./custom_addons:/mnt/custom-addons

Save:

Ctrl + O
Enter
Ctrl + X
5. Buat odoo.conf
nano /mnt/storage/odoo14/config/odoo.conf

Paste:

[options]

admin_passwd = CHANGE_THIS_MASTER_PASSWORD

db_host = db
db_port = 5432
db_user = odoo14
db_password = odoo

list_db = True

addons_path = /opt/odoo/addons,/mnt/extra-addons,/mnt/custom-addons

proxy_mode = True

server_wide_modules = web

logfile = /var/lib/odoo/odoo.log

workers = 0

limit_time_cpu = 600
limit_time_real = 1200

limit_memory_hard = 2684354560
limit_memory_soft = 2147483648

Save:

Ctrl + O
Enter
Ctrl + X
Ganti master password

Jangan commit password production ke GitHub.

Misalnya sebelum deployment:

nano /mnt/storage/odoo14/config/odoo.conf

ubah:

admin_passwd = CHANGE_THIS_MASTER_PASSWORD

menjadi password master yang kamu inginkan.

6. Permission

Untuk folder project:

sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14

PostgreSQL container menggunakan UID 999 pada image ini.

sudo chown -R 999:999 /mnt/storage/odoo14/postgres

Odoo container menggunakan user odoo.

Di Dockerfile user odoo dibuat sebagai system user, yang biasanya UID pertama yang tersedia setelah base image. Lebih aman cek UID image setelah build, jadi jangan menebak UID sekarang.

Untuk awal:

sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/odoo-data
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/addons
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/custom_addons
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/config
7. Build Odoo

Masuk project:

cd /mnt/storage/odoo14

Build:

docker compose build --no-cache

Ini bisa cukup lama.

Setelah selesai:

docker images | grep odoo14
8. Jalankan PostgreSQL + Odoo
docker compose up -d

Cek:

docker compose ps

Target:

odoo14-db    running
odoo14       running
9. Cek PostgreSQL
docker exec -it odoo14-db psql -U odoo14 -d postgres -c "SELECT version();"

Harus menunjukkan PostgreSQL 14.x.

10. Cek pg_dump

Ini penting untuk backup.

docker exec -it odoo14 pg_dump --version

Target:

pg_dump (PostgreSQL) 14.x

Jangan sampai:

pg_dump 13.x

karena PostgreSQL server kita 14.

11. Cek koneksi Odoo → PostgreSQL
docker exec -it odoo14 getent hosts db

Kemudian:

docker exec -e PGPASSWORD=odoo -it odoo14 \
psql -h db -U odoo14 -d postgres \
-c "SELECT version();"
12. Cek Odoo
curl -I http://127.0.0.1:8069/web

Biasanya akan mendapat:

HTTP/1.0 303 SEE OTHER

atau response HTTP Odoo lainnya.

Cek log:

docker logs --tail 100 odoo14

Live:

docker logs -f odoo14

Keluar:

Ctrl + C
13. Permission addons

Pastikan:

sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/addons
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/custom_addons

Kemudian:

find /mnt/storage/odoo14/custom_addons -type d -exec chmod 755 {} \;
find /mnt/storage/odoo14/custom_addons -type f -exec chmod 644 {} \;

Sama untuk addons:

find /mnt/storage/odoo14/addons -type d -exec chmod 755 {} \;
find /mnt/storage/odoo14/addons -type f -exec chmod 644 {} \;
14. Test custom addon

Misalnya:

custom_addons/
└── odoo_home_screen_14/
    ├── __init__.py
    ├── __manifest__.py
    └── ...

Masuk container:

docker exec -it odoo14 bash

Cek:

ls -lah /mnt/custom-addons

Cek Odoo:

grep addons_path /etc/odoo/odoo.conf

Harus:

/opt/odoo/addons,/mnt/extra-addons,/mnt/custom-addons

Keluar:

exit
