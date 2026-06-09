# Hands-On Sistem Operasi

**Nama:** I Dewa Nyoman Acarya Wibawantra
**NRP:** 5025251222
**Mata Kuliah:** Sistem Operasi — Semester 2

---

Repository ini berisi kumpulan laporan dan jawaban Hands-On praktikum Sistem Operasi. Setiap folder memuat README dengan script, hasil eksekusi, dan analisis untuk masing-masing lab.

## Daftar Hands-On

| No | Topik | Folder | Jumlah Lab |
|----|-------|--------|------------|
| 3 | Process Lifecycle | [Handson-3](./hands-on-3-Process-Lifecycle.md) | 5 task |
| 4 | Threads and Concurrency (Bash & C) | [Handson-4](./Handson-4) | 10 lab |
| 5 | Mutex and Threads | [Handson-5](./Handson-5) | 11 lab |
| 6 | Deadlock and Starvation | [Handson-6](./Handson-6) | 5 lab |

## Ringkasan Materi

**Hands-On 3 — Process Lifecycle:** Membuat dan mengamati proses, monitoring status, komunikasi antar-proses via signal file, siklus hidup proses dengan `trap`, dan dependensi antar-proses dalam workflow.

**Hands-On 4 — Threads and Concurrency:** Sequential vs concurrent execution, foreground/background tasks, race condition pada shared file, fixing race condition dengan lock (Bash), serta implementasi serupa menggunakan pthreads di C.

**Hands-On 5 — Mutex and Threads:** Multiple threads, race condition pada shared variable, mutual exclusion dengan mutex, producer-consumer dengan semaphore, condition variable, critical section (bank account), dan readers-writers problem.

**Hands-On 6 — Deadlock and Starvation:** Thread concurrency dan resource competition, simulasi deadlock, deadlock prevention via resource ordering, Dining Philosophers problem, dan blocking synchronization dengan semaphore.

## Teknologi

- **Bash scripting** — process management, file-based IPC, directory lock
- **C dengan pthreads** — thread creation, mutex, semaphore, condition variable
- **Linux/Unix environment** — GCC compiler, POSIX thread library
