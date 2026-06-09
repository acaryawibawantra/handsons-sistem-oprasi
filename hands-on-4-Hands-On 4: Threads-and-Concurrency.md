# Hands-On 4: Threads and Concurrency (Bash & C)

**Nama:** I Dewa Nyoman Acarya Wibawantra
**NRP:** 5025251222
**Mata Kuliah:** Sistem Operasi

---

## Part A — Bash Scripting Labs

---

### Lab 1: Sequential vs Concurrent Execution

#### Tujuan
Membandingkan eksekusi berurutan dengan eksekusi konkuren menggunakan background process (`&`) dan `wait`.

#### Script - `lab1.sh`

```bash
#!/bin/bash

task() {
    echo "Task $1 started"
    sleep 3
    echo "Task $1 finished"
}

echo "Sequential execution"
task A
task B
task C

echo "Concurrent execution"
task A &
task B &
task C &

wait
echo "All tasks completed"
```

#### Cara Menjalankan

```bash
chmod +x lab1.sh
./lab1.sh
```

#### Hasil Eksekusi

```
Sequential execution
Task A started
Task A finished
Task B started
Task B finished
Task C started
Task C finished
Concurrent execution
Task A started
Task B started
Task C started
Task A finished
Task C finished
Task B finished
All tasks completed
```

#### Analisis

Eksekusi sekuensial memakan waktu ~9 detik (3×3), sedangkan konkuren hanya ~3 detik karena ketiga task berjalan bersamaan. Urutan output pada bagian konkuren tidak selalu sama karena penjadwalan oleh OS bersifat nondeterministik. Ini menunjukkan bahwa concurrency menghemat waktu ketika task-task saling independen dan tidak berbagi data.

---

### Lab 2: Foreground & Background Tasks

#### Tujuan
Menunjukkan bagaimana background task berjalan bersamaan dengan foreground yang menerima input user.

#### Script - `lab2.sh`

```bash
#!/bin/bash

background_task() {
    while true; do
        echo "[Background] Periodic task is running..."
        sleep 5
    done
}

foreground_task() {
    while true; do
        read -p "Enter command: " cmd
        echo "You typed: $cmd"
    done
}

background_task &
foreground_task
```

#### Cara Menjalankan

```bash
chmod +x lab2.sh
./lab2.sh
```

#### Hasil Eksekusi

```
[Background] Periodic task is running...
Enter command: hello
You typed: hello
Enter command: test
You typed: test
[Background] Periodic task is running...
Enter command: ^C
```

#### Analisis

Foreground task tetap responsif menerima input meskipun background task mencetak pesan setiap 5 detik. Kadang output background muncul di tengah prompt — ini wajar karena keduanya berbagi terminal yang sama. Pola ini mencerminkan cara aplikasi nyata (misal text editor dengan autosave) mempertahankan responsivitas.

---

### Lab 3: Race Condition pada Shared File

#### Tujuan
Mendemonstrasikan race condition ketika tiga proses mengakses file counter secara bersamaan tanpa sinkronisasi.

#### Script - `lab3.sh`

```bash
#!/bin/bash

echo 0 > counter.txt

increment() {
    for i in {1..100}; do
        count=$(cat counter.txt)
        count=$((count + 1))
        echo $count > counter.txt
    done
}

increment &
increment &
increment &

wait
echo "Final counter value:"
cat counter.txt
```

#### Cara Menjalankan

```bash
chmod +x lab3.sh
./lab3.sh
```

#### Hasil Eksekusi (3x percobaan)

```
$ ./lab3.sh
Final counter value:
189

$ ./lab3.sh
Final counter value:
212

$ ./lab3.sh
Final counter value:
167
```

#### Analisis

Hasil seharusnya 300 (3×100), tapi selalu di bawah itu dan berbeda tiap eksekusi. Ini terjadi karena operasi read-increment-write tidak atomik — dua proses bisa membaca nilai yang sama (misal 50), keduanya menghitung 51, dan keduanya menulis 51, sehingga satu increment hilang (lost update). Ini membuktikan bahwa shared resource tanpa sinkronisasi akan menghasilkan data yang salah.

---

### Lab 4: Fixing Race Condition dengan Lock

#### Tujuan
Memperbaiki race condition menggunakan directory lock (`mkdir`) sebagai mekanisme mutual exclusion sederhana.

#### Script - `lab4.sh`

```bash
#!/bin/bash

echo 0 > counter.txt
lockdir="lockdir"

increment() {
    for i in {1..100}; do
        while ! mkdir "$lockdir" 2>/dev/null; do
            sleep 0.01
        done

        count=$(cat counter.txt)
        count=$((count + 1))
        echo $count > counter.txt

        rmdir "$lockdir"
    done
}

increment &
increment &
increment &

wait
echo "Final counter value:"
cat counter.txt
```

#### Cara Menjalankan

```bash
chmod +x lab4.sh
./lab4.sh
```

#### Hasil Eksekusi

```
Final counter value:
300
```

