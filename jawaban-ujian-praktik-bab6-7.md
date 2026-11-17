# Jawaban Ujian Praktik DevSecOps Bab 6–7

Disusun mengikuti urutan soal pada PDF "Soal Interview Ujian Praktik DevSecOps Bab 6–7" (25 soal, 5 paket). Setiap soal dipisah menjadi:

- **Demonstrasi**: perintah yang dijalankan, hal yang perlu ditunjukkan, dan penjelasannya.
- **Pertanyaan lanjutan**: jawaban atas pertanyaan penguji.
- **⚠ Penyesuaian modul** (hanya jika ada): langkah tambahan bila lab/konfigurasi modul dosen (`bab-06.md`, `bab-07.md`) berbeda dengan cheatsheet.

> **Aturan ujian (dari PDF):** boleh membuka cheatsheet, tetapi penjelasan harus dengan kata-kata sendiri. Evidence harus dari lab Anda saat sesi; output contoh tidak dihitung. Bagian bertanda **[isi dari lab Anda]** wajib diganti dengan output nyata saat ujian.

| Paket | Bab 6 | Bab 7  | Skenario PMB |
| ----- | ----- | ------ | ------------ |
| A     | 1, 5  | 13, 17 | 21           |
| B     | 2, 6  | 11, 16 | 22           |
| C     | 3, 8  | 14, 18 | 23           |
| D     | 4, 9  | 12, 15 | 24           |
| E     | 7, 10 | 19, 20 | 25           |

---

# BAB 6 — CENTRALIZED LOGGING

## Soal 1 — Alur log end-to-end

### Demonstrasi

```bash
docker compose ps --all
```

Yang ditunjukkan: 3 generator berstatus `Exited (0)` (selesai menulis 20 baris), sedangkan `postgres-db` dan `fluent-bit` tetap `Up`.

Perjalanan satu baris log:

1. Generator menulis JSON ke **stdout**.
2. **Docker Engine** (bukan container generator) mengirimnya lewat _fluentd logging driver_ ke `127.0.0.1:24224` dengan tag `docker.<nama-container>`.
3. **Fluent Bit** menerimanya di input `forward` dan menahannya di buffer filesystem (`flb-storage`).
4. Output **pgsql** menulis ke `logs.fluentbit` (kolom `tag`, `time`, `data` JSONB).

### Pertanyaan lanjutan — `fluentd-address` diganti `fluent-bit:24224`

Log tidak sampai. Yang menjalankan driver adalah **Docker daemon di host**, bukan container, sehingga nama service Compose `fluent-bit` tidak bisa di-resolve. Dengan `fluentd-async: "true"` container tetap start tetapi tabel tidak bertambah; tanpa async, container gagal dibuat.

> ⚠ **Penyesuaian modul:** modul memakai `localhost:24224` dan `tag: "docker.generator"` (statis). Terapkan Bab 0.1 Langkah 3–4 (port `127.0.0.1:24224` dipublish + tag `docker.{{.Name}}`).

---

## Soal 2 — Verifikasi record dan JSONB

### Demonstrasi

```bash
./scripts/psql-admin.sh -c "SELECT tag, count(*) FROM logs.fluentbit GROUP BY tag ORDER BY 2 DESC;"
./scripts/psql-admin.sh -c "SELECT pg_typeof(data), count(*) FROM logs.fluentbit GROUP BY pg_typeof(data);"
```

Hasil `pg_typeof` adalah `jsonb`. Lokasi data:

- `container_name` dan `source` adalah metadata Docker, disimpan sebagai **key di dalam objek `data`** (`data->>'container_name'`, `data->>'source'`).
- Isi log aplikasi ada di `data->>'log'` sebagai **string**.

### Pertanyaan lanjutan — membuat `severity` bisa di-query sendiri

- Tambahkan **filter parser** Fluent Bit: `Key_Name log`, `Parser json`, `Reserve_Data On`, sehingga muncul `data->>'severity'`.
- Tanpa parser, cast manual: `(data->>'log')::jsonb->>'severity'`.

---

## Soal 3 — Konfigurasi output PostgreSQL

### Demonstrasi

Tampilkan `fluent-bit/fluent-bit.conf` dan:

```bash
docker compose exec -T postgres-db psql -U fluentbit_user -d labdb -c 'SHOW search_path;'
```

