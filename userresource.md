# Dokumentasi Implementasi Monitoring Resource Per User

**Teknologi:** Prometheus, Grafana, Process Exporter, Docker, Azure Virtual Machine  
**Tanggal:** 15 Juli 2026  
**Disusun Oleh:** Ilham Lii Assidaq  

---

## 1. Pendahuluan

Dokumentasi ini dibuat untuk mencatat langkah-langkah implementasi sistem pemantauan (*monitoring*) penggunaan *resource* server (CPU, RAM, dan Disk I/O) secara spesifik per *user* Linux. Metrik dikumpulkan menggunakan **Process Exporter**, disimpan di database **Prometheus** (berjalan di dalam kontainer Docker), dan divisualisasikan menggunakan dashboard **Grafana**.

---

## 2. Arsitektur Jaringan & Firewall (Azure NSG)

Agar server Prometheus dapat menarik (*scrape*) data metrik dari target mesin virtual (VM) tempat Process Exporter terinstal, port komunikasi khusus harus dibuka pada firewall Azure.

1. Masuk ke **Azure Portal**.
2. Buka konfigurasi **Network Security Group (NSG)** pada VM Target.
3. Tambahkan **Inbound Security Rule** baru dengan spesifikasi berikut:

   * **Source:** `Any` (atau IP spesifik server Prometheus untuk keamanan ekstra)
   * **Destination Port Ranges:** `9256`
   * **Protocol:** `TCP`
   * **Action:** `Allow`
   * **Name:** `Allow_Process_Exporter`

---

## 3. Instalasi dan Konfigurasi Process Exporter (Server Target)

Process Exporter dipasang di server target untuk membaca informasi penggunaan proses dari direktori `/proc` Linux dan menyediakannya dalam format metrik Prometheus.

### Langkah-Langkah:

1. Unduh dan ekstrak *binary* Process Exporter versi terbaru.

   ```bash

   wget https://github.com/ncabatoff/process-exporter/releases/download/v0.7.10/process-exporter-0.7.10.linux-amd64.tar.gz

   tar -xvzf process-exporter-0.7.10.linux-amd64.tar.gz

   sudo mv process-exporter-0.7.10.linux-amd64/process-exporter /usr/local/bin/

   ```


2. Buat berkas konfigurasi bernama `process-exporter.yml` untuk mengelompokkan proses berdasarkan nama *user* (`groupname`):

   ```bash

   sudo mkdir -p /etc/process-exporter

   sudo nano /etc/process-exporter/process-exporter.yml

   ```

   Isi berkas `/etc/process-exporter/process-exporter.yml` dengan konfigurasi berikut:

   ```yaml

   process_names:

     - name: "{{.Username}}"

       cmdline:

         - '.+'

   ```


3. Konfigurasikan Process Exporter agar berjalan secara otomatis di latar belakang sebagai layanan systemd dengan membuat berkas `/etc/systemd/system/process-exporter.service`:

   ```bash

   sudo nano /etc/systemd/system/process-exporter.service

   ```

   Isi dengan konfigurasi layanan berikut:

   ```ini

   [Unit]

   Description=Process Exporter

   After=network.target

  

   [Service]

   User=root

   Type=simple

   ExecStart=/usr/local/bin/process-exporter -config.path /etc/process-exporter/process-exporter.yml

   Restart=always

  

   [Install]

   WantedBy=multi-user.target

   ```

  

4. Jalankan dan aktifkan layanan systemd:

   ```bash

   sudo systemctl daemon-reload

   sudo systemctl start process-exporter

   sudo systemctl enable process-exporter

   ```

  

5. Pastikan port aktif dan mengeluarkan data mentah dengan mengakses URL berikut melalui browser atau curl:

   ```plaintext

   http://<IP_SERVER_TARGET>:9256/metrics

   ```

  

---


## 4. Konfigurasi Prometheus (Docker Environment)

Setelah exporter aktif di sisi target, Prometheus harus dikonfigurasi untuk mendaftarkan target baru tersebut ke dalam daftar scraping.

### Langkah-Langkah:

1. Buka berkas konfigurasi Prometheus utama di server monitoring:

   ```bash

   sudo nano /etc/prometheus/prometheus.yml

   ```


2. Tambahkan *job block* baru di bagian paling bawah di bawah instruksi `scrape_configs`:

   ```yaml

     - job_name: 'process-exporter-users'

       static_configs:

         - targets: ['20.48.225.29:9256']

   ```

   *Catatan: Ganti `20.48.225.29` dengan IP Server Target Anda jika berbeda.*

  
3. Simpan perubahan berkas di editor `nano` dengan menekan `Ctrl + O`, lalu `ENTER`, dan keluar menggunakan `Ctrl + X`.
  

4. Lakukan restart pada kontainer Docker Prometheus agar konfigurasi baru dapat diterapkan (ini harus karena prometheus ada di dalam docker):

   ```bash

   sudo docker restart prometheus

   ```
  

5. Validasi status target dengan membuka halaman web UI Prometheus pada bagian **Status -> Targets** (`http://<IP_PROMETHEUS>:9090/targets`). Pastikan entri `process-exporter-users` berstatus hijau (**UP**).


---

  

## 5. Visualisasi Akhir di Grafana Dashboard

Data yang sudah masuk ke database Prometheus kemudian dipetakan ke dalam bentuk grafik interaktif pada dashboard Grafana.
  

Agar query tidak terblokir oleh filter variabel bawaan dari template dashboard bawaan (seperti Node Exporter Full), digunakan klausa filter instan `{job="process-exporter-users"}` serta optimasi visualisasi *Standard Options*.


Berikut adalah ringkasan konfigurasi 4 panel utama yang berhasil dibangun:

  

| No | Nama Panel | Formula PromQL (Query A) | Konfigurasi Unit (Standard Options) |

|---|---|---|---|

| 1 | CPU Usage per User (%) | `sum(rate(namedprocess_namegroup_cpu_seconds_total{job="process-exporter-users"}[5m])) by (groupname) * 100` | Misc → Percent (0-100) |

| 2 | Memory Usage per User | `sum(namedprocess_namegroup_memory_bytes{job="process-exporter-users", memtype="resident"}) by (groupname)` | Data (IEC) → Bytes (B) (Otomatis terkonversi ke MiB/GiB) |

| 3 | Disk Read Rate per User | `sum(rate(namedprocess_namegroup_read_bytes_total{job="process-exporter-users"}[5m])) by (groupname)` | Data rate → Bytes/second (B/s) |

| 4 | Disk Write Rate per User | `sum(rate(namedprocess_namegroup_write_bytes_total{job="process-exporter-users"}[5m])) by (groupname)` | Data rate → Bytes/second (B/s) |

  

### Langkah Pembuatan Panel di Grafana:

1. Pada Dashboard utama, klik **+ Add** → **Visualization**.

2. Pilih Data Source **prometheus** dan pindah ke mode **Code**.

3. Masukkan salah satu query di atas pada kolom input query.

4. Klik pada menu sebelah kanan, temukan bagian **Standard options**, lalu atur **Unit** sesuai tabel di atas.

5. Beri judul panel di **Panel options** → **Title**.

6. Klik **Apply** atau **Save** di pojok kanan atas untuk menyimpan panel ke dashboard.



Berikut adalah panel yang sudah jadi
<img width="1570" height="781" alt="image" src="https://github.com/user-attachments/assets/37ff66b0-2b1b-47dd-ab5e-7de0815c07c3" />


---

Terimakasih
