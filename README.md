# TUGAS PROYEK TERINTEGRASI: Distributed Database, SSO, dan Inference Model

**Mata Kuliah:** Sistem Basis Data Terdistribusi  
**Nama Mahasiswa:** Feri Bagus Riadi  
**NIM:** 20230801232  
**Repositori GitHub:** https://github.com/feri-hub/sistem-basis-data-terdistribusi  
**Bobot:** 40%  
**Fokus:** **Tahap 1: Kontainerisasi Layanan (Docker & Local Orchestration)**

---

## 📌 Ringkasan Arsitektur Proyek

Proyek ini mengimplementasikan sistem microservice terdistribusi modern yang mengintegrasikan:
1. **Distributed Relational Database (PostgreSQL 16)**:
   - **Primary Node (Write)**: Menangani semua operasi tulis (`INSERT`, `UPDATE`, `DELETE`) dan memancarkan write-ahead logs (WAL).
   - **Standby Read-Replica Node (Read)**: Menangani operasi baca (`SELECT`) yang disinkronisasi melalui *Physical Streaming Replication* secara asinkron menggunakan protokol internal PostgreSQL (`pg_basebackup` & WAL streaming).
2. **Centralized Identity Provider / Single Sign-On (SSO)**:
   - Menggunakan **Keycloak 24** berbasis protokol standar **OAuth2 / OpenID Connect (OIDC)**.
   - Realm otomatis diimpor saat inisialisasi (`kapita-realm`) dengan konfigurasi Client ID, Roles (`admin`, `student`, `user`), serta akun pengguna siap pakai.
3. **Machine Learning Inference Engine & Microservice API**:
   - Dibangun menggunakan **FastAPI (Python)** dengan arsitektur asynchronous.
   - Memuat model *Machine Learning* untuk klasifikasi (Iris Classification / Tabular Prediction) menggunakan `scikit-learn` & `joblib`.
   - Mengimplementasikan pola **CQRS (Read/Write Separation)**:
     - Endpoint `POST /api/v1/predict` (membutuhkan JWT Token SSO): Melakukan inferensi ML, mencatat riwayat prediksi ke **Primary Database Node**.
     - Endpoint `GET /api/v1/predictions` (membutuhkan JWT Token SSO): Mengambil riwayat prediksi langsung dari **Replica Database Node**.
4. **Multi-Stage Dockerfile**:
   - Memisahkan stage kompilasi/build dependencies dengan stage runtime minimal untuk menghasilkan image yang ramping, efisien, dan aman (*non-root user*).

---

## 🏗️ Diagram Arsitektur & Alur Data

```
                     +---------------------------------------+
                     |         Pengguna / Klien API         |
                     +---------------------------------------+
                        |                                |
         1. Request JWT |                   3. Request   |
            Credentials |                      Inference | + Bearer Token
                        v                                v
         +-----------------------------+     +-------------------------------+
         |     Keycloak SSO (OIDC)     |     |      FastAPI Microservice     |
         |         Port: 8080          |     |          Port: 8000           |
         |  (OAuth2 Provider / Realm)  |     |  - Verifikasi Token OIDC      |
         +-----------------------------+     |  - Engine Inferensi ML        |
                        ^                    +-------------------------------+
                        | Validasi Key (JWKS)       |                |
                        +---------------------------+                |
                                            4. WRITE                 | 5. READ
                                               (POST /predict)       |    (GET /predictions)
                                                    |                |
                                                    v                v
                        +-------------------------------+   +-------------------------------+
                        |   PostgreSQL Primary Node     |   |   PostgreSQL Replica Node     |
                        |          Port: 5432           |   |          Port: 5433           |
                        |      (Write Operations)       |   |       (Read Operations)       |
                        +-------------------------------+   +-------------------------------+
                                        |                                   ^
                                        |===== WAL Streaming Replication ===|
                                               (Asynchronous Sync)
```

---

## 📁 Struktur Direktori Proyek