Hasil: `logs, public`. Konfigurasi hanya berisi `Table fluentbit` tanpa `Schema`. Schema `logs` dipilih dari `ALTER ROLE fluentbit_user IN DATABASE ... SET search_path TO logs, public` di `init/01-init-logging.sh`. Jadi Fluent Bit tidak "tahu" schema; PostgreSQL yang menyelesaikan nama tabel lewat `search_path` role tersebut.

### Pertanyaan lanjutan — galat `relation fluentbit does not exist`

Yang diperiksa:

1. `SHOW search_path` untuk role tersebut.
2. `\dt logs.*` untuk memastikan tabel ada.
3. Apakah init script benar-benar jalan (init hanya dijalankan pada **volume kosong**).
4. Nama `Table`, `Database`, dan `User` di conf.
5. `\dn+ logs` untuk hak `USAGE`.

> ⚠ **Penyesuaian modul:** modul memakai `User labuser` dan `Schema logs` di conf. Terapkan Bab 0.1 Langkah 1–2 agar sesuai cheatsheet (user `fluentbit_user`, `search_path` via role). Jika Anda tetap memakai konfigurasi modul, jelaskan jawabannya sesuai konfigurasi yang benar-benar berjalan di lab Anda.

---

## Soal 4 — Least privilege akun ingest

### Demonstrasi

```sql
\dp logs.fluentbit
\dn+ logs
```

`fluentbit_user` hanya punya:

- `USAGE`, `CREATE` pada schema `logs` (plugin perlu bootstrap tabel).
- `SELECT`, `INSERT` pada `logs.fluentbit`.
- `SELECT` pada view `recent_logs`.

Tidak ada `UPDATE`/`DELETE`, sehingga akun ingest tidak bisa mengubah atau menghapus bukti.

### Pertanyaan lanjutan — bocor `fluentbit_user` vs admin

- **`fluentbit_user` bocor:** penyerang hanya bisa menyisipkan log palsu, membaca log, dan membuat objek di schema `logs`.
- **Admin bocor:** kendali penuh — ubah/hapus semua data, buat role.
- Setelah schema stabil, hak `CREATE` sebaiknya dicabut.

> ⚠ **Penyesuaian modul:** modul tidak membuat akun terpisah. Terapkan Bab 0.1 Langkah 1 sebelum sesi, lalu `\dp` dan `\dn+` akan menampilkan hak di atas.

---

## Soal 5 — Buffering saat backend mati

### Demonstrasi

```bash
./scripts/test-buffering.sh
docker compose logs --tail 200 fluent-bit
```

Yang dilihat: saat PostgreSQL berhenti, log Fluent Bit menampilkan kegagalan flush dan _retry_ ke `postgres-db` **[isi: salin pesan dari lab Anda]**. Setelah PostgreSQL hidup, jumlah baris naik dan skrip mencetak `[PASS]`.

Buffer yang bekerja adalah **buffer filesystem Fluent Bit** (`storage.type filesystem`, volume `flb-storage`, batas 256 MB), **bukan** buffer driver Docker yang kecil.

### Pertanyaan lanjutan — mengapa PASS bukan "tidak ada pesan galat"

Galat justru wajar saat backend mati. PASS berarti: kegagalan output **terlihat**, collector **tetap jalan**, dan record **bertambah setelah backend pulih**.

> ⚠ **Penyesuaian modul:** modul tidak punya buffer filesystem dan `test-buffering.sh`. Terapkan Bab 0.1 Langkah 2–3, lalu jika skrip tidak ada, lakukan manual:
>
> ```bash
> # 1. hitung awal
> docker compose exec -T postgres-db psql -U labadmin -d labdb -t -c "SELECT count(*) FROM logs.fluentbit;"
> # 2. matikan backend
> docker compose stop postgres-db
> # 3. hasilkan log baru (jalankan ulang generator)
> docker compose up generator-1 generator-2 generator-3    # sesuaikan dengan lab Anda
> # 4. amati kegagalan & retry
> docker compose logs --tail 200 fluent-bit
> # 5. hidupkan lagi, tunggu beberapa flush, hitung ulang
> docker compose start postgres-db
> sleep 20
> docker compose exec -T postgres-db psql -U labadmin -d labdb -t -c "SELECT count(*) FROM logs.fluentbit;"
> ```
>
> PASS jika jumlah akhir > jumlah awal dan Fluent Bit tetap `Up`.

