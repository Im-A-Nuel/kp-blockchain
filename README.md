# KP Blockchain – Federated Learning Audit On-Chain (Mandala/Substrate)

## Ringkasan
Repositori ini berisi pekerjaan bagian **blockchain** untuk KP: *Skor Kelayakan Penerima Subsidi Tanpa Tukar Data*. Fokus utama:
- Mencatat **hash kontribusi model Federated Learning** sebagai **audit trail on-chain**.
- Integrasi **IPFS** (untuk artefak besar) → hanya hash/metadata dicatat di chain.
- Menyediakan **query & event** agar proses dapat diverifikasi publik tanpa membuka data pribadi.

## Arsitektur 
- **Instansi** (Dinsos/Dukcapil/Kemenkes) melatih model lokal → hasilkan *model update*.
- Update dihitung **SHA-256** → `update_hash`.
- (Opsional) upload file update ke **IPFS** → dapat `CID`.
- Kirim extrinsic ke chain (pallet `fl-audit`) → simpan `update_hash`, `round_id`, `org_id`, `cid?`.
- **Event** dipancarkan → bukti publik immutabel.

## Struktur Repo
- `docs/` – catatan materi, roadmap.
- `chain/` – kode blockchain:
  - `parachain-template/` (template).
  - `solo-chain-template/` (template).

- `scripts/` – code extrinsic / utilitas.

## Quick Start (Solo Dev)
1) Ikuti `docs/materi/3-local-node-setup.md` untuk menjalankan node lokal.  
2) Uji extrinsic dasar `system.remarkWithEvent` (lihat Materi 4).  

## Self-Reporting
- **GitHub** – commit harian (kode & catatan).
- **YouTube** – vlog harian 5–10 menit (demo singkat + insight).

## Lisensi
@.....
