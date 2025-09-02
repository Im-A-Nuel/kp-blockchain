# Materi 4 – Extrinsic & Storage Concepts

## 🎯 Tujuan Pembelajaran
- Memahami konsep **extrinsic** sebagai cara interaksi dari luar chain.
- Memahami konsep **storage** sebagai tempat penyimpanan state di blockchain.
- Mencoba mengirim **extrinsic** sederhana dan membaca **storage** melalui Polkadot.js Apps.
- Menyambungkan konsep ini ke use case project: **Audit On-Chain Federated Learning**.

---

## 📚 Ringkasan Materi
### 1. Extrinsic
- **Definisi**: Input dari luar yang masuk ke blockchain untuk diproses runtime.
- **Jenis**:
  1. **Signed Extrinsic** → butuh tanda tangan akun (contoh: transfer balance).
  2. **Unsigned Extrinsic** → tanpa tanda tangan (jarang, raw input).
  3. **Inherent Extrinsic** → berasal dari node itu sendiri (contoh: timestamp).

### 2. Storage
- **Definisi**: State blockchain yang disimpan dalam bentuk **key-value database**.
- **Dikelola oleh**: Pallet di runtime.
- **Akses**: hanya lewat extrinsic atau query state.
- Semua update storage akan menghasilkan **event** untuk transparansi.

### 3. Flow: Extrinsic → Storage → Event
1. User kirim extrinsic ke node.
2. Runtime memproses dan mengubah storage.
3. Blockchain emit event sebagai bukti publik.
4. State terbaru tersimpan di semua node.

---

## 🛠️ Praktik

### A. Kirim Extrinsic (system.remarkWithEvent)
1. Jalankan node lokal:
   ```bash
   ./target/release/solochain-template-node --dev --tmp
2. Buka Polkadot.js Apps
3. Connect ke Local Node (127.0.0.1:9944).
4. Masuk ke menu **Developer** → **Extrinsics**.
5. Pilih akun Alice.
6. Pallet: system, Function: **remarkWithEvent**.
7. Isi remark: Hello Blockchain.
8. Klik Submit Transaction.
![alt text](<Screenshot 2025-09-01 110744.png>)
9. Buka Explorer → Events → pastikan muncul system.Remarked.
![alt text](<Screenshot 2025-09-01 110929.png>)


### B. Baca Storage 
1. Masih di Polkadot.js Apps.
2. Masuk ke menu Developer → Chain state.
3. **Query**: system → account → isi address Alice.
4. Klik + → tampil saldo, nonce, dan info lain.
![alt text](<Screenshot 2025-09-01 111333.png>)
![alt text](<Screenshot 2025-09-01 112247.png>)


### 📌 Relevansi ke Project KP
- **Extrinsic** → cara instansi mengirim hash kontribusi model ke blockchain.

- **Storage** → menyimpan hash + metadata sebagai audit log.

- **Event** → bukti bahwa hash sudah tersimpan immutable di chain.

- **Query state** → auditor/verifier bisa mengambil hash untuk verifikasi.