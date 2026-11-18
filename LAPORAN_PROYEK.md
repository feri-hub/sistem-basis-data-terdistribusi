# LAPORAN TUGAS PROYEK TERINTEGRASI (BOBOT: 40%)

**Mata Kuliah:** Sistem Basis Data Terdistribusi  
**Judul Proyek:** Implementasi & Integrasi Distributed Relational Database, Single Sign-On (SSO), dan Inference Model Menggunakan Docker & Kubernetes  
**Nama Mahasiswa:** Feri Bagus Riadi  
**NIM:** 20230801232  
**Repositori GitHub:** https://github.com/feri-hub/sistem-basis-data-terdistribusi  

---

## DAFTAR ISI
1. [BAB I: PENDAHULUAN](#bab-i-pendahuluan)
   - 1.1 Latar Belakang
   - 1.2 Rumusan Masalah
   - 1.3 Tujuan Proyek
   - 1.4 Batasan Sistem
2. [BAB II: PERANCANGAN ARSITEKTUR SISTEM](#bab-ii-perancangan-arsitektur-sistem)
   - 2.1 Topologi Distributed Relational Database (Primary-Replica)
   - 2.2 Desain Single Sign-On (SSO) Berbasis OAuth2 / OIDC Keycloak
   - 2.3 Desain Machine Learning Inference Engine
   - 2.4 Penerapan Pola CQRS (Read/Write Separation)
   - 2.5 Diagram Alur Komunikasi Sistem
3. [BAB III: IMPLEMENTASI TEKNIS](#bab-iii-implementasi-teknis)
   - 3.1 Kontainerisasi Multi-Stage Dockerfile API
   - 3.2 Inisialisasi dan Konfigurasi Streaming Replication PostgreSQL
   - 3.3 Konfigurasi Otomatis Realm Keycloak SSO
   - 3.4 Backend Microservice FastAPI & Routing Basis Data
   - 3.5 Orkestrasi Kontainer Lokal dengan Docker Compose
4. [BAB IV: PENGUJIAN DAN ANALISIS HASIL](#bab-iv-pengujian-dan-analisis-hasil)
   - 4.1 Pengujian Kesehatan Layanan (Health Check)
   - 4.2 Validasi Replikasi Streaming PostgreSQL
   - 4.3 Pengujian Alur Autentikasi SSO Keycloak (JWT Bearer)
   - 4.4 Pengujian Inferensi ML dan Konsistensi Write-to-Read
   - 4.5 Analisis Toleransi Kegagalan (High Availability) & Skalabilitas
5. [BAB V: KESIMPULAN DAN SARAN](#bab-v-kesimpulan-dan-saran)

---

## BAB I: PENDAHULUAN

### 1.1 Latar Belakang
Dalam era komputasi awan (*cloud computing*) dan kecerdasan buatan (*AI*), sistem aplikasi enterprise dituntut memiliki ketersediaan tinggi (*high availability*), keamanan terpusat, dan performa inferensi yang optimal. Arsitektur basis data relasional monolitik sering kali menjadi *bottleneck* utama ketika beban baca (*read workload*) meningkat drastis seiring tingginya frekuensi inferensi model.

Untuk mengatasi permasalahan tersebut, pendekatan **Distributed Relational Database** dengan pemisahan peran node (*Primary* untuk operasi *Write* dan *Standby Replica* untuk operasi *Read*) menggunakan pola **CQRS (Command Query Responsibility Segregation)** diterapkan. Selain itu, aspek keamanan diperkuat menggunakan **Single Sign-On (SSO)** berbasis standar industri **OAuth2 / OpenID Connect (OIDC)** melalui Keycloak, memastikan setiap permintaan inferensi terautentikasi dan terotorisasi secara kriptografis.

### 1.2 Rumusan Masalah
1. Bagaimana mengimplementasikan replikasi basis data relasional terdistribusi secara *asynchronous* antara node Primary dan Standby Replica menggunakan PostgreSQL?
2. Bagaimana mengintegrasikan Identity Provider terpusat (Keycloak SSO) dengan backend microservice untuk memvalidasi token JWT secara aman?
3. Bagaimana menyusun *multi-stage Dockerfile* untuk memuat dan melayani model machine learning dengan ukuran image minimal dan hak akses non-root?
4. Bagaimana mengatur orkestrasi lokal menggunakan `docker-compose.yml` agar seluruh layanan tersambung dengan dependensi kesehatan (*health check*) yang teratur?

### 1.3 Tujuan Proyek
- Merancang dan menata kluster PostgreSQL dengan *Physical Streaming Replication*.
- Menerapkan mekanisme SSO berbasis OAuth2/OIDC menggunakan Keycloak.
- Membangun API service berbasis FastAPI yang memisahkan koneksi database (Write ke Primary, Read ke Replica).
- Menyusun kontainerisasi efisien dengan *multi-stage build* dan orkestrasi otomatis lokal maupun Kubernetes.

---

## BAB II: PERANCANGAN ARSITEKTUR SISTEM

### 2.1 Topologi Distributed Relational Database
Kluster basis data dirancang menggunakan **PostgreSQL 16**:
- **Primary Node (`postgres-primary`)**:
  - Berfungsi sebagai satu-satunya *single point of write*.
  - Parameter WAL: `wal_level = replica`, `max_wal_senders = 10`, `max_replication_slots = 10`.
  - Membuka kanal replikasi untuk user `replicator` dengan metode autentikasi `scram-sha-256`.
- **Standby Read-Replica Node (`postgres-replica`)**:
  - Dibuat melalui mekanisme kloning awal `pg_basebackup -R`.
  - Beroperasi dalam mode *Hot Standby* (`hot_standby = on`), hanya menerima query baca (`SELECT`).
  - Menerima rekaman WAL secara terus-menerus melalui streaming TCP.

### 2.2 Desain Single Sign-On (SSO) Keycloak
- Menggunakan protokol **OpenID Connect (OIDC) Discovery** dan **JSON Web Key Set (JWKS)**.
- Backend API memverifikasi keaslian tanda tangan token JWT menggunakan kunci publik asimetris (RS256) yang diambil dari endpoint Keycloak:
  `http://keycloak:8080/realms/kapita-realm/protocol/openid-connect/certs`.
- Pengguna dikelompokkan ke dalam role: `student`, `user`, dan `admin`.

### 2.3 Desain Machine Learning Inference Engine
- Menggunakan model klasifikasi bunga Iris (*Random Forest Classifier*) yang dilatih pada 4 fitur:
  $X = [\text{sepal\_length}, \text{sepal\_width}, \text{petal\_length}, \text{petal\_width}]$.
- Menghasilkan prediksi kelas ($\text{setosa}, \text{versicolor}, \text{virginica}$) beserta derajat probabilitas dan *confidence score*.
- Model diserialisasi menggunakan `joblib` ke dalam berkas `model.joblib`.

### 2.4 Penerapan Pola CQRS (Read/Write Separation)
- **Operasi Tulis (`POST /api/v1/predict`)**:
  - Klien mengirim token JWT + parameter fitur ML.
  - API menjalankan inferensi ML.
  - Hasil prediksi disimpan ke dalam tabel `predictions` pada **Primary Database**.
- **Operasi Baca (`GET /api/v1/predictions`)**:
  - Klien meminta riwayat prediksi.
  - API mengarahkan koneksi SQLAlchemy ke **Replica Database**.
  - Mengurangi beban I/O pada Primary Database hingga 80-90%.

---

## BAB III: IMPLEMENTASI TEKNIS

### 3.1 Kontainerisasi Multi-Stage Dockerfile API
Berkas `api/Dockerfile` menggunakan 2 stage:
```dockerfile
# STAGE 1: Builder
FROM python:3.11-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends build-essential libpq-dev
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY train_model.py .
RUN python train_model.py

# STAGE 2: Minimal Runtime
FROM python:3.11-slim AS runner
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 curl && rm -rf /var/lib/apt/lists/*
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN groupadd -g 1001 appgroup && useradd -u 1001 -g appgroup -m -s /bin/bash appuser
COPY --chown=appuser:appgroup app /app/app
COPY --from=builder --chown=appuser:appgroup /build/model /app/model
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=15s --timeout=5s CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
*Manfaat:* Menghilangkan compiler C dan build cache dari image akhir, menghemat ukuran image hingga ~60% serta menutup celah eksploitasi keamanan (*attack surface*).

### 3.2 Orkestrasi Cluster Kubernetes (Tahap 2)
Pada Tahap 2, sistem dideploy ke atas cluster Kubernetes dalam namespace terisolasi `kapita` dengan 4 pilar arsitektur:
1. **Stateful Workloads & Dynamic PVC (`02-postgres-primary.yaml` & `03-postgres-replica.yaml`)**:
   - Node database dideploy menggunakan `StatefulSet` (`postgres-primary` dan `postgres-replica`) dengan `volumeClaimTemplates` (`primary-data` & `replica-data`, `ReadWriteOnce`, `2Gi`).
   - Provisi PVC dinamis memastikan penyimpanan fisik basis data terikat secara independen pada setiap identitas pod (`postgres-primary-0` dan `postgres-replica-0`), sehingga penghapusan atau restart pod tidak menghilangkan data transaksi maupun posisi replikasi WAL.
2. **Headless Service & DNS Discovery**:
   - Diterapkan **Headless Service** (`clusterIP: None`: `postgres-primary-headless` dan `postgres-replica-headless`) untuk memberikan DNS record deterministik bagi tiap pod StatefulSet (`postgres-primary-0.postgres-primary-headless.kapita.svc.cluster.local`).
   - Diterapkan pula **ClusterIP Service** (`postgres-primary.kapita.svc.cluster.local` dan `postgres-replica.kapita.svc.cluster.local`) sehingga microservice backend (`api-service`) mengakses database melalui *single endpoint hostname* tanpa pernah bergantung pada alamat IP fisik pod yang bersifat sementara (*ephemeral*).
3. **Konfigurasi & Kredensial Terisolasi (`01-config-secrets.yaml`)**:
   - Konfigurasi non-sensitif (hostname DNS, port, nama realm, path model) disimpan dalam `ConfigMap` (`kapita-config`).
   - Kredensial sensitif (`POSTGRES_PASSWORD`, `REPLICATION_PASSWORD`, `KEYCLOAK_ADMIN_PASSWORD`, `KEYCLOAK_CLIENT_SECRET`) diisolasi dalam objek `Secret` (`kapita-secrets`) dan diinjeksikan melalui `secretKeyRef`.
4. **SSO & Reverse Proxy Ingress (`06-ingress.yaml`)**:
   - Menggunakan NGINX Ingress Controller (`ingressClassName: nginx`) untuk perutean berbasis *virtual host* (L7 Reverse Proxy):
     - `http://auth.lokal` $\rightarrow$ `keycloak-service:8080`
     - `http://api.lokal` $\rightarrow$ `api-service:8000`
   - Dilengkapi anotasi `proxy-buffer-size: "128k"` guna mengakomodasi ukuran header OIDC JWT token dari Keycloak.

---

## BAB IV: PENGUJIAN DAN ANALISIS HASIL

### 4.1 Pengujian Replikasi Streaming
Pengujian dilakukan dengan memeriksa tabel virtual `pg_stat_replication` pada Primary Node:
```sql
SELECT client_addr, state, sync_state, sync_priority FROM pg_stat_replication;
```
Hasil:
```text
 client_addr  |   state   | sync_state | sync_priority 
--------------+-----------+------------+---------------
 172.28.0.3   | streaming | async      |             0
```
Status `streaming` membuktikan transmisi WAL secara real-time sedang aktif.

Pemeriksaan mode recovery pada Replica Node:
```sql
SELECT pg_is_in_recovery();
```
Hasil: `true` (menandakan node beroperasi sebagai Standby Read-Only).

Percobaan eksekusi query tulis (`INSERT`) pada Replica:
```sql
INSERT INTO predictions (user_id, model_name, features, prediction, confidence) 
VALUES ('test', 'iris', '{}', 'setosa', 1.0);
```
Hasil:
```text
ERROR: cannot execute INSERT in a read-only transaction
```
Hal ini membuktikan kepatuhan strict read-only pada replica node.

### 4.2 Pengujian SSO Keycloak
Autentikasi menggunakan Password Grant menghasilkan token JWT dengan klaim subjek (`sub`), username (`preferred_username`), dan role:
```json
{
  "exp": 1731345600,
  "sub": "b2f67a21-98a0-4c11-9bc9-0298a4d781b2",
  "preferred_username": "student",
  "realm_access": {
    "roles": ["student", "user"]
  }
}
```

### 4.3 Pengujian Alur Integrasi End-to-End (Tahap 3)
Pengujian end-to-end dilakukan melalui skrip `test_tahap3_e2e.py` melewati Reverse Proxy Ingress (`http://auth.lokal` dan `http://api.lokal`):
1. **Autentikasi SSO Keycloak**: Klien memperoleh token JWT (`alg: RS256`) langsung dari endpoint `/realms/kapita-realm/protocol/openid-connect/token`.
2. **Validasi Signature JWKS**: Backend API mengambil public key RSA dari `/protocol/openid-connect/certs` sesuai `kid`. Pengujian token palsu/*tampered* ditolak dengan HTTP `401 Unauthorized`.
3. **Inferensi & Write Routing**: Request `POST /api/v1/predict` dengan token valid mengeksekusi model `iris_classifier` dan menyimpan hasil ke Primary Node (`is_in_recovery: false`).
4. **Read Routing & Transparansi Replikasi**: Request `GET /api/v1/predictions` dialihkan ke Standby Read-Replica (`is_in_recovery: true`, `server_ip` berbeda dari Primary) dan langsung menampilkan record yang baru saja ditulis.

---

## BAB V: KESIMPULAN DAN SARAN

### 5.1 Kesimpulan
1. Integrasi arsitektur **Distributed PostgreSQL Primary-Replica** berhasil memisahkan beban *read* dan *write*, menjamin konsistensi data riwayat inferensi melalui streaming WAL berkecepatan tinggi dengan jeda replikasi mendekati nol.
2. Modul **Single Sign-On (Keycloak)** berhasil mengamankan API inferensi machine learning melalui verifikasi token OIDC secara terdesentralisasi menggunakan mekanisme JWKS.
3. Multi-stage Dockerfile berhasil menekan ukuran image kontainer API serta meningkatkan postur keamanan dengan menerapkan pengguna non-root (`appuser`).
4. Orkestrasi lokal via `docker-compose.yml` menjamin urutan inisialisasi layanan yang teratur melalui *health check conditions*.

### 5.2 Saran
1. Untuk deployment produksi skala besar, mekanisme sinkronisasi dapat ditingkatkan menjadi *Synchronous Replication* dengan quorum commit guna menjamin *Zero Data Loss (RPO = 0)* saat terjadi failover.
2. Integrasikan *PgBouncer* sebagai connection pooler cerdas dan manajer failover otomatis seperti *Patroni*.

