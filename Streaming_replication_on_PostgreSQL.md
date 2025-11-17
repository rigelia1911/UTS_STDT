## 3. Membuat Streaming Replication di PostgreSQL

# Konsep Dasar
Pada PostgreSQL, arsitektur **Primary–Replica** bekerja seperti berikut:

- **Primary** → menerima semua *write* (INSERT, UPDATE, DELETE)  
- **Replica** → bersifat *read-only* dan menyalin perubahan dari Primary  

Replica menerima perubahan melalui *streaming WAL (Write-Ahead Log)* sehingga datanya mengikuti Primary secara real-time.

---

## 2. Tanda-tanda Streaming Replication Aktif  
Dari log container:

### Primary membuat *replication slot*
```
pg_create_physical_replication_slot
(replication_slot,)
```
###  Primary menjalankan `00_init.sql`
```
running /docker-entrypoint-initdb.d/00_init.sql
CREATE ROLE
CREATE PHYSICAL REPLICATION SLOT
```

Artinya:
- Role untuk replikasi dibuat  
- Replication slot dibuat  

---

## 3. Cara Replica Terhubung ke Primary
Replica dijalankan dengan konfigurasi *standby mode*, misalnya melalui:

- `primary_conninfo`  
- `standby.signal`  
- `postgresql.conf` dan `pg_hba.conf`

Pada mode standby, Replica:

1. Membaca konfigurasi
2. Menghubungi Primary
3. Mengambil WAL dari replication slot
4. Menerapkan WAL nomor demi nomor
5. Menjadi salinan real-time dari Primary

Replica **read-only** karena ini aturan PostgreSQL untuk server standby.

---
## 4. Urutan Kerja Streaming Replication

### **1. Primary Start**
- Membuat role
- Membuat replication slot

### **2. Replica Start**
- Mengenali dirinya sebagai server standby
- Menghubungi Primary
- Mengambil WAL secara streaming
- Menyalin semua perubahan tanpa henti

---

## 5. Bukti Replikasi Berjalan Sukses

Streaming replication dinyatakan berhasil jika:

###  **Insert di Primary muncul di Replica**

Contoh:
```sql
INSERT INTO test_replication(message)
VALUES ('Hello replica!');
```

Pada Replica:
```sql
SELECT * FROM test_replication;
```

Muncul hasil yang sama → berarti WAL diterapkan dengan benar.

###  **Replica menolak INSERT**

Contoh:
```sql
INSERT INTO test_replication(message)
VALUES ('should fail');
```

Hasil:
```
ERROR: cannot execute INSERT in a read-only transaction
```

Ini menunjukkan Replica berjalan dalam mode standby dan replikasi aktif.

---

## 6. Mengapa Disebut “Streaming”?
Karena Replica:

- Tidak menunggu backup penuh  
- Tidak melakukan sinkronisasi per batch  
- Tapi langsung menerima WAL secara **live, real-time**

Primary → mengirim WAL → Replica menerima → replay WAL

Sehingga data dua server selalu sinkron.

---
## Kesimpulan
- Primary membuat replication slot dan siap kirim WAL  
- Replica menjalankan server standby dan mengambil WAL dari Primary  
- Replica *read-only* dan selalu mengikuti perubahan Primary  
- Data tersinkronisasi real-time  
- INSERT/UPDATE hanya boleh di Primary