#### Analisis

Hasil sekarang selalu konsisten 300. `mkdir` bersifat atomik di filesystem — hanya satu proses yang bisa berhasil membuat directory pada satu waktu, sehingga berfungsi sebagai mutex. Proses yang gagal `mkdir` akan retry dengan `sleep 0.01`. Trade-off-nya: correctness terjamin, tapi performa sedikit lebih lambat karena critical section menjadi serial.

---

### Lab 5: Blocking Simulation

#### Tujuan
Menunjukkan bahwa satu task yang ter-block tidak menghentikan task lain yang berjalan konkuren.

#### Script - `lab5.sh`

```bash
#!/bin/bash

(sleep 5; echo "Task A completed after waiting") &

(for i in {1..5}; do
    echo "Task B is working: step $i"
    sleep 1
done) &

wait
echo "Both tasks finished"
```

#### Cara Menjalankan

```bash
chmod +x lab5.sh
./lab5.sh
```

#### Hasil Eksekusi

```
Task B is working: step 1
Task B is working: step 2
Task B is working: step 3
Task B is working: step 4
Task B is working: step 5
Task A completed after waiting
Both tasks finished
```

#### Analisis

Task A diam selama 5 detik (simulasi I/O blocking), sementara Task B terus bekerja mencetak progress tiap detik. Keduanya selesai hampir bersamaan (~5 detik). Ini menunjukkan bahwa blocking bersifat lokal pada task yang melakukannya — task lain tetap bisa berjalan. Konsep ini sama dengan cara OS menjadwalkan thread lain ketika satu thread menunggu I/O.

---

## Part B — C Programming Labs (pthreads)

---

### Lab 6: Sequential vs Multithread (C)

#### Tujuan
Membandingkan pemanggilan fungsi biasa dengan eksekusi menggunakan thread di C.

#### Script - `lab6.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* task(void* arg) {
    char* name = (char*) arg;
    printf("Task %s started\n", name);
    sleep(3);
    printf("Task %s finished\n", name);
    return NULL;
}

int main() {
    pthread_t t1, t2, t3;

    printf("Multithread execution\n");
    pthread_create(&t1, NULL, task, "A");
    pthread_create(&t2, NULL, task, "B");
    pthread_create(&t3, NULL, task, "C");

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    printf("All threads completed\n");
    return 0;
}
```

#### Cara Menjalankan

```bash
gcc lab6.c -o lab6 -pthread
./lab6
```

#### Hasil Eksekusi

```
Multithread execution
Task A started
Task C started
Task B started
Task A finished
Task B finished
Task C finished
All threads completed
```

#### Analisis

Ketiga task dimulai hampir bersamaan dan selesai dalam ~3 detik total, bukan ~9 detik. Urutan start/finish bervariasi antar eksekusi karena thread scheduling dikontrol oleh OS kernel, bukan programmer. `pthread_join` memastikan `main()` menunggu semua thread selesai sebelum program keluar. Nondeterminisme urutan ini aman selama thread-thread tidak berbagi data.

---

### Lab 7: Background Thread in One Process

#### Tujuan
Menjalankan background thread dan main thread secara bersamaan dalam satu proses.

#### Script - `lab7.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* background(void* arg) {
    while (1) {
        printf("[Background thread] Running periodic work...\n");
        sleep(3);
    }
    return NULL;
}

int main() {
    pthread_t bg;
    pthread_create(&bg, NULL, background, NULL);

    for (int i = 1; i <= 10; i++) {
        printf("[Main thread] Working step %d\n", i);
        sleep(1);
    }
    return 0;
}
```

#### Cara Menjalankan

```bash
gcc lab7.c -o lab7 -pthread
./lab7
```

#### Hasil Eksekusi

```
[Background thread] Running periodic work...
[Main thread] Working step 1
[Main thread] Working step 2
[Main thread] Working step 3
[Background thread] Running periodic work...
[Main thread] Working step 4
[Main thread] Working step 5
[Main thread] Working step 6
[Background thread] Running periodic work...
[Main thread] Working step 7
[Main thread] Working step 8
[Main thread] Working step 9
[Background thread] Running periodic work...
[Main thread] Working step 10
```

#### Analisis

Main thread mencetak setiap 1 detik, background thread setiap 3 detik — keduanya berjalan bersamaan dalam satu proses. Berbeda dengan Bash yang membuat proses terpisah, di sini keduanya adalah thread dalam address space yang sama. Ketika `main()` selesai (setelah 10 iterasi), seluruh proses berakhir dan background thread ikut berhenti — thread tidak bisa hidup tanpa proses induknya.

---

### Lab 8: Race Condition (Shared Variable)

#### Tujuan
Mendemonstrasikan race condition pada variabel global yang di-increment oleh 3 thread tanpa proteksi.

#### Script - `lab8.c`

```c
#include <stdio.h>
#include <pthread.h>

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;
    }
    return NULL;
}

