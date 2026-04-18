# MySQL Tuning untuk SIMRS - Crash Safe & Performance Optimization

Repository ini berisi dokumentasi lengkap, konfigurasi, dan panduan penerapan MySQL Tuning khusus untuk server SIMRS dengan fokus pada crash-safety dan optimasi performa.

---

## 📋 Daftar Isi

1. [Prinsip Tuning](#prinsip-tuning)
2. [Konfigurasi my.cnf](#konfigurasi-mycnf)
3. [Langkah Penerapan](#langkah-penerapan)
4. [Monitoring & Verifikasi](#monitoring--verifikasi)
5. [Backup & Recovery](#backup--recovery)
6. [Troubleshooting](#troubleshooting)

---

## 🎯 Prinsip Tuning

### 1. **InnoDB Only - Hindari MyISAM**
- Semua tabel SIMRS harus menggunakan InnoDB
- InnoDB menyediakan ACID compliance dan crash recovery
- MyISAM tidak crash-safe dan tidak recommended untuk production

### 2. **Crash-Safe Adalah Prioritas**
Konfigurasi berikut **tidak boleh diubah** untuk menjamin data safety:

```sql
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

- `innodb_flush_log_at_trx_commit = 1`: Log di-flush ke disk setiap transaction
- `sync_binlog = 1`: Binary log di-sync ke disk setiap transaction
- Kedua setting ini adalah **trade-off antara safety vs performance** tapi untuk production SIMRS, **safety harus diutamakan**

### 3. **Maksimalkan Penggunaan RAM**
- Total RAM: **125GB**
- Alokasi untuk InnoDB Buffer Pool: **64GB**
- Sisa: ~60GB untuk OS, MySQL process, dan service lain (Nextcloud, dsb)
- Buffer pool yang cukup besar mengurangi disk I/O secara signifikan

### 4. **Optimasi untuk HDD (Bukan SSD)**
- Storage menggunakan HDD, bukan SSD
- Flush strategy tidak terlalu agresif, tapi tetap crash-safe
- I/O capacity di-tune untuk HDD performance
- Prioritas: Consistency > Speed

### 5. **Disable Legacy Features** (MySQL 5.7 ke bawah)
- Matikan `query_cache` jika masih MySQL 5.7
- MySQL 5.7: `query_cache_type = 0` dan `query_cache_size = 0`
- MySQL 8.0: Query cache sudah dihapus

---

## ⚙️ Konfigurasi my.cnf

### File Lokasi
```bash
/etc/mysql/mysql.conf.d/mysqld.cnf
# atau tergantung distro:
/etc/my.cnf
/etc/mysql/my.cnf
```

### Konfigurasi Lengkap Crash-Safe

```ini
[mysqld]

##################
# DASAR SERVER
##################
user                      = mysql
pid-file                  = /var/run/mysqld/mysqld.pid
socket                    = /var/run/mysqld/mysqld.sock
datadir                   = /var/lib/mysql
bind-address              = 0.0.0.0
max_connections           = 400
thread_cache_size         = 64

##################
# INNODB - ENGINE UTAMA
##################
default_storage_engine    = InnoDB
innodb_file_per_table     = 1
innodb_flush_method       = O_DIRECT
innodb_buffer_pool_size   = 64G
innodb_buffer_pool_instances = 8
innodb_log_file_size      = 4G
innodb_log_files_in_group = 2
innodb_flush_log_at_trx_commit = 1
innodb_flush_neighbors    = 0
innodb_io_capacity        = 400
innodb_io_capacity_max    = 800
innodb_read_io_threads    = 8
innodb_write_io_threads   = 8
innodb_thread_concurrency = 0
innodb_adaptive_hash_index = 1
innodb_doublewrite        = 1
innodb_rollback_on_timeout = 1
innodb_print_all_deadlocks = 1

##################
# LOGGING & CRASH SAFETY
##################
log_error                 = /var/log/mysql/error.log
log_bin                   = /var/log/mysql/mysql-bin
binlog_format             = ROW
sync_binlog               = 1
expire_logs_days          = 7
max_binlog_size           = 512M
binlog_cache_size         = 4M

##################
# QUERY & PERFORMANCE
##################
slow_query_log            = 1
slow_query_log_file       = /var/log/mysql/slow.log
long_query_time           = 1
log_queries_not_using_indexes = 0

tmp_table_size            = 256M
max_heap_table_size       = 256M
table_open_cache          = 4000
open_files_limit          = 65535
join_buffer_size          = 4M
sort_buffer_size          = 4M
read_buffer_size          = 2M
read_rnd_buffer_size      = 4M

##################
# CHARACTER SET
##################
character-set-server      = utf8mb4
collation-server          = utf8mb4_unicode_ci

##################
# KHUSUS MYSQL 5.7 (JIKA MASIH PAKAI)
##################
query_cache_type          = 0
query_cache_size          = 0
```

---

## 🚀 Langkah Penerapan

### Step 1: Backup Konfigurasi Lama
```bash
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /root/mysqld.cnf.backup.$(date +%F)
```

### Step 2: Edit Konfigurasi
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
# atau
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Paste konfigurasi dari section di atas. Sesuaikan:
- Path log files (jika berbeda)
- Buffer pool size (jika RAM berbeda)
- Log file size (jika workload berbeda)

### Step 3: Cek Syntax Konfigurasi
```bash
sudo mysqld --verbose --help > /dev/null
```

✅ Jika tidak ada error, lanjut ke step 4.
❌ Jika ada error, cek kembali syntax di my.cnf

### Step 4: Restart MySQL
```bash
sudo systemctl restart mysql
# atau
sudo service mysql restart
```

Cek status:
```bash
sudo systemctl status mysql
```

### Step 5: Verifikasi Nilai Konfigurasi
```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'sync_binlog';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'binlog_format';
```

**Expected output:**
```
innodb_buffer_pool_size       = 68719476736 (64GB in bytes)
innodb_flush_log_at_trx_commit = 1
sync_binlog                    = 1
innodb_log_file_size          = 4294967296 (4GB in bytes)
binlog_format                 = ROW
```

---

## 📊 Monitoring & Verifikasi

### 1. Cek InnoDB Status (untuk debug crash, deadlock, I/O)
```sql
SHOW ENGINE INNODB STATUS\G
```

Perhatikan section:
- **TRANSACTIONS**: Cek jika ada deadlock
- **BUFFER POOL**: Cek usage dan efektivitas
- **FILE I/O**: Cek pending I/O operations

### 2. Cek Buffer Pool Effectiveness
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
```

**Interpretasi:**
- **Hit ratio** = (read_requests - reads) / read_requests × 100%
- Target: **99%+** (artinya data mostly dari memory, bukan disk)
- Jika < 95%, pertimbangkan naikkan buffer pool size jika RAM masih longgar

### 3. Cek I/O Performance (dari OS level)
```bash
iostat -xm 1 10
# -x: extended output
# -m: output dalam MB/s
# 1: refresh setiap 1 detik
# 10: 10 iteration
```

Perhatikan:
- **%util**: Percentage of CPU time
- **r/s, w/s**: Read/Write ops per second
- **rMB/s, wMB/s**: Read/Write throughput

### 4. Monitor Slow Queries
```bash
# Tail slow query log
tail -f /var/log/mysql/slow.log
```

### 5. Check Thread & Connection Usage
```sql
SHOW STATUS LIKE 'Threads%';
SHOW STATUS LIKE 'Connections';
SHOW PROCESSLIST;
```

---

## 💾 Backup & Recovery

### Backup Strategy
1. **Full Backup Harian**
   ```bash
   mysqldump -u root -p --all-databases --single-transaction > backup.sql
   ```

2. **Binary Log Active**
   ```bash
   # Pastikan log_bin = /var/log/mysql/mysql-bin
   # Binary log otomatis aktif sesuai konfigurasi
   ```

3. **Point-in-Time Recovery**
   - Dengan binary log aktif, bisa recovery sampai titik waktu tertentu
   - Contoh recovery dari timestamp:
   ```bash
   mysqlbinlog /var/log/mysql/mysql-bin.000001 --stop-datetime="2026-04-18 10:30:00" | mysql -u root -p
   ```

---

## 🔧 Troubleshooting

### Problem 1: MySQL Restart Gagal
**Log Error:**
```
[ERROR] InnoDB: redo log size mismatch
```

**Solusi:**
- Delete existing log files:
  ```bash
  sudo rm /var/lib/mysql/ib_logfile0
  sudo rm /var/lib/mysql/ib_logfile1
  sudo systemctl restart mysql
  ```

### Problem 2: Buffer Pool Cache Hit Ratio Rendah
**Gejala:**
- `Innodb_buffer_pool_reads` sangat tinggi
- Slow queries banyak

**Solusi:**
1. Cek apakah RAM masih tersedia
2. Naikkan `innodb_buffer_pool_size` incrementally
3. Monitor lagi setelah 24 jam

### Problem 3: Frequent Deadlocks
**Log Error:**
```
[ERROR] InnoDB: Deadlock found when trying to get lock
```

**Solusi:**
1. Enable `innodb_print_all_deadlocks = 1` (sudah di config)
2. Analisis slow log untuk query pattern
3. Optimasi query atau tambah index yang tepat
4. Review transaction isolation level

### Problem 4: Disk Space Penuh (dari log files)
**Gejala:**
- `/var/log/mysql/` penuh
- Binlog files terus bertambah

**Solusi:**
```bash
# Cek size log files
du -sh /var/log/mysql/

# Purge old binlog (keep last 7 days per config)
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);

# Manual clean (hati-hati, backup dulu)
# rm /var/log/mysql/mysql-bin.00000X
```

---

## ✅ Checklist Sebelum Production

- [ ] Backup config lama sudah dibuat
- [ ] File my.cnf sudah diupdate dengan values yang tepat
- [ ] Syntax check berhasil
- [ ] MySQL restart berhasil
- [ ] Verifikasi nilai konfigurasi sesuai expected
- [ ] Semua tabel SIMRS menggunakan InnoDB (check dengan: `SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE FROM information_schema.TABLES WHERE TABLE_SCHEMA='simrs';`)
- [ ] Binary log aktif
- [ ] Slow query log aktif
- [ ] Buffer pool hit ratio > 95% (setelah warmup 24 jam)
- [ ] Tidak ada pending deadlock di INNODB STATUS
- [ ] Backup procedure sudah disetup
- [ ] Monitoring tools sudah configured

---

## 📞 Support & Documentation

- **Official MySQL Docs**: https://dev.mysql.com/doc/
- **Percona Documentation**: https://www.percona.com/software/mysql-tools/percona-server
- **MySQL Error Reference**: https://dev.mysql.com/doc/mysql-errors/

---

## 📝 Versi & Changelog

- **Last Updated**: 2026-04-18
- **MySQL Version Target**: 5.7, 8.0
- **Environment**: Production SIMRS with 125GB RAM, HDD Storage

---

**Created by**: diona-katzlein  
**Repository**: mysql-tuning-simrs
