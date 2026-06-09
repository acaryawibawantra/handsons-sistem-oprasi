# Hands-On 3: Process Lifecycle

**Nama:** I Dewa Nyoman Acarya Wibawantra
**NRP:** 5025251222
**Mata Kuliah:** Sistem Operasi

---

## Task 1 - Creating and Observing a Simple Process

### Tujuan
Memahami cara membuat proses dari Bash, menjalankannya di foreground dan background, serta mengamati PID dan status dasarnya menggunakan perintah `ps`.

### Script - `task1_process_basic.sh`

```bash
#!/bin/bash

echo "Running a sleep process in the background..."
sleep 30 &
PID_BG=$!

echo "Background process PID: $PID_BG"
echo "Displaying process status with ps:"
ps -p $PID_BG -o pid,ppid,stat,cmd

echo "Waiting for 3 seconds..."
sleep 3

echo "Checking process status again:"
ps -p $PID_BG -o pid,ppid,stat,cmd

echo "Stopping the process..."
kill $PID_BG

echo "Status after kill:"
ps -p $PID_BG -o pid,ppid,stat,cmd
```

### Cara Menjalankan

```bash
chmod +x task1_process_basic.sh
./task1_process_basic.sh
```

### Hasil Eksekusi

```
Running a sleep process in the background...
Background process PID: 48217
Displaying process status with ps:
    PID    PPID STAT CMD
  48217   48210 S    sleep 30
Waiting for 3 seconds...
Checking process status again:
    PID    PPID STAT CMD
  48217   48210 S    sleep 30
Stopping the process...
Status after kill:
    PID    PPID STAT CMD
```

### Analisis

Proses berhasil dibuat di background dan mendapat PID 48217. Perintah `sleep 30 &` menciptakan proses baru yang berjalan terpisah dari shell utama. Operator `&` membuat shell tidak menunggu proses tersebut selesai, sehingga eksekusi script bisa berlanjut.

- Variabel `$!` secara otomatis menyimpan PID dari proses background terakhir, memungkinkan kita merujuk proses tersebut di baris selanjutnya.
- Status `S` (sleeping) menunjukkan bahwa proses `sleep` sedang menunggu — bukan berarti proses mati, melainkan sedang idle menunggu timer selesai.
- Kolom PPID menunjukkan 48210, yaitu PID dari script itu sendiri, membuktikan hubungan parent-child antara shell script dan proses yang ia buat.
- Setelah perintah `kill` dikirim, `ps` tidak lagi menampilkan proses tersebut karena proses sudah benar-benar dihentikan (terminated).
- Percobaan ini memperlihatkan bahwa setiap proses di Linux memiliki identitas unik (PID) dan bisa dikontrol secara eksplisit oleh proses lain.

---

## Task 2 - Monitoring the Status of Another Process

### Tujuan
Memahami bagaimana sebuah script dapat memeriksa status proses lain secara berkala dan mengambil keputusan berdasarkan status tersebut, sebagai dasar koordinasi antar-proses.

### Script - `task2_check_process_status.sh`

```bash
#!/bin/bash

echo "Starting worker process..."

sleep 10 &

WORKER_PID=$!

echo "Worker PID: $WORKER_PID"

while true
do
    if kill -0 $WORKER_PID 2>/dev/null; then
        echo "[$(date +%H:%M:%S)] Worker is still running..."
    else
        echo "[$(date +%H:%M:%S)] Worker has finished."
        break
    fi

    sleep 2
done

echo "Monitoring script finished."
```

### Cara Menjalankan

```bash
chmod +x task2_check_process_status.sh
./task2_check_process_status.sh
```

### Hasil Eksekusi

```
Starting worker process...
Worker PID: 48305
[15:04:12] Worker is still running...
[15:04:14] Worker is still running...
[15:04:16] Worker is still running...
[15:04:18] Worker is still running...
[15:04:20] Worker is still running...
[15:04:22] Worker has finished.
Monitoring script finished.
```

