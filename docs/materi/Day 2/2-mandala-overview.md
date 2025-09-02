# Materi 2 – Mandala Chain Overview

## Tujuan
- Mengenal arsitektur Mandala Chain.
- Memahami konsensus dan governance yang digunakan.
- Menyambungkan konsep Mandala dengan use case Federated Learning.

---

## Ringkasan Materi

### 1. Apa itu Mandala Chain?
- Mandala Chain adalah **parachain berbasis Substrate** di ekosistem Polkadot.
- Dirancang untuk mendukung **interoperabilitas** dan **governance terdesentralisasi**.
- Fokusnya pada ekosistem lokal Indonesia, dengan potensi mendukung sektor publik dan swasta.

### 2. Governance
- Menggunakan mekanisme **on-chain governance**: council, referendum, dan proposal publik.
- Setiap perubahan protokol atau upgrade runtime dapat dilakukan tanpa hard fork.
- Hal ini mendukung transparansi serta partisipasi semua pemegang token.

### 3. Konsensus
- Memanfaatkan konsensus **BABE (Blind Assignment for Blockchain Extension)** untuk block production.
- Menggunakan **GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement)** untuk finality.
- Kombinasi ini membuat chain efisien dalam produksi blok dan cepat dalam finalisasi.

---

## Praktik
- Membaca whitepaper / dokumentasi resmi Mandala.
- Membandingkan arsitektur Mandala dengan template parachain Substrate.
- Diskusi: bagaimana governance dan konsensus mendukung keamanan & transparansi.

---

## Relevansi ke KP
Dalam project **Audit On-Chain Federated Learning**:
- Governance Mandala dapat menjadi dasar untuk membuat aturan siapa instansi yang berhak mengirim kontribusi model.  
- Konsensus BABE + GRANDPA memastikan setiap hash kontribusi yang dicatat benar-benar final dan tidak bisa dimanipulasi.  
- Audit trail di Mandala Chain membantu meningkatkan **kepercayaan** publik dalam distribusi subsidi tanpa melanggar privasi data.

---

## Screenshot
![Mandala Overview](./screenshots/mandala-overview.png)