```
kapita/
├── docker-compose.yml             # Konfigurasi orkestrasi 4 layanan kontainer
├── .env.example                   # Contoh konfigurasi environment variables
├── .env                           # Konfigurasi environment lokal aktif
├── .gitignore                     # Filter berkas build/cache
├── test_system.py                 # Skrip Python pengujian otomatis end-to-end
├── README.md                      # Dokumentasi komprehensif proyek
│
├── database/                      # Konfigurasi Distributed Database
│   ├── primary/
│   │   ├── 01-init-replication.sh # Skrip inisialisasi user replicator & pg_hba.conf
│   │   └── 02-init-schema.sql     # Skrip DDL tabel predictions & indeks
│   └── replica/
│       └── entrypoint-replica.sh  # Entrypoint replika (pg_basebackup & standby)
│
├── keycloak/                      # Konfigurasi Identity Provider (SSO)
│   └── realm-config.json          # Konfigurasi realm, client, user, dan role
│
└── api/                           # Backend Microservice & Model Inferensi
    ├── Dockerfile                 # Multi-stage Dockerfile
    ├── requirements.txt           # Dependensi Python
    ├── train_model.py             # Skrip pelatihan & ekspor model ML
    ├── model/
    │   └── model.joblib           # Model ML ter-serialisasi
    └── app/
        ├── config.py              # Konfigurasi pydantic settings
        ├── database.py            # Dual Engine SQLAlchemy (Primary & Replica)
        ├── auth.py                # Verifikasi token JWT Keycloak (JWKS)
        ├── ml_model.py            # Engine inferensi model ML
        ├── schemas.py             # Pydantic schemas (Request & Response)
        ├── main.py                # FastAPI entrypoint & middleware
        └── routes/
            ├── health.py          # GET /health
            ├── auth.py            # POST /api/v1/auth/token & GET /me
            ├── db_status.py       # GET /api/v1/db/status (Cek Replikasi)
            └── inference.py       # POST /api/v1/predict & GET /api/v1/predictions
```

---

## 🔑 Kredensial Default

### 1. Keycloak SSO
- **Konsol Admin**: `http://localhost:8080/admin`
  - **Username:** `admin`
  - **Password:** `admin`
- **Realm Name:** `kapita-realm`
- **Client ID:** `api-client`
- **Client Secret:** `api-client-secret-12345`
- **Akun Uji Coba Mahasiswa:**
  - **Username:** `student`
  - **Password:** `student123`
  - **Role:** `student`, `user`

### 2. Database PostgreSQL
- **Primary Host:** `localhost:5432` (Internal: `postgres-primary:5432`)
- **Replica Host:** `localhost:5433` (Internal: `postgres-replica:5432`)
- **Database:** `app_db`
- **User Aplikasi:** `app_user` / `app_password`
- **User Replikasi:** `replicator` / `replica_password`

---

## 🚀 Panduan Menjalankan Sistem

### Langkah 1: Pastikan Docker & Docker Compose Aktif
Pastikan Docker Desktop sudah berjalan di perangkat Anda:
```bash
docker --version
docker compose version
```

### Langkah 2: Jalankan Cluster Menggunakan Docker Compose
Di direktori proyek (`c:\Users\ekal\Downloads\tugas\kapita`), jalankan:
```bash
docker compose up --build -d
```

### Langkah 3: Periksa Status Kontainer
Pastikan ke-4 kontainer berjalan dalam kondisi **healthy**:
```bash
docker compose ps
```
Hasil yang diharapkan:
- `postgres-primary`: `Up (healthy)`
- `postgres-replica`: `Up (healthy)`
- `keycloak-sso`: `Up (healthy)`
- `api-service`: `Up (healthy)`

---

## 🧪 Pengujian & Verifikasi Terintegrasi

Anda dapat memverifikasi seluruh komponen secara otomatis dengan menjalankan skrip uji bawaan:
```bash
python test_system.py
```

Skrip ini akan memvalidasi secara sekuensial:
1. `GET /health` -> Memastikan konektivitas API, Primary DB, Replica DB, dan Keycloak.
2. `GET /api/v1/db/status` -> Memvalidasi `pg_is_in_recovery()` (`false` di Primary, `true` di Replica) dan koneksi *streaming replication*.
3. `POST /api/v1/auth/token` -> Login via Keycloak SSO dan mendapatkan JWT Bearer Token.
4. `POST /api/v1/predict` -> Eksekusi inferensi ML dan penulisan data ke Primary DB.
5. `GET /api/v1/predictions` -> Memastikan data yang baru ditulis langsung tersinkron dan terbaca dari Read-Replica DB.

---

## 🔍 Verifikasi Manual Komponen (Bukti Laporan)

### 1. Bukti Replikasi Basis Data PostgreSQL
Jalankan perintah SQL berikut untuk melihat status replikasi streaming aktif pada **Primary Node**:
```bash
docker exec -it postgres-primary psql -U app_user -d app_db -c "SELECT client_addr, state, sync_state, sync_priority FROM pg_stat_replication;"
```
Output akan menampilkan baris koneksi aktif dari node replika dengan status `streaming`.

Jalankan query pemeriksaan status recovery pada **Replica Node**:
```bash
docker exec -it postgres-replica psql -U app_user -d app_db -c "SELECT pg_is_in_recovery();"
```
Output harus bernilai `t` (*true*), membuktikan node berada dalam mode standby read-only.

Uji coba proteksi write pada Replica (harus ditolak):
```bash
docker exec -it postgres-replica psql -U app_user -d app_db -c "INSERT INTO predictions (user_id, model_name, features, prediction, confidence) VALUES ('test', 'iris', '{}', 'setosa', 1.0);"
```
*Output yang diharapkan:* `ERROR: cannot execute INSERT in a read-only transaction`.

