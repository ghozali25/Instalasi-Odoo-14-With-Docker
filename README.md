# Instalasi Odoo 14 With Docker

Panduan lengkap instalasi **Odoo 14 + PostgreSQL 14 menggunakan Docker**.

Target struktur:

```text
/mnt/storage/odoo14/
├── Dockerfile
├── docker-compose.yml
├── config/
│   └── odoo.conf
├── addons/
├── custom_addons/
├── odoo-data/
├── postgres/
└── scripts/
```

---

# 1. Install Docker

Update package:

```bash
sudo apt update
```

Install Docker, Docker Compose Plugin, Git, dan Curl:

```bash
sudo apt install -y docker.io docker-compose-plugin git curl
```

Aktifkan Docker:

```bash
sudo systemctl enable --now docker
```

Cek Docker:

```bash
docker --version
```

Cek Docker Compose:

```bash
docker compose version
```

Tambahkan user `ubuntu` ke group Docker:

```bash
sudo usermod -aG docker $USER
```

**Logout dan login SSH kembali** agar perubahan group aktif.

Kemudian test:

```bash
docker ps
```

Jika tidak ada error permission, Docker sudah siap.

---

# 2. Buat Folder Odoo

Buat folder project:

```bash
sudo mkdir -p /mnt/storage/odoo14
```

Set ownership:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14
```

Masuk ke folder project:

```bash
cd /mnt/storage/odoo14
```

Buat struktur folder:

```bash
mkdir -p config
```

```bash
mkdir -p addons
```

```bash
mkdir -p custom_addons
```

```bash
mkdir -p odoo-data
```

```bash
mkdir -p postgres
```

```bash
mkdir -p scripts
```

Install `tree` jika belum tersedia:

```bash
sudo apt install -y tree
```

Cek struktur:

```bash
tree -L 2 /mnt/storage/odoo14
```

Hasil awal kira-kira:

```text
/mnt/storage/odoo14
├── addons
├── config
├── custom_addons
├── odoo-data
├── postgres
└── scripts
```

---

# 3. Buat Dockerfile

Buat file:

```bash
nano /mnt/storage/odoo14/Dockerfile
```

Paste seluruh isi berikut:

```dockerfile
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
```

Simpan:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 4. Buat docker-compose.yml

Buat file:

```bash
nano /mnt/storage/odoo14/docker-compose.yml
```

Paste:

```yaml
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
```

Simpan:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 5. Buat odoo.conf

Buat file:

```bash
nano /mnt/storage/odoo14/config/odoo.conf
```

Paste:

```ini
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
```

Simpan:

```text
Ctrl + O
Enter
Ctrl + X
```

## Ganti Master Password

Jangan menggunakan:

```ini
admin_passwd = CHANGE_THIS_MASTER_PASSWORD
```

untuk production.

Edit:

```bash
nano /mnt/storage/odoo14/config/odoo.conf
```

Contoh:

```ini
admin_passwd = PASSWORD_MASTER_KAMU
```

**Jangan commit password production ke GitHub.**

---

# 6. Permission

Set ownership project:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14
```

PostgreSQL container menggunakan UID `999` pada image ini:

```bash
sudo chown -R 999:999 /mnt/storage/odoo14/postgres
```

Untuk Odoo:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/odoo-data
```

Addon:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/addons
```

Custom addon:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/custom_addons
```

Config:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/config
```

> UID user `odoo` di dalam image tidak perlu ditebak dari host. User `odoo` dibuat ketika Docker image dibuild.

---

# 7. Build Odoo

Masuk project:

```bash
cd /mnt/storage/odoo14
```

Build image:

```bash
docker compose build --no-cache
```

Proses build dapat membutuhkan waktu cukup lama.

Setelah selesai:

```bash
docker images | grep odoo14
```

Target:

```text
odoo14-odoo
```

---

# 8. Jalankan PostgreSQL + Odoo

Jalankan:

```bash
cd /mnt/storage/odoo14
```

Kemudian:

```bash
docker compose up -d
```

Cek:

```bash
docker compose ps
```

Target:

```text
odoo14-db    running
odoo14       running
```

---

# 9. Cek PostgreSQL

Cek versi PostgreSQL:

```bash
docker exec -it odoo14-db psql -U odoo14 -d postgres -c "SELECT version();"
```

Harus menunjukkan:

```text
PostgreSQL 14.x
```

---

# 10. Cek pg_dump

Backup database membutuhkan `pg_dump`.

Cek versi:

```bash
docker exec -it odoo14 pg_dump --version
```

Target:

```text
pg_dump (PostgreSQL) 14.x
```

Pastikan bukan:

```text
pg_dump (PostgreSQL) 13.x
```

atau versi PostgreSQL yang lebih lama.

Client PostgreSQL di image Odoo sengaja menggunakan PostgreSQL 14 agar sesuai dengan PostgreSQL server.

---

# 11. Cek Koneksi Odoo → PostgreSQL

Cek DNS/container network:

```bash
docker exec -it odoo14 getent hosts db
```

Kemudian test koneksi PostgreSQL dari container Odoo:

```bash
docker exec -e PGPASSWORD=odoo -it odoo14 \
psql -h db -U odoo14 -d postgres \
-c "SELECT version();"
```

Jika PostgreSQL version muncul, koneksi Odoo → PostgreSQL berhasil.

---

# 12. Cek Odoo

Test port Odoo:

```bash
curl -I http://127.0.0.1:8069/web
```

Response yang mungkin:

```text
HTTP/1.0 303 SEE OTHER
```

atau response HTTP Odoo lainnya.

Cek container:

```bash
docker compose ps
```

Cek log:

```bash
docker logs --tail 100 odoo14
```

Untuk melihat log secara live:

```bash
docker logs -f odoo14
```

Keluar dari live log:

```text
Ctrl + C
```

---

# 13. Permission Addons

Set ownership:

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/addons
```

```bash
sudo chown -R ubuntu:ubuntu /mnt/storage/odoo14/custom_addons
```

Permission directory custom addons:

```bash
find /mnt/storage/odoo14/custom_addons -type d -exec chmod 755 {} \;
```

Permission file custom addons:

```bash
find /mnt/storage/odoo14/custom_addons -type f -exec chmod 644 {} \;
```

Permission directory addons:

```bash
find /mnt/storage/odoo14/addons -type d -exec chmod 755 {} \;
```

Permission file addons:

```bash
find /mnt/storage/odoo14/addons -type f -exec chmod 644 {} \;
```

---

# 14. Test Custom Addon

Contoh struktur:

```text
custom_addons/
└── odoo_home_screen_14/
    ├── __init__.py
    ├── __manifest__.py
    └── ...