### Analisis

Script monitoring berhasil mendeteksi kapan proses worker selesai. Mekanisme `kill -0` digunakan bukan untuk menghentikan proses, melainkan untuk mengecek apakah proses dengan PID tertentu masih ada di sistem.

- Proses worker (`sleep 10`) berjalan selama 10 detik. Selama waktu itu, loop monitoring melakukan pengecekan setiap 2 detik, sehingga terlihat 5 kali pesan "still running" sebelum worker selesai.
- Redirect `2>/dev/null` berfungsi menyembunyikan pesan error yang muncul ketika `kill -0` gagal menemukan proses (karena sudah terminated) — tanpa ini, terminal akan menampilkan error "No such process".
- Teknik polling seperti ini merupakan bentuk sederhana dari process monitoring. Meski tidak seefisien mekanisme event-driven, polling cukup efektif untuk kebutuhan scripting dasar.
- Pola ini mirip dengan cara kerja health-check pada sistem nyata: sebuah service secara periodik memeriksa apakah service lain masih hidup, lalu bereaksi jika service tersebut sudah berhenti.

---

## Task 3 - Interacting Between Processes Using a Signal File

### Tujuan
Memahami bagaimana dua proses dapat berkomunikasi secara sederhana menggunakan file sebagai media sinyal — konsep dasar Inter-Process Communication (IPC) berbasis filesystem.

### Script - `task3_process_interaction.sh`

```bash
#!/bin/bash

SIGNAL_FILE="/tmp/process_done.signal"

rm -f "$SIGNAL_FILE"

producer() {
    echo "Producer: starting work..."

    sleep 5

    echo "Producer: work completed." > "$SIGNAL_FILE"

    echo "Producer: signal file created."
}

consumer() {
    echo "Consumer: waiting for signal from producer..."

    while [ ! -f "$SIGNAL_FILE" ]
    do
        echo "Consumer: signal not found yet, checking again..."
        sleep 1
    done

    echo "Consumer: signal received."

    echo "Signal content:"

    cat "$SIGNAL_FILE"
}

producer &

PID_PRODUCER=$!

consumer &

PID_CONSUMER=$!

wait $PID_PRODUCER

wait $PID_CONSUMER

rm -f "$SIGNAL_FILE"

echo "All processes completed."
```

### Cara Menjalankan

```bash
chmod +x task3_process_interaction.sh
./task3_process_interaction.sh
```

### Hasil Eksekusi

```
Producer: starting work...
Consumer: waiting for signal from producer...
Consumer: signal not found yet, checking again...
Consumer: signal not found yet, checking again...
Consumer: signal not found yet, checking again...
Consumer: signal not found yet, checking again...
Consumer: signal not found yet, checking again...
Producer: signal file created.
Consumer: signal received.
Signal content:
Producer: work completed.
All processes completed.
```

### Analisis

Dua proses berhasil berkoordinasi melalui file sinyal tanpa mekanisme IPC yang kompleks. Producer dan consumer berjalan secara bersamaan (concurrent), namun consumer menunggu hingga producer menyelesaikan tugasnya sebelum melanjutkan.

- Producer bekerja selama 5 detik (simulasi tugas berat), kemudian membuat file `/tmp/process_done.signal`. Consumer melakukan polling setiap 1 detik menggunakan `[ ! -f "$SIGNAL_FILE" ]` untuk mengecek keberadaan file tersebut.
- Output menunjukkan consumer melakukan 5 kali pengecekan (5 detik) sebelum akhirnya menemukan signal file — sesuai dengan durasi kerja producer.
- Pendekatan file-based signaling ini sederhana tapi efektif sebagai pengantar konsep IPC. Kelemahannya adalah overhead I/O pada disk dan potensi race condition jika tidak ditangani dengan hati-hati.
- Perintah `wait` pada akhir script memastikan shell utama tidak berhenti sebelum kedua proses (producer dan consumer) selesai, sehingga cleanup file signal bisa dilakukan dengan aman.
- Dalam praktik nyata, mekanisme serupa digunakan pada pipeline data: satu stage menulis marker file untuk memberi tahu stage berikutnya bahwa data sudah siap diproses.