### 2. Bukti Single Sign-On (SSO) Keycloak
Mendapatkan JWT Token melalui terminal (Password Grant):
```bash
curl -X POST http://localhost:8000/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "student123"}'
```

Salin nilai `access_token` yang didapat untuk digunakan pada request berikutnya.

### 3. Bukti Inferensi Model & Write ke Primary Node
Kirim data ukuran bunga Iris untuk diprediksi:
```bash
curl -X POST http://localhost:8000/api/v1/predict \
  -H "Authorization: Bearer <MASUKKAN_ACCESS_TOKEN_DI_SINI>" \
  -H "Content-Type: application/json" \
  -d '{
    "sepal_length": 5.1,
    "sepal_width": 3.5,
    "petal_length": 1.4,
    "petal_width": 0.2
  }'
```
Respons:
```json
{
  "id": 1,
  "user_id": "...",
  "username": "student",
  "model_name": "iris_classifier",
  "model_version": "v1.0.0-rf",
  "input_features": {
    "sepal_length": 5.1,
    "sepal_width": 3.5,
    "petal_length": 1.4,
    "petal_width": 0.2
  },
  "predicted_class": "setosa",
  "confidence": 0.99,
  "db_node_written": "postgres-primary (Write)"
}
```

### 4. Bukti Pembacaan Riwayat dari Standby Read-Replica
Ambil data prediksi yang tersimpan:
```bash
curl -X GET http://localhost:8000/api/v1/predictions \
  -H "Authorization: Bearer <MASUKKAN_ACCESS_TOKEN_DI_SINI>"
```
Respons:
```json
{
  "total": 1,
  "db_node_read": "postgres-replica (Read)",
  "items": [
    {
      "id": 1,
      "user_id": "...",
      "username": "student",
      "prediction": "setosa",
      "confidence": 0.99
    }
  ]
}
```

---

## 📦 Penjelasan Multi-Stage Dockerfile

Dockerfile pada direktori `api/Dockerfile` disusun dalam 2 tahap (*multi-stage*):
1. **Stage 1 (`builder`)**:
   - Menggunakan image `python:3.11-slim`.
   - Menginstal compiler (`build-essential`, `libpq-dev`).
   - Membuat *virtual environment* (`/opt/venv`) dan menginstal seluruh package dependensi.
   - Menjalankan skrip `train_model.py` untuk mengompilasi model ML (`model.joblib`) saat build.
2. **Stage 2 (`runner`)**:
   - Menggunakan base image bersih `python:3.11-slim`.
   - Hanya menginstal runtime library minimal (`libpq5` dan `curl` untuk healthcheck).
   - Menyalin folder virtual environment `/opt/venv` dan model ML dari stage builder.
   - **Keamanan (Security Best Practice)**: Membuat dan menggunakan non-root user (`appuser`, UID: 1001) sehingga kontainer tidak berjalan sebagai `root`.
   - Menentukan instruksi `HEALTHCHECK` otomatis untuk Docker daemon.

---

## 🌐 Dokumentasi Antarmuka Web (Swagger UI)