int main() {
    pthread_t t1, t2, t3;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_create(&t3, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    printf("Expected counter: 300000\n");
    printf("Actual counter  : %d\n", counter);
    return 0;
}
```

#### Cara Menjalankan

```bash
gcc lab8.c -o lab8 -pthread
./lab8
```

#### Hasil Eksekusi (3x percobaan)

```
$ ./lab8
Expected counter: 300000
Actual counter  : 213847

$ ./lab8
Expected counter: 300000
Actual counter  : 198432

$ ./lab8
Expected counter: 300000
Actual counter  : 241509
```

#### Analisis

`counter++` terlihat sederhana tapi sebenarnya terdiri dari 3 instruksi mesin: load, add, store. Ketika dua thread melakukan load pada nilai yang sama sebelum salah satu melakukan store, satu increment hilang. Hasilnya selalu di bawah 300000 dan berubah-ubah karena bergantung pada timing penjadwalan. Race condition di level thread lebih parah dari level proses (Lab 3) karena thread berbagi memori secara langsung — akses lebih cepat berarti lebih banyak peluang konflik.

---

### Lab 9: Fix Race Condition dengan Mutex

#### Tujuan
Memperbaiki race condition menggunakan `pthread_mutex` untuk melindungi critical section.

#### Script - `lab9.c`

```c
#include <stdio.h>
#include <pthread.h>

int counter = 0;
pthread_mutex_t lock;

void* increment(void* arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main() {
    pthread_t t1, t2, t3;
    pthread_mutex_init(&lock, NULL);

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_create(&t3, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    printf("Expected counter: 300000\n");
    printf("Actual counter  : %d\n", counter);

    pthread_mutex_destroy(&lock);
    return 0;
}
```

#### Cara Menjalankan

```bash
gcc lab9.c -o lab9 -pthread
./lab9
```

#### Hasil Eksekusi

```
Expected counter: 300000
Actual counter  : 300000
```

#### Analisis

Dengan mutex, hasilnya selalu tepat 300000. `pthread_mutex_lock` memastikan hanya satu thread yang bisa mengeksekusi `counter++` pada satu waktu — thread lain harus menunggu sampai `unlock` dipanggil. Ini membuktikan bahwa sinkronisasi mengembalikan correctness, namun dengan biaya performa karena bagian critical section menjadi serial. Prinsip desainnya: buat critical section sekecil mungkin agar thread tidak terlalu lama menunggu.

---

### Lab 10: Blocking Threads

#### Tujuan
Menunjukkan bahwa thread yang ter-block (I/O wait) tidak menghalangi thread lain untuk tetap bekerja.

#### Script - `lab10.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* slow_task(void* arg) {
    printf("Thread A: waiting for simulated I/O...\n");
    sleep(6);
    printf("Thread A: I/O completed\n");
    return NULL;
}

void* fast_task(void* arg) {
    for (int i = 1; i <= 6; i++) {
        printf("Thread B: working step %d\n", i);
        sleep(1);
    }
    return NULL;
}

int main() {
    pthread_t thread_a, thread_b;
    pthread_create(&thread_a, NULL, slow_task, NULL);
    pthread_create(&thread_b, NULL, fast_task, NULL);

    pthread_join(thread_a, NULL);
    pthread_join(thread_b, NULL);

    printf("Both threads finished\n");
    return 0;
}
```

#### Cara Menjalankan

```bash
gcc lab10.c -o lab10 -pthread
./lab10
```

#### Hasil Eksekusi

```
Thread A: waiting for simulated I/O...
Thread B: working step 1
Thread B: working step 2
Thread B: working step 3
Thread B: working step 4
Thread B: working step 5
Thread B: working step 6
Thread A: I/O completed
Both threads finished
```

#### Analisis

Thread A ter-block selama 6 detik (simulasi I/O), tapi Thread B tetap berjalan mencetak progress setiap detik. Keduanya selesai hampir bersamaan (~6 detik). Ini menunjukkan bahwa blocking bersifat per-thread, bukan per-proses — scheduler OS akan menjalankan thread lain yang ready sementara satu thread menunggu. Tanpa multithreading, program harus menunggu I/O selesai sebelum melakukan pekerjaan lain.

---

## Kesimpulan

| Aspek | Bash (Part A) | C/pthreads (Part B) |
|-------|---------------|---------------------|
| Unit concurrency | Background process (PID terpisah) | Thread (address space sama) |
| Shared resource | File (counter.txt) | Variabel global (counter) |
| Mekanisme lock | mkdir sebagai mutex | pthread_mutex_lock/unlock |
| Race condition | Lab 3 → fixed di Lab 4 | Lab 8 → fixed di Lab 9 |
| Blocking behavior | Lab 5 (proses independen) | Lab 10 (thread independen) |

Concurrency meningkatkan performa dan responsivitas, tetapi akses ke shared resource tanpa sinkronisasi akan menyebabkan race condition. Mutual exclusion (mutex) mengembalikan correctness dengan trade-off performa yang harus dikelola melalui desain critical section yang minimal.
