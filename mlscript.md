# Naskah Video Demo Project Machine Learning: Customer Churn Prediction

**Durasi Estimasi:** 3-5 Menit
**Nada Suara:** Profesional, Jelas, dan Antusias
**Setting:** Screen recording Jupyter Notebook yang sedang dijalankan per cell.

---

## 1. Pembukaan (0:00 - 0:30)
**(Visual: Menampilkan Judul Notebook dan Nama Anggota Kelompok)**

**Narator:**
"Halo semuanya! Selamat datang di presentasi project Machine Learning kelompok kami. Pada kesempatan kali ini, kami akan mendemonstrasikan proses pemodelan data untuk memprediksi **Customer Churn** atau pelanggan yang berhenti berlangganan.

Tujuan dari project ini adalah membantu perusahaan telekomunikasi mengidentifikasi pelanggan yang berpotensi pindah ke kompetitor, sehingga bisa dilakukan tindakan pencegahan lebih dini. Mari kita mulai!"

---

## 2. Business & Data Understanding (0:30 - 1:15)
**(Visual: Scroll ke bagian '1. Business Understanding' dan '2. Data Understanding'. Jalankan cell load data dan visualisasi)**

**Narator:**
"Pertama, kita masuk ke tahap **Data Understanding**. Di sini kami menggunakan dataset *Telco Customer Churn* yang bersifat publik.

Kita load datanya langsung dari URL repository agar bisa dijalankan di mana saja.
*(Klik Run Cell Load Data)*

Bisa dilihat di sini (`df.head()`), data kita memiliki berbagai fitur seperti `gender`, `Partner`, `Dependents`, layanan yang dipakai, hingga `MonthlyCharges`. Target kita adalah kolom **'Churn'**.

Kami juga melakukan visualisasi sederhana untuk melihat sebaran data target.
*(Klik Run Cell Visualisasi)*
Terlihat bahwa jumlah pelanggan yang Churn (Yes) lebih sedikit dibandingkan yang tidak (No), yang menandakan dataset ini sedikit *imbalanced*, tapi masih cukup oke untuk pemodelan."

---

## 3. Data Preparation (1:15 - 2:00)
**(Visual: Masuk ke bagian '3. Data Preparation'. Jalankan cell cleaning dan encoding)**

**Narator:**
"Lanjut ke tahap krusial, yaitu **Data Preparation**.
Saat kami mengecek tipe data, ditemukan bahwa kolom `TotalCharges` masih terbaca sebagai object/teks karena ada spasi kosong. Jadi, kami ubah dulu menjadi numerik dan mengisi nilai yang kosong dengan nilai median.

Selanjutnya, kami membuang kolom `customerID` karena tidak relevan untuk prediksi.

Untuk data kategorikal seperti `InternetService` atau `PaymentMethod`, kami lakukan proses **Encoding**.
- Kolom dengan 2 nilai (Yes/No) kami ubah jadi 0 dan 1.
- Kolom dengan banyak kategori kami gunakan *One-Hot Encoding*.

*(Klik Run Cell Encoding)*
Hasilnya, semua data kini sudah dalam bentuk angka dan siap untuk dimodelkan."

---

## 4. Modeling (2:00 - 2:45)
**(Visual: Masuk ke bagian '4. Modeling'. Jalankan cell splitting dan training)**

**Narator:**
"Sekarang kita masuk ke inti proses, yaitu **Modeling**.
Pertama, data kami bagi menjadi **80% Training** dan **20% Testing**. Kami juga melakukan **Scaling** agar rentang nilai data seragam, yang sangat penting untuk algoritma seperti Logistic Regression.

Di sini, kami bereksperimen dengan dua algoritma populer:
1. **Logistic Regression**: Karena ini masalah klasifikasi biner yang linear.
2. **Random Forest**: Untuk menangkap pola data yang lebih kompleks.

Mari kita latih kedua model tersebut.
*(Klik Run Cell Training)*
Oke, proses training selesai!"

---

## 5. Model Evaluation (2:45 - 3:30)
**(Visual: Masuk ke bagian '5. Model Evaluation'. Jalankan cell evaluasi)**

**Narator:**
"Bagaimana performanya? Mari kita evaluasi.
*(Klik Run Cell Evaluasi)*

Bisa dilihat hasilnya:
- **Logistic Regression** mendapatkan akurasi sekitar **82%**.
- Sedangkan **Random Forest** mendapatkan akurasi sekitar **79%**.

Ternyata untuk kasus ini, model Logistic Regression bekerja sedikit lebih baik. Kita bisa lihat detailnya di *Confusion Matrix*, di mana model cukup baik dalam memprediksi pelanggan yang setia (Tidak Churn), namun masih ada tantangan dalam mendeteksi yang Churn."

---

## 6. Deployment & Penutup (3:30 - 4:30)
**(Visual: Masuk ke bagian '6. Deployment'. Jalankan cell simpan model dan prediksi baru)**

**Narator:**
"Terakhir, tahap **Deployment**.
Kami menyimpan model terbaik dan scaler-nya ke dalam file `.pkl` agar bisa digunakan nanti tanpa perlu training ulang.

Kami juga membuat simulasi prediksi menggunakan data baru.
*(Klik Run Cell Simulasi)*

Nah, ini contohnya. Sistem berhasil memprediksi 5 data pelanggan baru. Ada yang diprediksi 'Churn' dengan probabilitas tinggi (misalnya 73% atau 81%), dan ada yang 'Tidak Churn'. Informasi probabilitas ini sangat berguna bagi tim marketing untuk memprioritaskan penanganan pelanggan.

Demikian demo project Machine Learning kami. Terima kasih!"

---