---

## Soal 6 — Monitoring collector

### Demonstrasi

```bash
curl -fsS http://127.0.0.1:2020/api/v1/health
curl -fsS http://127.0.0.1:2020/api/v1/metrics
```

`health` hanya membuktikan proses hidup. Angka penting di `metrics`:

- **input:** `records`, `bytes` (yang diterima).
- **output pgsql:** `proc_records` (berhasil diproses), `errors`, `retries`, `retries_failed`, `dropped_records`.

### Pertanyaan lanjutan — health OK tetapi tabel tidak bertambah

Urutan pemeriksaan: input `records` (apakah log masuk) → output `proc_records` → `errors` → `retries` → `dropped_records` → terakhir `docker compose logs fluent-bit`.

> ⚠ **Penyesuaian modul:** modul tidak mengaktifkan HTTP server, jadi curl akan gagal (_connection refused_). Terapkan Bab 0.1 Langkah 2 (`HTTP_Server On`, port 2020, `Health_Check On`) dan Langkah 3 (publish `127.0.0.1:2020`).

---

## Soal 7 — Rekonsiliasi kelengkapan event

### Demonstrasi

```bash
./scripts/psql-admin.sh -c "SELECT tag, count(*) FROM logs.fluentbit WHERE tag LIKE '%generator-%' GROUP BY tag;"
```

- **Dihasilkan** = 3 generator × 20 = 60.
- **Diterima** = `input records` di `/api/v1/metrics`.
- **Diproses** = `proc_records` output di `/api/v1/metrics`.
- **Tersimpan** = `count(*)` di PostgreSQL.

Menafsirkan selisih:

| Kondisi                                        | Arti                                                                       |
| ---------------------------------------------- | -------------------------------------------------------------------------- |
| Input < dihasilkan                             | Log hilang di driver Docker (buffer async penuh atau collector belum siap) |
| Output < input                                 | Belum flush, ada retry, atau `dropped_records`                             |
| Baris DB > output                              | Duplikat akibat retry                                                      |
| Generator dijalankan dua kali tanpa `TRUNCATE` | Jumlah jadi 40 per tag, bukan 20                                           |

### Pertanyaan lanjutan — mendeteksi dan mencegah record ganda

Deteksi:

```sql
SELECT tag, data->>'log', count(*) FROM logs.fluentbit
GROUP BY 1,2 HAVING count(*) > 1;
```

Pencegahan: tambahkan **event ID unik**, lalu deduplikasi saat query atau di tahap berikutnya. Pipeline ini **at-least-once**, bukan exactly-once.

> ⚠ **Penyesuaian modul:** tag statis `docker.generator` di modul membuat semua generator bertag sama, sehingga hitungan per generator tidak bisa dipisah. Terapkan Bab 0.1 Langkah 4–5 (tag `docker.{{.Name}}`). Catat juga angka `input records` / `proc_records` **sebelum dan sesudah** run agar selisih per run bisa dihitung dengan benar.

---

## Soal 8 — Redaction dengan nilai canary

### Demonstrasi

Tambahkan token sintetis, mis. `"api_token":"tok_canary_9f3c2a71d4b8"`, ke satu generator. Pasang filter Lua `redact.lua` (`Match docker.*`, `call redact`) **sebelum** output. Buktikan:

```sql
SELECT count(*) FROM logs.fluentbit WHERE data::text ILIKE '%tok_canary%';   -- harus 0
SELECT count(*) FROM logs.fluentbit WHERE data::text LIKE '%[REDACTED]%';    -- > 0 (kontrol positif)
```

Kontrol positif penting agar hasil 0 tidak sekadar berarti "log tidak masuk".

### Pertanyaan lanjutan — ke mana lagi canary bisa bocor

- stdout container asli di sisi producer.
- Output stdout Fluent Bit (`docker compose logs fluent-bit`).
- Buffer filesystem `flb-storage`.
- Backup atau ekspor data.
- Screenshot atau laporan.

Karena itu redaction idealnya dilakukan di **producer**, lalu diverifikasi lagi di collector.