---

## Task 4 - Observing the Process Lifecycle

### Tujuan
Mengamati tahapan siklus hidup sebuah proses: dibuat (created), berjalan (running), menerima sinyal (signal received), melakukan cleanup, dan akhirnya berhenti (terminated).

### Script - `task4_process_lifecycle.sh`

```bash
#!/bin/bash

cleanup() {

    echo "Process received termination signal."

    echo "Performing cleanup before exiting..."

    exit 0

}

trap cleanup SIGTERM SIGINT

echo "Process started with PID $$"

COUNT=1

while true
do
    echo "Process is running... iteration $COUNT"

    COUNT=$((COUNT + 1))

    sleep 2
done
```

### Cara Menjalankan

```bash
chmod +x task4_process_lifecycle.sh
./task4_process_lifecycle.sh
```

Jalankan di Terminal 1, lalu kirim sinyal dari Terminal 2.

### Hasil Eksekusi

**Terminal 1 — menjalankan script:**

```
$ ./task4_process_lifecycle.sh
Process started with PID 48523
Process is running... iteration 1
Process is running... iteration 2
Process is running... iteration 3
Process is running... iteration 4
Process is running... iteration 5
Process is running... iteration 6
Process received termination signal.
Performing cleanup before exiting...
```

**Terminal 2 — memonitor dan mengirim sinyal:**

```
$ ps -p 48523 -o pid,ppid,stat,cmd
    PID    PPID STAT CMD
  48523   48401 S+   /bin/bash ./task4_process_lifecycle.sh

$ kill -TERM 48523
```

### Analisis

Proses menunjukkan siklus hidup yang lengkap dan terkontrol: start → running → signal → cleanup → exit. Perintah `trap` memungkinkan proses menangkap sinyal tertentu dan menjalankan fungsi handler sebelum benar-benar berhenti.

- Variabel `$$` menampilkan PID dari shell script yang sedang berjalan (48523). PID ini diperlukan agar kita bisa berinteraksi dengan proses dari terminal lain.
- Status `S+` pada output `ps` menunjukkan proses dalam keadaan sleeping dan berada di foreground process group (`+`). Ini normal karena loop utama menggunakan `sleep 2` di setiap iterasi.
- `trap cleanup SIGTERM SIGINT` mendaftarkan fungsi `cleanup` sebagai handler untuk dua sinyal: SIGTERM (sinyal terminasi standar dari `kill`) dan SIGINT (sinyal dari Ctrl+C).
- Ketika `kill -TERM 48523` dikirim dari terminal kedua, proses tidak langsung mati. Sebaliknya, fungsi `cleanup` dieksekusi terlebih dahulu — ini membuktikan bahwa proses dapat melakukan graceful shutdown.
- Konsep graceful shutdown sangat penting dalam sistem nyata. Contohnya, sebuah web server perlu menyelesaikan request yang sedang diproses dan menutup koneksi database sebelum benar-benar berhenti, bukan langsung mematikan semua koneksi secara paksa.

---

## Task 5 - Inter-Process Dependency in a Simple Workflow

### Tujuan
Memahami bahwa sebuah proses dapat bergantung pada hasil proses lain, dan Bash mampu mengatur urutan eksekusi (dependency) menggunakan `wait` dan exit status.

### Script - `task5_process_dependency.sh`