```

Cek folder custom addons dari host:

```bash
ls -lah /mnt/storage/odoo14/custom_addons
```

Masuk ke container Odoo:

```bash
docker exec -it odoo14 bash
```

Cek custom addons:

```bash
ls -lah /mnt/custom-addons
```

Cek addons path:

```bash
grep addons_path /etc/odoo/odoo.conf
```

Harus menunjukkan:

```text
addons_path = /opt/odoo/addons,/mnt/extra-addons,/mnt/custom-addons
```

Keluar dari container:

```bash
exit
```

---

# 15. Restart Odoo

Restart semua service:

```bash
cd /mnt/storage/odoo14
docker compose restart
```

Cek:

```bash
docker compose ps
```

---

# 16. Stop Odoo

Stop container:

```bash
cd /mnt/storage/odoo14
docker compose stop
```

Cek:

```bash
docker compose ps
```

---

# 17. Start Odoo Kembali

Start container:

```bash
cd /mnt/storage/odoo14
docker compose start
```

Cek:

```bash
docker compose ps
```

---

# 18. Update Custom Addon

Setelah menambahkan atau mengubah addon:

```bash
cd /mnt/storage/odoo14
```

Restart Odoo:

```bash
docker compose restart odoo
```

Jika membutuhkan upgrade module dari command line:

```bash
docker exec -it odoo14 \
python3 /opt/odoo/odoo-bin \
-c /etc/odoo/odoo.conf \
-d NAMA_DATABASE \
-u NAMA_MODULE \
--stop-after-init
```

Contoh:

```bash
docker exec -it odoo14 \
python3 /opt/odoo/odoo-bin \
-c /etc/odoo/odoo.conf \
-d HLB \
-u odoo_home_screen_14 \
--stop-after-init
```

Kemudian restart:

```bash
docker compose restart odoo
```

---

# 19. Cek Semua Container

```bash
docker ps
```

Atau:

```bash
docker compose ps
```

---

# 20. Cek Penggunaan Docker

```bash
docker system df
```

Cek penggunaan disk:

```bash
df -h
```

Cek folder project:

```bash
du -sh /mnt/storage/odoo14/*
```

---

# 21. Backup Database dan Filestore

Backup harus mencakup:

1. PostgreSQL database
2. Odoo filestore

Jangan hanya melakukan backup PostgreSQL.

Buat folder backup:

```bash
mkdir -p /mnt/storage/odoo14/backups
```

Contoh backup database:

```bash
docker exec odoo14-db pg_dump \
-U odoo14 \
-F c \
-d NAMA_DATABASE \
> /mnt/storage/odoo14/backups/NAMA_DATABASE_$(date +%Y-%m-%d).dump
```

Contoh:

```bash
docker exec odoo14-db pg_dump \
-U odoo14 \
-F c \
-d HLB \
> /mnt/storage/odoo14/backups/HLB_$(date +%Y-%m-%d).dump
```

Backup filestore:

```bash
tar -czf \
/mnt/storage/odoo14/backups/HLB_filestore_$(date +%Y-%m-%d).tar.gz \
-C /mnt/storage/odoo14/odoo-data \
filestore
```

> Untuk backup production, gunakan script backup otomatis dan retention. Jangan mengandalkan backup manual.

---

# 22. Restore Database

Stop Odoo terlebih dahulu:

```bash
cd /mnt/storage/odoo14
docker compose stop odoo
```

Restore database menggunakan:

```bash
cat backup.dump | docker exec -i odoo14-db \
pg_restore \
-U odoo14 \
-d NAMA_DATABASE \
--clean \
--if-exists
```

Contoh:

```bash
cat HLB_backup.dump | docker exec -i odoo14-db \
pg_restore \
-U odoo14 \
-d HLB \
--clean \
--if-exists
```

Start Odoo:

```bash
docker compose start odoo
```

Cek:

```bash
docker compose ps
```

---

# 23. Rebuild Docker Image

Jika `Dockerfile` berubah:

```bash
cd /mnt/storage/odoo14
```

Build ulang:

```bash
docker compose build
```

Kemudian:

```bash
docker compose up -d
```

Jika ingin build dari awal tanpa cache:

```bash
docker compose build --no-cache
```

Kemudian:

```bash
docker compose up -d
```

---

# 24. Update Odoo Source

Dockerfile menggunakan:

```dockerfile
git clone \
    --depth 1 \
    --branch 14.0 \
    https://github.com/odoo/odoo.git \
    /opt/odoo
```

Jika ingin mengambil source Odoo 14 terbaru dari branch `14.0`, rebuild image:

```bash
cd /mnt/storage/odoo14
```

```bash
docker compose build --no-cache
```

Kemudian:

```bash
docker compose up -d
```

---

# 25. Useful Docker Commands

Melihat semua container:

```bash
docker ps -a
```

Melihat image:

```bash
docker images
```

Melihat log Odoo:

```bash
docker logs odoo14
```

Melihat log PostgreSQL:

```bash
docker logs odoo14-db
```

Masuk container Odoo:

```bash
docker exec -it odoo14 bash
```

Masuk PostgreSQL:

```bash
docker exec -it odoo14-db bash
```

Masuk PostgreSQL menggunakan psql:

```bash
docker exec -it odoo14-db \
psql -U odoo14 -d postgres
```

Keluar:

```text
\q
```

---

# 26. Final Checklist

Setelah instalasi selesai, jalankan:

```bash
cd /mnt/storage/odoo14
```

Cek Docker:

```bash
docker --version
```

Cek Compose:

```bash
docker compose version
```

Cek container:

```bash
docker compose ps
```

Cek PostgreSQL:

```bash
docker exec -it odoo14-db \
psql -U odoo14 -d postgres \
-c "SELECT version();"
```

Cek pg_dump:

```bash
docker exec -it odoo14 pg_dump --version
```

Cek koneksi Odoo → PostgreSQL:

```bash
docker exec -e PGPASSWORD=odoo -it odoo14 \
psql -h db -U odoo14 -d postgres \
-c "SELECT version();"
```

Cek Odoo:

```bash
curl -I http://127.0.0.1:8069/web
```

Cek log:

```bash
docker logs --tail 100 odoo14
```

Cek addons:

```bash
docker exec -it odoo14 \
ls -lah /mnt/custom-addons
```

Cek addons path:

```bash
docker exec -it odoo14 \
grep addons_path /etc/odoo/odoo.conf
```

Target:

```text
/opt/odoo/addons,/mnt/extra-addons,/mnt/custom-addons
```

Jika seluruh pengecekan berhasil, Odoo 14 sudah siap digunakan.

---

# 27. Struktur Final

Struktur project yang diharapkan:

```text
/mnt/storage/odoo14/
├── Dockerfile
├── docker-compose.yml
│
├── config/
│   └── odoo.conf
│
├── addons/
│
├── custom_addons/
│   └── odoo_home_screen_14/
│       ├── __init__.py
│       ├── __manifest__.py
│       └── ...
│
├── odoo-data/
│   ├── filestore/
│   └── odoo.log
│
├── postgres/
│
├── scripts/
│
└── backups/
```

---

# Important

## Jangan commit password production

File berikut mengandung credential:

```text
config/odoo.conf
```

Pastikan password production tidak dimasukkan ke GitHub.

Gunakan placeholder seperti:

```ini
admin_passwd = CHANGE_THIS_MASTER_PASSWORD
db_password = CHANGE_THIS_DATABASE_PASSWORD
```

## Jangan commit data production

Jangan upload folder berikut ke repository:

```text
postgres/
odoo-data/
backups/
```

## Jangan commit secrets

Tambahkan `.gitignore`:

```bash
nano /mnt/storage/odoo14/.gitignore
```

Isi:

```gitignore
# Odoo runtime data
odoo-data/

# PostgreSQL data
postgres/

# Backups
backups/

# Python cache
__pycache__/
*.pyc

# Logs
*.log

# Environment / secrets
.env

# OS files
.DS_Store
Thumbs.db
```

Simpan:

```text
Ctrl + O
Enter
Ctrl + X
```

Cek:

```bash
cat /mnt/storage/odoo14/.gitignore
```

---

# Git Repository

Setelah project siap:

```bash
cd /mnt/storage/odoo14
```

Inisialisasi Git:

```bash
git init
```

Tambahkan file:

```bash
git add Dockerfile docker-compose.yml config/odoo.conf .gitignore
```

Cek:

```bash
git status
```

Buat commit:

```bash
git commit -m "Initial Odoo 14 Docker setup"
```

Tambahkan remote GitHub:

```bash
git remote add origin https://github.com/USERNAME/REPOSITORY.git
```

Rename branch:

```bash
git branch -M main
```

Push:

```bash
git push -u origin main
```

> Pastikan `config/odoo.conf` sudah menggunakan password placeholder sebelum melakukan `git push`.