> ⚠ **Penyesuaian modul:** modul tidak punya `redact.lua`. Langkah:
>
> 1. Buat `fluent-bit/redact.lua`:
>    ```lua
>    function redact(tag, timestamp, record)
>      local log = record["log"]
>      if log ~= nil then
>        log = string.gsub(log, "tok_canary_[%w]+", "[REDACTED]")
>        record["log"] = log
>        return 1, timestamp, record
>      end
>      return 0, timestamp, record
>    end
>    ```
> 2. Mount ke container Fluent Bit (`./fluent-bit/redact.lua:/fluent-bit/etc/redact.lua:ro`).
> 3. Tambahkan di conf, **di atas** blok `[OUTPUT]`:
>    ```ini
>    [FILTER]
>      Name   lua
>      Match  docker.*
>      script /fluent-bit/etc/redact.lua
>      call   redact
>    ```
> 4. Karena modul memiliki `[OUTPUT] stdout`, ia menerima record setelah filter sehingga stdout Fluent Bit tidak berisi canary, tetapi **stdout container generator tetap berisi canary**. Tunjukkan itu (`docker logs generator-x`) sebagai bukti jalur bocor nyata.

---

## Soal 9 — Kebijakan retensi

### Demonstrasi

Tunjukkan query `DELETE` yang dikomentari di `analysis.sql`:

```sql
-- DELETE FROM logs.fluentbit WHERE time < CURRENT_TIMESTAMP - INTERVAL '7 days';
```

Rancangan retensi tujuh hari:

- Hapus **per batch** (prosedur `purge_old_logs`, 5000 baris per batch, `COMMIT` tiap batch) dengan `lock_timeout`.
- Jadwal malam hari lewat cron.
- Index `time` sudah ada (`idx_fb_time`).
- Hitung dulu kandidat dengan `SELECT count(*)`.
- Produksi: **partisi per hari** lalu `DROP PARTITION`.

### Pertanyaan lanjutan — risiko `TIMESTAMP WITHOUT TIME ZONE`

- Tidak ada informasi zona. `CURRENT_TIMESTAMP` dibandingkan menurut **zona sesi**, sehingga batas tujuh hari bisa meleset beberapa jam.
- Korelasi dengan sistem lain (Prometheus/Grafana memakai UTC) bisa bergeser.
- Mitigasi: pastikan DB dan sesi UTC, atau pakai `timestamptz`.

> ⚠ **Penyesuaian modul:** modul tidak menyertakan `analysis.sql`/`purge_old_logs`. Buat `queries/analysis.sql` yang berisi query distribusi tag dan baris `DELETE` di atas (dikomentari), serta hitung kandidat:
> `SELECT count(*) FROM logs.fluentbit WHERE time < CURRENT_TIMESTAMP - INTERVAL '7 days';`
> Modul sendiri memakai `TIMESTAMP` tanpa zona (sama dengan soal), jadi jawaban risiko tetap berlaku.

---

## Soal 10 — Ekspor log sebagai evidence

### Demonstrasi

```bash
docker compose exec -T postgres-db psql -U labadmin -d labdb \
  -c "\copy (SELECT time, tag, data FROM logs.fluentbit WHERE time >= '2026-11-17 09:00' AND time < '2026-11-17 10:00' ORDER BY time) TO STDOUT CSV HEADER" > reports/export.csv
sha256sum reports/export.csv | tee reports/export.csv.sha256
```

Ganti rentang waktu dengan waktu lab Anda. Ekspor harus disertai:

- rentang waktu,
- query yang dipakai,
- zona waktu,
- jumlah baris,
- checksum,
- siapa dan kapan mengambilnya,

agar analisis bisa diulang.

### Pertanyaan lanjutan — kapan PostgreSQL tidak lagi cocok

Ketika **volume ingest tinggi dan terus-menerus**, **retensi panjang**, atau **pencarian teks besar** mendominasi. Pertimbangkan Loki, OpenSearch, atau ClickHouse.

> ⚠ Pastikan folder `reports/` sudah ada (`mkdir -p reports`) sebelum redirect `>`.

---

# BAB 7 — MONITORING

## Soal 11 — Arsitektur dan exposure stack

### Demonstrasi

```bash
docker compose ps
ss -lntp
```

Hanya dua port terbuka ke host, keduanya di loopback: Prometheus `127.0.0.1:9090` dan Grafana `127.0.0.1:3000`. Node Exporter (`:9100`) dan cAdvisor (`:8080`) **tidak dipublikasikan**. Prometheus menjangkaunya lewat network internal `bab7-monitoring-net` dengan nama service (`node-exporter:9100`, `cadvisor:8080` di `prometheus.yml`).