```bash
#!/bin/bash

DATA_FILE="/tmp/sample_data.txt"

RESULT_FILE="/tmp/report.txt"

rm -f "$DATA_FILE" "$RESULT_FILE"

download_data() {

    echo "Step 1: Collecting data..."

    sleep 3

    echo -e "value1\nvalue2\nvalue3" > "$DATA_FILE"

    echo "Step 1 completed: data saved in $DATA_FILE"

}

validate_data() {

    echo "Step 2: Validating data..."

    if [ ! -f "$DATA_FILE" ]; then

        echo "ERROR: data is not available yet."

        return 1

    fi

    if [ ! -s "$DATA_FILE" ]; then

        echo "ERROR: data file is empty."

        return 1

    fi

    echo "Data is valid."

    return 0

}

generate_report() {

    echo "Step 3: Generating report..."

    wc -l "$DATA_FILE" > "$RESULT_FILE"

    echo "Report created in $RESULT_FILE"

}

download_data &

PID_DOWNLOAD=$!

echo "Waiting for download process to finish..."

wait $PID_DOWNLOAD

validate_data

STATUS_VALID=$?

if [ $STATUS_VALID -eq 0 ]; then

    generate_report

    echo "Workflow completed successfully."

    echo "Report content:"

    cat "$RESULT_FILE"

else

    echo "Workflow failed because validation did not pass."

fi
```

### Cara Menjalankan

```bash
chmod +x task5_process_dependency.sh
./task5_process_dependency.sh
```

### Hasil Eksekusi

```
Waiting for download process to finish...
Step 1: Collecting data...
Step 1 completed: data saved in /tmp/sample_data.txt
Step 2: Validating data...
Data is valid.
Step 3: Generating report...
Report created in /tmp/report.txt
Workflow completed successfully.
Report content:
3 /tmp/sample_data.txt
```

### Analisis

Workflow tiga tahap berhasil dijalankan secara berurutan dengan kontrol dependensi yang jelas. Setiap step hanya berjalan jika step sebelumnya telah selesai dan berhasil, mencerminkan pola pipeline yang umum dalam dunia nyata.

- Tahap 1 (download_data) dijalankan di background, kemudian `wait $PID_DOWNLOAD` memastikan script utama menunggu hingga proses tersebut selesai. Tanpa `wait`, validasi bisa berjalan sebelum data tersedia — mengakibatkan kegagalan workflow.
- Tahap 2 (validate_data) melakukan dua pengecekan: apakah file ada (`-f`) dan apakah file tidak kosong (`-s`). Ini menunjukkan defensive programming — script tidak hanya menganggap data pasti tersedia.
- Exit status (`$?`) dari validasi digunakan sebagai decision point: jika 0 (sukses), lanjut ke tahap 3; jika bukan 0, workflow dihentikan. Ini adalah mekanisme standar di Unix untuk mengkomunikasikan hasil eksekusi antar-proses.
- Output `3 /tmp/sample_data.txt` dari `wc -l` menunjukkan file berisi 3 baris (value1, value2, value3), mengonfirmasi bahwa seluruh pipeline data berjalan dengan benar dari pengumpulan hingga pelaporan.
- Pola sequential dependency ini adalah fondasi dari sistem seperti cron jobs, CI/CD pipeline, dan ETL (Extract-Transform-Load) — di mana setiap tahap bergantung pada keberhasilan tahap sebelumnya.

---

## Kesimpulan

| Task | Topik | Pelajaran Utama |
|------|-------|-----------------|
| 1 | Creating & Observing Process | Setiap proses punya PID unik, bisa dikontrol via `ps` dan `kill` |
| 2 | Monitoring Process Status | `kill -0` untuk mengecek keberadaan proses tanpa menghentikannya |
| 3 | Inter-Process Communication | File sebagai media komunikasi sederhana antar-proses |
| 4 | Process Lifecycle | `trap` memungkinkan graceful shutdown saat menerima sinyal |
| 5 | Process Dependency | `wait` dan exit status mengatur urutan eksekusi workflow |

Lima percobaan ini membangun pemahaman bertahap tentang bagaimana proses hidup, berinteraksi, dan mati di dalam sistem operasi Linux. Bash bukan hanya alat untuk menjalankan perintah, tetapi juga control tool yang mampu mengelola lifecycle, koordinasi, dan dependensi antar-proses.
