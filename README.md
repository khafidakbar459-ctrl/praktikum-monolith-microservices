# Hands-on: Arsitektur Monolith vs Microservices dengan Flask Python

Praktikum mata kuliah **Paradigma Sistem untuk IT** (Politeknik Negeri Lhokseumawe).
Repositori ini membandingkan dua arsitektur dengan studi kasus yang sama, yaitu **toko buku sederhana** yang punya fitur melihat daftar buku dan membuat pesanan.

| | Monolith | Microservices |
|---|---|---|
| Program | `monolith_app.py` | `book_service.py` + `order_service.py` |
| Port | 5000 | 5001 (Book), 5002 (Order) |

> Data buku dan pesanan disimpan di variabel Python (in-memory), jadi **tidak ada database**. Fokusnya hanya pada perbedaan arsitektur.

**Penulis:** Muhammad Khafid Akbar, TRKJ 3C, Politeknik Negeri Lhokseumawe

---

## Daftar Isi

1. [Tujuan](#1-tujuan)
2. [Alat dan Bahan](#2-alat-dan-bahan)
3. [Struktur Project](#3-struktur-project)
4. [Langkah 1: Persiapan Environment](#4-langkah-1-persiapan-environment)
5. [Langkah 2: Membuat dan Menjalankan Monolith](#5-langkah-2-membuat-dan-menjalankan-monolith)
6. [Langkah 3: Menguji Monolith](#6-langkah-3-menguji-monolith)
7. [Langkah 4: Membuat Book Service](#7-langkah-4-membuat-book-service)
8. [Langkah 5: Membuat Order Service](#8-langkah-5-membuat-order-service)
9. [Langkah 6: Menjalankan Kedua Service](#9-langkah-6-menjalankan-kedua-service)
10. [Langkah 7: Menguji Microservices](#10-langkah-7-menguji-microservices)
11. [Langkah 8: Menguji Fault Isolation](#11-langkah-8-menguji-fault-isolation)
12. [Ringkasan Endpoint](#12-ringkasan-endpoint)
13. [Perbandingan Monolith dan Microservices](#13-perbandingan-monolith-dan-microservices)
14. [Catatan Penting](#14-catatan-penting)

---

## 1. Tujuan

1. Memahami konsep dasar aplikasi Monolith dan Microservices.
2. Mampu membangun backend sederhana menggunakan Flask Python.
3. Menganalisis kelebihan dan kekurangan masing-masing arsitektur lewat praktik langsung.

## 2. Alat dan Bahan

| Alat | Fungsi |
|---|---|
| Linux | Sistem operasi untuk menjalankan semua perintah |
| Python 3 | Bahasa pemrograman |
| `venv` | Virtual environment, memisahkan library praktikum dari Python lain |
| Flask | Framework untuk membuat aplikasi dan API |
| Requests | Library untuk mengirim HTTP request dari Order Service ke Book Service |
| Visual Studio Code | Editor kode (terminal terintegrasi dipakai untuk menjalankan program) |
| `curl` | Mengirim request ke API, termasuk request POST |
| `lsof` | Memeriksa apakah port 5000 sedang dipakai |
| Browser | Menguji endpoint `GET /books` pada Monolith |

## 3. Struktur Project

```text
praktikum-monolith-microservices/
├── monolith_app.py      # Monolith (buku + pesanan), port 5000
├── book_service.py      # Book Service, port 5001
├── order_service.py     # Order Service, port 5002
└── venv/                # Virtual environment Python
```

---

## 4. Langkah 1: Persiapan Environment

Masuk ke folder project, buat virtual environment, aktifkan, lalu pasang Flask dan Requests.

```bash
cd ~/Dokumen/"paradigma sistem it"/praktikum-monolith-microservices
python3 -m venv venv
source venv/bin/activate
pip install Flask requests
```

Kalau berhasil, di depan prompt terminal muncul tulisan `(venv)`.

> Setiap membuka terminal baru, jalankan lagi `source venv/bin/activate` di folder project.

Setelah itu buat tiga file kosong di VS Code: `monolith_app.py`, `book_service.py`, dan `order_service.py`.

---

## 5. Langkah 2: Membuat dan Menjalankan Monolith

Pada Monolith, fitur buku dan fitur pesanan berada dalam **satu file** dan **satu proses**. Fitur pesanan membaca data buku langsung dari variabel `books`, tanpa lewat jaringan.

### Kode `monolith_app.py`

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

# Database bohongan (sementara)
books = [
    {
        "id": 1,
        "title": "Belajar Flask",
        "stock": 5
    }
]

orders = []


# Menampilkan semua buku
@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)


# Membuat pesanan
@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')

    for book in books:
        if book['id'] == book_id and book['stock'] > 0:
            book['stock'] -= 1

            order = {
                "id": len(orders) + 1,
                "book_id": book_id,
                "status": "berhasil"
            }

            orders.append(order)

            return jsonify(order), 201

    return jsonify({
        "error": "Buku tidak ditemukan atau stok habis"
    }), 400


if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

### Cara kerja singkat

- `GET /books` mengembalikan isi variabel `books` dalam format JSON.
- `POST /orders` membaca `book_id`, lalu mengecek variabel `books` secara langsung.
  - Buku ditemukan dan stok lebih dari 0: stok dikurangi 1, pesanan dibuat dengan status `berhasil`, respons kode **201**.
  - Buku tidak ditemukan atau stok habis: respons kode **400**.

### Periksa port 5000 lalu jalankan

Pastikan port 5000 kosong. Kalau tidak ada proses yang tampil, berarti aman.

```bash
sudo lsof -i :5000
```

Jalankan Monolith:

```bash
python monolith_app.py
```

Hasil yang diharapkan di terminal:

```text
 * Serving Flask app 'monolith_app'
 * Debug mode: on
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
```

> Peringatan merah *"This is a development server..."* adalah peringatan bawaan Flask. Itu normal untuk mode development.

---

## 6. Langkah 3: Menguji Monolith

Biarkan Monolith tetap jalan di terminal pertama, lalu buka **terminal baru** untuk pengujian.

### 3.1 Uji `GET /books` lewat browser

Buka alamat berikut di browser:

```text
http://127.0.0.1:5000/books
```

Hasil yang diharapkan:

```json
[
  {
    "id": 1,
    "stock": 5,
    "title": "Belajar Flask"
  }
]
```

### 3.2 Uji `POST /orders` lewat curl

```bash
curl -X POST -H "Content-Type: application/json" -d '{"book_id":1}' http://localhost:5000/orders
```

Hasil yang diharapkan:

```json
{
  "book_id": 1,
  "id": 1,
  "status": "berhasil"
}
```

Pesanan pertama berhasil karena buku dengan id 1 ditemukan dan stoknya masih ada.

Setelah selesai, hentikan Monolith dengan `Ctrl + C` di terminal pertama.

---

## 7. Langkah 4: Membuat Book Service

Pada Microservices, fitur buku dipisah menjadi layanan sendiri. Book Service punya data `books` sendiri dan menyediakan dua endpoint. Endpoint `GET /books/<book_id>` nanti dipanggil oleh Order Service.

### Kode `book_service.py`

```python
from flask import Flask, jsonify

app = Flask(__name__)

books = [
    {
        "id": 1,
        "title": "Belajar Flask",
        "stock": 5
    }
]


@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)


@app.route('/books/<int:book_id>', methods=['GET'])
def get_book(book_id):
    for b in books:
        if b['id'] == book_id:
            return jsonify(b)

    return jsonify({
        "error": "Not found"
    }), 404


if __name__ == '__main__':
    app.run(port=5001, debug=True)
```

---

## 8. Langkah 5: Membuat Order Service

Fitur pesanan dipisah menjadi layanan sendiri. Order Service **tidak punya data buku**, jadi tidak bisa membaca variabel `books`. Ia harus meminta data ke Book Service lewat HTTP.

### Kode `order_service.py`

```python
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)

orders = []

BOOK_SERVICE_URL = "http://localhost:5001"


@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')

    try:
        response = requests.get(
            f"{BOOK_SERVICE_URL}/books/{book_id}"
        )

        if response.status_code == 200:
            book_data = response.json()

            if book_data['stock'] > 0:
                order = {
                    "id": len(orders) + 1,
                    "book_id": book_id,
                    "status": "berhasil"
                }

                orders.append(order)

                return jsonify(order), 201

        return jsonify({
            "error": "Buku tidak tersedia"
        }), 400

    except requests.exceptions.ConnectionError:
        return jsonify({
            "error": "Book Service sedang down!"
        }), 500


if __name__ == '__main__':
    app.run(port=5002, debug=True)
```

### Alur komunikasi antar service

```text
Pengguna ──POST /orders──▶ Order Service (5002)
                                │
                                │ GET /books/{book_id}
                                ▼
                          Book Service (5001)
                                │
                                │ data buku (200) atau "Not found" (404)
                                ▼
                          Order Service membuat pesanan (201)
                          atau menolak (400)
```

Kalau Book Service tidak bisa dihubungi, blok `except requests.exceptions.ConnectionError` menangkapnya dan Order Service membalas pesan `Book Service sedang down!` dengan kode **500**.

---

## 9. Langkah 6: Menjalankan Kedua Service

Kedua service harus jalan **bersamaan**, masing-masing di terminal sendiri. Di setiap terminal, pastikan `(venv)` aktif dan posisi ada di folder project.

**Terminal 1: Book Service**

```bash
python book_service.py
```

Hasil yang diharapkan:

```text
 * Serving Flask app 'book_service'
 * Debug mode: on
 * Running on http://127.0.0.1:5001
```

**Terminal 2: Order Service**

```bash
python order_service.py
```

Hasil yang diharapkan:

```text
 * Serving Flask app 'order_service'
 * Debug mode: on
 * Running on http://127.0.0.1:5002
```

Terminal 3 dipakai untuk pengujian.

---

## 10. Langkah 7: Menguji Microservices

Semua perintah di bawah dijalankan di **terminal 3**.

### 7.1 Uji Book Service

```bash
curl http://localhost:5001/books
```

Hasil yang diharapkan:

```json
[
  {
    "id": 1,
    "stock": 5,
    "title": "Belajar Flask"
  }
]
```

Isinya sama dengan hasil pada Monolith, tetapi sekarang dilayani oleh Book Service di port 5001.

### 7.2 Uji Order Service

```bash
curl -X POST -H "Content-Type: application/json" -d '{"book_id":1}' http://localhost:5002/orders
```

Hasil yang diharapkan:

```json
{
  "book_id": 1,
  "id": 1,
  "status": "berhasil"
}
```

Order Service tidak membaca data buku sendiri. Ia mengirim HTTP request ke Book Service untuk mengecek ketersediaan dan stok. Nilai `id` 1 muncul karena Order Service punya daftar `orders` sendiri yang terpisah dari Monolith.

---

## 11. Langkah 8: Menguji Fault Isolation

Fault isolation adalah kemampuan sistem membatasi dampak kegagalan satu bagian agar tidak menjalar ke bagian lain.

1. Hentikan Book Service dengan menekan `Ctrl + C` di **terminal 1**.
2. Biarkan Order Service di **terminal 2** tetap berjalan.
3. Kirim lagi request pesanan dari **terminal 3**:

   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"book_id":1}' http://localhost:5002/orders
   ```

Hasil yang diharapkan:

```json
{
  "error": "Book Service sedang down!"
}
```

**Artinya:** Order Service masih hidup dan membalas dengan pesan yang jelas (kode 500), walaupun Book Service mati. Perlu dicatat, fault isolation tidak berarti semua fitur tetap normal. Pesanan tetap tidak bisa diproses selama Book Service mati. Yang terjaga adalah Order Service tetap hidup dan memberi respons yang jelas.

---

## 12. Ringkasan Endpoint

| Endpoint | Metode | Keterangan | Ada di |
|---|---|---|---|
| `/books` | GET | Menampilkan daftar buku | Monolith, Book Service |
| `/books/<book_id>` | GET | Menampilkan satu buku berdasarkan id | Book Service |
| `/orders` | POST | Membuat pesanan | Monolith, Order Service |

Kode status HTTP yang dipakai:

| Kode | Arti |
|---|---|
| 200 | Berhasil |
| 201 | Data berhasil dibuat |
| 400 | Permintaan tidak dapat dipenuhi |
| 404 | Data tidak ditemukan |
| 500 | Kesalahan pada sisi server |

---

## 13. Perbandingan Monolith dan Microservices

| Aspek | Monolith | Microservices |
|---|---|---|
| Jumlah program | 1 file: `monolith_app.py` | 2 file: `book_service.py` dan `order_service.py` |
| Port | 5000 | Book Service 5001, Order Service 5002 |
| Penyimpanan data | `books` dan `orders` dalam satu program | `books` di Book Service, `orders` di Order Service (terpisah) |
| Akses pesanan ke data buku | Langsung membaca variabel `books` | Mengirim HTTP request GET ke Book Service |
| Pemeriksaan stok | Stok diperiksa dan langsung dikurangi | Stok hanya diperiksa, tidak dikurangi |
| Cara menjalankan | Satu perintah, satu terminal | Dua perintah di dua terminal |
| Komunikasi antar fitur | Tanpa jaringan (dalam satu proses) | Lewat HTTP antar service |
| Saat sebagian sistem berhenti | Satu proses, jadi jika berhenti semua fitur ikut berhenti (konsep, tidak diuji) | Book Service mati, Order Service tetap hidup dan memberi pesan error (diuji) |

**Kesimpulan singkat:** Monolith lebih sederhana (satu file, satu port, satu perintah, tanpa komunikasi jaringan). Microservices butuh lebih banyak file, port, dan terminal serta komunikasi HTTP, tetapi antar layanan lebih terpisah. Pemilihan arsitektur bergantung pada kebutuhan aplikasi.

---

## 14. Catatan Penting

- **In-memory:** data hilang setiap program dihentikan lalu dijalankan ulang. Itu disengaja, supaya praktikum fokus ke arsitektur.
- **Stok tidak dikurangi di Microservices:** Order Service hanya memeriksa stok. Mengurangi stok butuh komunikasi tambahan (misalnya endpoint baru di Book Service).
- **Belum ada timeout:** `requests.get()` di Order Service belum diberi `timeout`. Di dunia nyata, sebaiknya ditambahkan agar latensi tinggi tidak menahan seluruh sistem.
- **Kecepatan tidak diukur:** praktikum ini tidak membandingkan waktu respons Monolith dan Microservices.
- **Development server:** `debug=True` hanya untuk belajar, jangan dipakai di produksi.
- **Port bentrok:** kalau muncul error *Address already in use*, cek proses yang memakai port dengan `sudo lsof -i :<port>`, lalu hentikan prosesnya.

---

## Referensi

- Fowler, M., & Lewis, J. (2014). *Microservices: A definition of this new architectural term.* https://martinfowler.com/articles/microservices.html
- Newman, S. (2015). *Building Microservices.* O'Reilly Media.
- Fielding, R. T. (2000). *Architectural styles and the design of network-based software architectures.* University of California, Irvine.
- Flask documentation: https://flask.palletsprojects.com/
- Requests documentation: https://requests.readthedocs.io/
- Python `venv`: https://docs.python.org/3/library/venv.html