### Pertanyaan lanjutan — mengapa cAdvisor `privileged: true`

Ia perlu membaca metadata host dan runtime container (mount `/`, `/sys`, `/var/lib/docker`). Tidak boleh dijalankan di host produksi atau laptop yang memegang data sensitif tanpa isolasi dan persetujuan, karena visibilitasnya ke host sangat luas.

> ⚠ **Penyesuaian modul:** modul mempublish `9090:9090`, `3000:3000`, dan cAdvisor `8081:8080` ke semua interface, serta tidak memakai `privileged`. Terapkan Bab 0.2 Langkah 1. Jika `ss -lntp` masih menunjukkan `0.0.0.0:8081` atau `*:3000`, berarti ada yang belum diubah.

---

## Soal 12 — Validasi rule sebelum dijalankan

### Demonstrasi

```bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus test rules /etc/prometheus/tests/rules.test.yml
```

- `check config` membuktikan sintaks valid dan rule file termuat. **Tidak** membuktikan target hidup.
- `test rules` membuktikan logika `TargetDown` secara deterministik dengan data sintetis. **Tidak** membuktikan jaringan, target nyata, atau notifikasi.

### Pertanyaan lanjutan — `eval_time` diubah `1m`

Test **gagal**. Seri `up` baru bernilai 0 pada menit ke-1, sedangkan `for: 1m` membuat alert baru _firing_ pada menit ke-2. Pada menit ke-1 statusnya baru _pending_, sehingga `exp_alerts` tidak cocok.

> ⚠ **Penyesuaian modul:** modul tidak punya `rules.test.yml`. Terapkan Bab 0.2 Langkah 2–3 (file `alert-rules.yml`, `tests/rules.test.yml`, dan mount-nya). Pastikan `job` di test sama dengan label di rule/`prometheus.yml`.

---

## Soal 13 — Status target dan makna `up`

### Demonstrasi

```bash
curl -fsS http://127.0.0.1:9090/api/v1/targets
./scripts/verify-stack.sh
```

`up = 1` berarti scrape terakhir ke target **berhasil**. Itu tidak membuktikan fungsi bisnis sehat.

### Pertanyaan lanjutan — semua target UP tetapi pengguna terdampak

Contoh: container `Up` tetapi CPU jenuh sehingga layanan lambat; terlihat di `container:cpu_usage_cores:rate1m`. Untuk error aplikasi dibutuhkan metric tambahan seperti **error rate** atau **latency** (SLI).

> ⚠ **Penyesuaian modul:** job node exporter di modul bernama `node`, sedangkan cheatsheet memakai `node-exporter`. Samakan (Bab 0.2 Langkah 2). Jika `verify-stack.sh` tidak ada, pakai pengganti di Bab 0.2 Langkah 7.

---

## Soal 14 — Counter dan `rate()`

### Demonstrasi

Di Prometheus Graph jalankan `node_cpu_seconds_total` mentah, lalu `rate(node_cpu_seconds_total[5m])`.

- `node_cpu_seconds_total` adalah **counter kumulatif** yang naik terus, sehingga grafiknya menanjak dan tidak menunjukkan beban.
- `rate(...[5m])` menunjukkan **laju rata-rata per detik** dalam 5 menit terakhir, yaitu pemakaian CPU sebenarnya.

### Pertanyaan lanjutan — `rate(...[5m])` kosong sesaat setelah stack hidup

Jendela 5 menit belum berisi sedikitnya **dua sampel** (scrape tiap 15 detik). Tunggu beberapa scrape. **Kosong bukan berarti nol.**

---

## Soal 15 — Recording rule CPU host

### Demonstrasi

Definisi di `recording-rules.yml`:

```promql
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

Manual:

```promql
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

Hasilnya sama; rule berupa **rasio 0–1**, sedangkan manual dalam **persen** (kalikan rule dengan 100).

### Pertanyaan lanjutan — hasil berbeda sedikit

- Waktu evaluasi berbeda (rule dievaluasi tiap 15 detik).
- Jendela `rate` bergeser.
- `avg by (instance)` berbeda dari `avg` tanpa grouping bila ada beberapa instance.

