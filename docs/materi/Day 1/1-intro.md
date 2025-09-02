# Materi 1 – Intro Blockchain

## Tujuan
- Memahami konsep dasar blockchain.
- Mengenal struktur blok (hash, prev hash, data, nonce).
- Melihat bagaimana rantai blok terbentuk dan kenapa immutabel.

## Ringkasan
- Blockchain adalah rangkaian blok yang saling terkait dengan hash.
- Setiap blok menyimpan: `prev_hash`, `timestamp`, `data`, dan `hash` blok itu sendiri.
- Jika satu blok dimodifikasi, maka seluruh rantai setelahnya menjadi tidak valid.
- Mekanisme ini membuat blockchain tahan manipulasi.

## Praktik
- Menggunakan [Blockchain Demo – Anders Brownworth](https://andersbrownworth.com/blockchain/blockchain).
- Mengubah data di blok #2 akan membuat semua hash berikutnya tidak valid.
- Hanya dengan meng-*mine* ulang blok, hash bisa disesuaikan, tapi seluruh rantai tetap harus dihitung ulang.

## Screenshot
![Demo Blockchain](./screenshots/blockchain-demo.png)

## Relevansi ke KP
Konsep hash dan chaining inilah yang akan dipakai untuk mencatat kontribusi model Federated Learning.  
Hash dari model update dicatat di chain agar **immutabel** dan bisa diverifikasi kapan pun.