Buka browser Anda dan akses:
- **Swagger UI Interactive Docs (Docker):** [http://localhost:8000/docs](http://localhost:8000/docs)
- **ReDoc Documentation (Docker):** [http://localhost:8000/redoc](http://localhost:8000/redoc)
- **Keycloak Admin Console (Docker):** [http://localhost:8080](http://localhost:8080)

---

## ☸️ Tahap 2: Orkestrasi Cluster Kubernetes (Minikube / Kind / K3s / Docker Desktop)

Seluruh manifest untuk **Tahap 2** berada pada direktori `k8s/` dan memenuhi keempat spesifikasi utama:

| Spesifikasi Tahap 2 | Manifest Terkait | Implementasi Teknis |
| :--- | :--- | :--- |
| **1. Stateful Workloads & Dynamic PVC** | `k8s/02-postgres-primary.yaml`<br>`k8s/03-postgres-replica.yaml` | Menggunakan `StatefulSet` (`postgres-primary` & `postgres-replica`) dengan `volumeClaimTemplates` (`primary-data` & `replica-data`, `ReadWriteOnce`, `2Gi`) untuk provisi PVC dinamis yang independen dari siklus hidup pod. |
| **2. Headless Service & DNS Discovery** | `k8s/02-postgres-primary.yaml`<br>`k8s/03-postgres-replica.yaml` | Menyediakan **Headless Service** (`clusterIP: None`: `postgres-primary-headless` & `postgres-replica-headless`) untuk identitas jaringan pod StatefulSet, serta **ClusterIP Service** (`postgres-primary.kapita.svc.cluster.local` & `postgres-replica.kapita.svc.cluster.local`) sebagai single endpoint hostname tanpa merujuk ke IP fisik pod. |
| **3. Konfigurasi & Kredensial Terisolasi** | `k8s/01-config-secrets.yaml`<br>`k8s/05-api-service.yaml` | Memisahkan konfigurasi non-sensitif pada `ConfigMap` (`kapita-config`) dan seluruh password/secret sensitif pada `Secret` (`kapita-secrets`). |
| **4. SSO & Reverse Proxy Ingress** | `k8s/06-ingress.yaml`<br>`k8s/04-keycloak.yaml` | Menerapkan NGINX `Ingress` (`kapita-ingress`) untuk merutekan `http://auth.lokal` ke `keycloak-service:8080` dan `http://api.lokal` ke `api-service:8000`. |

### Langkah 1: Aktifkan Kubernetes Cluster
- **Docker Desktop (Rekomendasi di Windows):** Buka *Docker Desktop -> Settings -> Kubernetes -> Enable Kubernetes -> Apply & restart*.
- **Alternatif Minikube:** `minikube start --driver=docker && minikube addons enable ingress`

### Langkah 2: Tambahkan Domain Lokal (`auth.lokal` & `api.lokal`) ke File Hosts
Jalankan PowerShell **sebagai Administrator**, lalu tambahkan entri DNS lokal ke `C:\Windows\System32\drivers\etc\hosts`:
```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n127.0.0.1 auth.lokal api.lokal"
```

### Langkah 3: Deploy Seluruh Manifest Tahap 2
Jalankan skrip otomasi deployment:
```powershell
.\k8s\deploy.ps1
```
Atau secara manual dengan `kubectl`:
```powershell
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-config-secrets.yaml
kubectl apply -f k8s/02-postgres-primary.yaml
kubectl apply -f k8s/03-postgres-replica.yaml
kubectl apply -f k8s/04-keycloak.yaml
kubectl apply -f k8s/05-api-service.yaml
kubectl apply -f k8s/06-ingress.yaml
```

### Langkah 4: Verifikasi Otomatis Tahap 2
Jalankan skrip pengujian otomatis Kubernetes:
```powershell
python test_k8s.py
```

### Langkah 5: Bukti Independensi Fisik Data (PVC Persistence Test)
Untuk membuktikan bahwa data prediksi tetap aman di `PersistentVolumeClaim` meskipun pod database dihapus:
```powershell
# 1. Hapus pod primary database secara paksa
kubectl delete pod postgres-primary-0 -n kapita

# 2. Tunggu StatefulSet menghidupkan kembali postgres-primary-0 dan me-mount PVC yang sama
kubectl wait --for=condition=ready pod/postgres-primary-0 -n kapita --timeout=60s

# 3. Cek bahwa data tabel predictions tetap utuh!
kubectl exec -it postgres-primary-0 -n kapita -- psql -U app_user -d app_db -c "SELECT id, username, prediction, confidence, created_at FROM predictions;"
```

---

## 🔗 Tahap 3: Alur Integrasi End-to-End (Data, SSO, & Model)

Alur end-to-end menghubungkan Keycloak SSO, validasi terdistribusi JWKS, model inferensi ML, serta pemisahan Write/Read database (`CQRS`):

1. **Autentikasi Langsung ke Server SSO (Keycloak)**: Klien meminta JWT Access Token langsung dari endpoint OIDC `/realms/kapita-realm/protocol/openid-connect/token` (`http://auth.lokal`).
2. **Pengiriman Request ke Backend API**: Klien mengirim request `POST /api/v1/predict` (`http://api.lokal`) dengan header `Authorization: Bearer <access_token>`.
3. **Validasi Signature JWKS Terdistribusi**: Backend API mengambil public key RSA dari endpoint JWKS Keycloak (`/protocol/openid-connect/certs`) sesuai `kid` pada header JWT dan memverifikasi tanda tangan `RS256` secara ketat (token palsu/*tampered* otomatis ditolak dengan `401 Unauthorized`).
4. **Eksekusi Inferensi Model ML**: Model *Random Forest Classifier* (`iris_classifier`) memprediksi kelas bunga beserta skor probabilitasnya.
5. **Write Routing ke Primary Database**: Hasil inferensi disimpan otomatis ke node `postgres-primary` (`is_in_recovery = false`), disertai bukti metadata `db_routing_proof` (`operation: "WRITE"`, `server_ip`).
6. **Read Routing ke Standby Read-Replica**: Query riwayat `GET /api/v1/predictions` dialihkan otomatis oleh backend ke node `postgres-replica` (`is_in_recovery = true`), membuktikan transparansi replikasi dengan `server_ip` berbeda dari node Primary.

### Menjalankan Pengujian Otomatis Tahap 3
```powershell
python test_tahap3_e2e.py
```