> ⚠ **Penyesuaian modul:** modul hanya memuat query manual `100 - avg(...) * 100`, tanpa recording rule. Terapkan Bab 0.2 Langkah 3 (`recording-rules.yml`), lalu cek di Prometheus → Status → Rules bahwa rule sehat sebelum membandingkan.

---

## Soal 16 — Load test dan denominator CPU

### Demonstrasi

```bash
docker compose --profile load up -d load-generator
# amati 2 menit, lalu:
docker compose --profile load stop load-generator
```

Container naik mendekati 0,5 core (batas `cpus: 0.50`) dan `ContainerHighCPU` (>0,30 selama 1 menit) _firing_. CPU host naik kecil karena denominatornya jumlah core host (0,5 dibagi N core). Jangan samakan **core usage** dengan **persen seluruh host**.

### Pertanyaan lanjutan — membuktikan kenaikan berasal dari load generator

- Nyalakan lalu matikan generator, perhatikan CPU host turun pada waktu yang sama.
- Pastikan container lain tidak naik (`topk`).
- Selisih host harus ≈ 0,5 core.

> ⚠ **Penyesuaian modul:** modul tidak punya `load-generator`, rule `ContainerHighCPU`, maupun profile `load`. Terapkan Bab 0.2 Langkah 3–4. Catat waktu mulai dan berhenti untuk dicocokkan dengan grafik.

---

## Soal 17 — Negative test `TargetDown`

### Demonstrasi

```bash
docker compose stop node-exporter
```

- Sebelum: `up{job="node-exporter"} = 1` dan `ALERTS{alertname="TargetDown"}` kosong.
- Setelah 75 detik: `up = 0`, alert _pending_ lalu _firing_ (`for: 1m`).
- Setelah `docker compose start node-exporter`: `up` kembali 1 dan alert hilang.

### Pertanyaan lanjutan — bagian alerting yang belum terbukti

**Pengiriman notifikasi.** Stack ini tidak punya Alertmanager atau receiver, jadi yang terbukti hanya perubahan state alert di Prometheus, bukan routing, notifikasi, atau eskalasi.

> ⚠ **Penyesuaian modul:** modul tidak punya rule `TargetDown`, dan job-nya bernama `node`. Terapkan Bab 0.2 Langkah 2–3 dulu. Query pengecekan pakai job yang benar-benar ada di lab Anda.

---

## Soal 18 — Provisioning dan akses Grafana

### Demonstrasi

Tunjukkan `grafana/provisioning/datasources/datasources.yml` dan `provider.yml`, lalu coba edit panel lewat UI.

- `datasources.yml`: Prometheus (uid `prometheus-bab7`, default, `editable: false`) dan PostgreSQL log Bab 6 (role read-only).
- `provider.yml`: `allowUiUpdates: false`, membaca dashboard dari `/var/lib/grafana/dashboards`.

Hasil: edit panel lewat UI **tidak bisa disimpan** karena dashboard terprovisi. Perubahan dilakukan di file JSON dan di-review lewat version control.

Akses Grafana: login admin dari `.env`, anonymous dan sign-up dimatikan, hanya `127.0.0.1:3000`.

### Pertanyaan lanjutan — panel `No data` padahal query jalan di Prometheus

Urutan diagnosis:

1. Rentang waktu dashboard.
2. Datasource dan UID `prometheus-bab7` (**Test datasource**).
3. Query inspector pada panel.
4. Recording rule sehat (Prometheus → Rules), karena panel memakai nama rule.
5. Label dan selector.
6. Log Grafana dan network ke Prometheus.

> ⚠ **Penyesuaian modul:** modul menambah datasource lewat UI dan memakai kredensial default. Terapkan Bab 0.2 Langkah 1 (env admin + anonymous off) dan Langkah 5 (file provisioning). Siapkan minimal satu dashboard JSON di `grafana/dashboards/` agar ada panel yang bisa dicoba edit. Jika dashboard belum ada, panel tidak bisa didemonstrasikan.

---

## Soal 19 — Cardinality dan storage TSDB

### Demonstrasi

```bash
curl -fsS http://127.0.0.1:9090/api/v1/status/tsdb
```

atau Status → TSDB. Tunjukkan `seriesCountByMetricName` teratas **[isi dari lab Anda; umumnya metric `container_*` dari cAdvisor]**.

### Pertanyaan lanjutan — label cAdvisor yang dipertahankan vs dibuang

- **Pertahankan:** `name` (stabil, nama container/service).
- **Buang atau pindahkan ke log:** label yang berubah tiap recreate dan bercardinality tinggi, seperti `id` (path cgroup), `image` dengan hash, dan `container_label_*` yang rinci.

---

## Soal 20 — Integrasi log Bab 6 ke Grafana

### Demonstrasi

```bash
./scripts/connect-bab6.sh
```

Skrip melakukan:

1. Membaca `.env` kedua bab.
2. Menyambungkan container PostgreSQL Bab 6 ke `bab7-monitoring-net` dengan alias `postgres-db`.
3. Membuat role read-only `grafana_reader` (`CONNECT`, `USAGE` pada schema `logs`, `SELECT` pada tabel).
4. Me-restart Grafana.

Lalu di Explore: `WHERE $__timeFilter(time)` pada rentang yang sama dengan kenaikan CPU.

Kesimpulan yang aman: log dan CPU **berkorelasi secara waktu**, tetapi itu belum cukup untuk menyatakan **sebab-akibat**.

### Pertanyaan lanjutan — "Network Bab 7 belum tersedia"

Network `bab7-monitoring-net` belum ada karena stack Bab 7 belum dijalankan. Perbaikan: jalankan `docker compose up -d` di folder bab-7 (dan pastikan Bab 6 berjalan), lalu ulangi skrip.

> ⚠ **Penyesuaian modul:** modul hanya menyebut "Grafana dapat membaca data source PostgreSQL" tanpa skrip dan tanpa network bersama. Gunakan Bab 0.2 Langkah 6 (pengganti manual). Contoh query Explore:
>
> ```sql
> SELECT time, tag, LEFT(data->>'log', 120) AS log_preview
> FROM logs.fluentbit
> WHERE $__timeFilter(time)
> ORDER BY time DESC LIMIT 50;
> ```

---

# STUDI KASUS PORTAL PMB

> **Konteks:** beberapa pendaftaran berhenti di status `PROCESSING` padahal semua container hidup. Setiap kesimpulan harus didukung event di centralized logging dan data di database transaksi, bukan status container.
>
> ⚠ **Penyesuaian modul:** seluruh skenario ini tidak ada di modul Bab 6/7. Lihat Bab 0.3 sebelum sesi.

## Soal 21 — Tahap A: Baseline transaksi normal

### Demonstrasi

```bash
./scripts/demo-normal.sh     # catat Request ID dan waktunya
```

Kronologi dari centralized log (query `incident-analysis.sql` dengan `request_id` tersebut), urutan yang diharapkan:

1. `registration_submitted` (registration-api)
2. `verification_started` (worker, mode `none`)
3. `verification_completed` (attempt 1)

Status di database transaksi:

```sql
SELECT public_id, status, attempt, failure_code FROM registrations WHERE request_id = 'REQ-...';
-- status: VERIFIED
```

### Pertanyaan lanjutan — event time vs waktu collector vs waktu DB

- **Event time:** timestamp (UTC) di dalam JSON, saat aplikasi menulis event.
- **Waktu terima collector:** kolom `time` pada tabel.
- **Waktu simpan DB:** saat `INSERT`, bisa tertunda jika ada buffer atau retry.

Untuk kronologi pakai **event time** (timestamp di JSON), karena `time` bisa bergeser saat backlog.

---

## Soal 22 — Tahap B: Failure injection `timeout`

### Demonstrasi

```bash
./scripts/set-incident.sh timeout
./scripts/demo-incident.sh
```

Dari log (tanpa membuka kode): service pertama yang menghasilkan WARN adalah **verification-worker** (`verification_retry` attempt 1, 2, 3), lalu ERROR `dependency_timeout` dengan `error_code: VERIFY-TIMEOUT`.

Dampak di database: `status = FAILED`, `attempt = 3`, `failure_code = VERIFY-TIMEOUT`.

### Pertanyaan lanjutan — mengapa `Up` tidak membuktikan fungsi bisnis sehat

`Up` hanya berarti proses berjalan. Worker bisa hidup sementara verifikasi gagal dan pendaftaran tidak selesai. Buktinya ada di **event log** dan **status transaksi**.

---

## Soal 23 — Tahap C: Recovery dan transaksi yang gagal

### Demonstrasi

```bash
./scripts/set-incident.sh none
./scripts/demo-normal.sh
```

Transaksi baru `VERIFIED`, transaksi insiden tetap `FAILED`. Alasannya: worker hanya mengklaim baris berstatus `SUBMITTED` (`WHERE status = 'SUBMITTED'`), dan `FAILED` adalah status akhir yang tidak dijadwalkan ulang.

Usulan: **retry terkontrol** (batas percobaan, backoff, idempotency key) dan **dead-letter queue** untuk transaksi yang melampaui batas, ditinjau manual.

### Pertanyaan lanjutan — risiko `FAILED` diproses ulang otomatis tanpa idempotensi

- Efek ganda (verifikasi atau notifikasi dobel).
- Data tidak konsisten.
- _Retry storm_ yang membebani dependency.

---

## Soal 24 — Tahap D: Resilience logging saat `log-db` mati

### Demonstrasi

```bash
docker compose stop log-db
./scripts/demo-normal.sh
docker compose logs --tail 100 fluent-bit      # retry ke log-db
curl -fsS http://127.0.0.1:2020/api/v1/metrics # retries naik
docker compose start log-db
```

Setelah `log-db` hidup, jumlah `logs.fluentbit` naik (backlog terkirim dari buffer filesystem). Deteksi record ganda:

```sql
WITH e AS (SELECT (data->>'log')::jsonb AS ev FROM logs.fluentbit
           WHERE tag LIKE 'pmb.%' AND left(ltrim(data->>'log'),1)='{')
SELECT ev->>'request_id', ev->>'event', ev->>'attempt', count(*)
FROM e GROUP BY 1,2,3 HAVING count(*) > 1;
```

### Pertanyaan lanjutan — buffer mencapai 256 MB

Fluent Bit membuang **chunk paling lama** agar yang baru muat (`storage.total_limit_size`), sehingga **log lama hilang**. Hal ini harus terlihat lewat metrik atau log Fluent Bit.

---

## Soal 25 — Privasi, kepercayaan, dan kesiapan produksi

### Demonstrasi

```bash
./scripts/verify.sh                  # regex password|authorization|bearer|cookie|session_secret|api_key pada log -> harus 0
./scripts/collect-evidence.sh        # menghasilkan reports/config-sha256.txt
cat reports/config-sha256.txt
```

**Mengapa nama dan email tidak masuk structured log:** prinsip **minimisasi data**. Log dibaca banyak pihak, disalin ke buffer, backup, dan Grafana, serta sulit dihapus. Aplikasi memakai `user_ref` (pseudonim), dan `redact()` di `common.py` menyensor key sensitif (password, token, cookie, dst.).

**Memperlakukan `request_id` dari client:** tidak dipercaya mentah. `normalize_request_id` memvalidasi dengan regex `^[A-Za-z0-9][A-Za-z0-9._:-]{7,127}$`. Jika tidak valid, dibuat ID baru `REQ-<uuid>`. Ini mencegah pemalsuan korelasi audit dan _log injection_.

### Pertanyaan lanjutan — kontrol tambahan sebelum produksi

- TLS untuk Forward dan database.
- Secret manager dengan rotasi.
- Akun analyst read-only dan pencabutan hak `CREATE` pada akun ingest.
- Retensi, partisi, dan arsip.
- Alert untuk `dropped_records` dan `retries`.
- Autentikasi endpoint monitoring.
- Backend yang lebih skalabel (Loki, OpenSearch, ClickHouse).

---

# Checklist Sebelum Sesi

- [ ] Bab 0.1 Langkah 1–5 sudah diterapkan; `docker compose ps --all` menunjukkan `fluent-bit` Up dan 3 generator Exited (0).
- [ ] `curl http://127.0.0.1:2020/api/v1/health` berhasil.
- [ ] `fluentbit_user` ada, `SHOW search_path` → `logs, public`.
- [ ] Bab 0.2 Langkah 1–5 sudah diterapkan; `ss -lntp` hanya menampilkan `127.0.0.1:9090` dan `127.0.0.1:3000`.
- [ ] `promtool check config` dan `promtool test rules` lulus.
- [ ] Datasource Grafana terprovisi dan dashboard contoh tampil.
- [ ] Repo/skrip Portal PMB tersedia dan `demo-normal.sh` berhasil minimal sekali.
- [ ] Tempat evidence (`reports/`) sudah dibuat; semua bagian **[isi dari lab Anda]** siap diganti dengan output nyata.
